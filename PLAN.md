# OpenClaw Misskey Channel Plugin — 実装計画

## 概要

OpenClaw の channel plugin として Misskey を追加する。
Discord / Matrix と同様に、Misskey のメッセージ（チャット）・メンション・ノートを OpenClaw のエージェントが受信・応答できるようにする。

### 対象バージョン

新チャット機能（メッセージ）を搭載した Misskey **v2025.4.0 以降** のみサポート。
旧 messaging API（v13.7.0 で廃止）や v12 系の互換性は考慮しない。

## 参考実装

### OpenClaw 既存プラグイン

- [openclaw/openclaw](https://github.com/openclaw/openclaw) — 本体リポジトリ
- `extensions/matrix/` — Matrix channel plugin（E2E暗号化、スレッド対応）← **主要参考実装**
- `extensions/discord/` — Discord channel plugin
- [mzkri/claw-matrix](https://github.com/mzkri/claw-matrix) — Matrix（外部リポ版）
- [soimy/openclaw-channel-dingtalk](https://github.com/soimy/openclaw-channel-dingtalk) — DingTalk
- [ProofOfReach/openclaw-mattermost](https://github.com/ProofOfReach/openclaw-mattermost) — Mattermost

## Plugin SDK Interface

OpenClaw 本体のソースコード（`src/plugins/types.ts`, `src/channels/plugins/types.plugin.ts`）から確認した正確なパターン。

### エントリポイント（index.ts）

```typescript
import type { OpenClawPluginApi } from "openclaw/plugin-sdk";
import { emptyPluginConfigSchema } from "openclaw/plugin-sdk";
import { misskeyPlugin } from "./src/channel.js";
import { setMisskeyRuntime } from "./src/runtime.js";

const plugin = {
  id: "misskey",
  name: "Misskey",
  description: "Misskey channel plugin for Fediverse",
  configSchema: emptyPluginConfigSchema(),
  register(api: OpenClawPluginApi) {
    setMisskeyRuntime(api.runtime);
    api.registerChannel({ plugin: misskeyPlugin });
  },
};

export default plugin;
```

### ChannelPlugin オブジェクト（src/channel.ts）

```typescript
import type { ChannelPlugin } from "openclaw/plugin-sdk";

export const misskeyPlugin: ChannelPlugin<ResolvedMisskeyAccount> = {
  id: "misskey",
  meta: { id: "misskey", label: "Misskey", selectionLabel: "Misskey (plugin)", ... },
  capabilities: { chatTypes: ["direct", "group"], reactions: true, media: true },

  // 必須: アカウント設定の解決
  config: {
    listAccountIds: (cfg) => ...,
    resolveAccount: (cfg, accountId) => ...,
  },

  // セキュリティ: DM ポリシー
  security: {
    resolveDmPolicy: ({ account }) => ({
      policy: account.config.dm?.policy ?? "pairing",
      allowFrom: account.config.dm?.allowFrom ?? [],
      policyPath: "channels.misskey.dm.policy",
      allowFromPath: "channels.misskey.dm.allowFrom",
      approveHint: formatPairingApproveHint("misskey"),
    }),
    collectWarnings: ({ account }) => [...],
  },

  // ペアリング
  pairing: {
    idLabel: "misskeyUserId",
    normalizeAllowEntry: (entry) => entry.replace(/^misskey:/i, ""),
  },

  // Gateway: WebSocket 接続管理
  gateway: {
    startAccount: async (ctx) => { /* Streaming API 接続 */ },
    stopAccount: async (ctx) => { /* クリーンアップ */ },
  },

  // Outbound: メッセージ送信
  outbound: {
    deliveryMode: "direct",
    sendText: async (ctx) => { /* notes/create or chat/messages/create-to-user */ },
  },
};
```

### PluginRuntime（Inbound メッセージフロー）

Gateway の `startAccount` 内で以下のパターンで受信メッセージを処理する（Matrix 実装に準拠）:

```typescript
// 1. ルーティング解決
const route = core.channel.routing.resolveAgentRoute({
  cfg, channel: "misskey", accountId,
  peer: { kind: isDirectMessage ? "user" : "group", id: senderId },
});

// 2. エンベロープ構築
const body = core.channel.reply.formatAgentEnvelope({
  channel: "Misskey", from: senderName,
  timestamp, previousTimestamp, envelope: envelopeOptions, body: textWithId,
});

// 3. Inbound コンテキスト確定
const ctxPayload = core.channel.reply.finalizeInboundContext({
  Body: body,
  BodyForAgent: bodyText,
  RawBody: bodyText,
  CommandBody: bodyText,
  From: `misskey:${senderId}`,
  To: isDirectMessage ? `user:${senderId}` : `note:${noteId}`,
  SessionKey: route.sessionKey,
  AccountId: route.accountId,
  ChatType: isDirectMessage ? "direct" : "group",
  Provider: "misskey",
  Surface: "misskey",
  SenderName: senderName,
  SenderId: senderId,
  MessageSid: noteId,
  // ...
});

// 4. セッション記録
await core.channel.session.recordInboundSession({ ... });

// 5. 応答ディスパッチ
const { dispatcher } = core.channel.reply.createReplyDispatcherWithTyping({
  deliver: async (payload) => { /* notes/create or chat/messages/create-to-user */ },
});
await core.channel.reply.dispatchReplyFromConfig({ ctx: ctxPayload, cfg, dispatcher });
```

### ライフサイクル

```
1. Discovery    → package.json の openclaw.extensions をスキャン
2. Validation   → openclaw.plugin.json の configSchema で設定検証
3. Loading      → エントリポイントを import（jiti による動的ロード）
4. Register     → plugin.register(api) 呼び出し → api.registerChannel()
5. Runtime      → gateway.startAccount(ctx) で Streaming 接続開始
6. Shutdown     → gateway.stopAccount(ctx) でクリーンアップ
```

### 主要な OpenClaw SDK 型

```typescript
// ChannelGatewayContext — startAccount/stopAccount に渡される
type ChannelGatewayContext<ResolvedAccount> = {
  cfg: OpenClawConfig;
  accountId: string;
  account: ResolvedAccount;
  runtime: RuntimeEnv;
  abortSignal: AbortSignal;
  log?: ChannelLogSink;
  getStatus: () => ChannelAccountSnapshot;
  setStatus: (next: ChannelAccountSnapshot) => void;
};

// ChannelOutboundContext — sendText に渡される
type ChannelOutboundContext = {
  cfg: OpenClawConfig;
  to: string;
  text: string;
  mediaUrl?: string;
  replyToId?: string | null;
  threadId?: string | number | null;
  accountId?: string | null;
};

// ChannelSecurityDmPolicy — security.resolveDmPolicy の返却値
type ChannelSecurityDmPolicy = {
  policy: string;
  allowFrom?: Array<string | number> | null;
  policyPath: string;
  allowFromPath: string;
  approveHint: string;
};
```

## ファイル構成

```
openclaw-misskey/
├── openclaw.plugin.json    # プラグインマニフェスト
├── package.json            # npm パッケージ定義
├── tsconfig.json
├── vitest.config.ts
├── index.ts                # エントリポイント（register + export default）
├── PLAN.md
├── SECURITY.md
├── README.md
├── LICENSE
└── src/
    ├── channel.ts          # ChannelPlugin オブジェクト（メイン）
    ├── runtime.ts          # PluginRuntime シングルトン管理
    ├── types.ts            # 型定義（ResolvedMisskeyAccount, MisskeyConfig 等）
    ├── config-schema.ts    # 設定スキーマ（Zod — OpenClaw 本体が Zod 採用）
    ├── accounts.ts         # listAccountIds, resolveAccount
    ├── outbound.ts         # ChannelOutboundAdapter 実装
    ├── onboarding.ts       # セットアップウィザード（Phase 3）
    ├── actions.ts          # メッセージアクション（Phase 3）
    └── misskey/
        ├── client.ts       # misskey-js APIClient ファクトリ + TLS 検証
        ├── stream.ts       # WebSocket Streaming 接続管理
        ├── monitor.ts      # Inbound メッセージハンドラ（mention, chat）
        ├── send.ts         # ノート投稿 / チャット送信
        └── probe.ts        # 接続テスト（/api/i）
```

### openclaw.plugin.json

```json
{
  "id": "misskey",
  "channels": ["misskey"],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

### package.json

```json
{
  "name": "@openclaw/misskey",
  "version": "0.1.0",
  "description": "OpenClaw Misskey channel plugin",
  "type": "module",
  "main": "./dist/index.js",
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch",
    "test": "vitest run",
    "test:watch": "vitest"
  },
  "dependencies": {
    "misskey-js": "^2025.11.0",
    "zod": "^4.3"
  },
  "devDependencies": {
    "openclaw": "workspace:*",
    "typescript": "^5.7",
    "vitest": "^3.0"
  },
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "misskey",
      "label": "Misskey",
      "selectionLabel": "Misskey (plugin)",
      "docsPath": "/channels/misskey",
      "docsLabel": "misskey",
      "blurb": "Fediverse microblogging; configure instance URL + API token.",
      "order": 80,
      "quickstartAllowFrom": true
    },
    "install": {
      "npmSpec": "@openclaw/misskey"
    }
  }
}
```

**依存関係ルール**: `openclaw` は `devDependencies`（モノレポ内）または `peerDependencies`（外部配布時）に置く。

**Zod vs TypeBox**: OpenClaw 本体は Matrix plugin 等で Zod を使用（`zod: "^4.3.6"`）。TypeBox ではなく Zod に統一する。

## 設定構造

OpenClaw の `openclaw.json` の `channels.misskey` セクション:

```json
{
  "channels": {
    "misskey": {
      "enabled": true,
      "host": "https://yami.ski",
      "token": "${MISSKEY_TOKEN}",
      "dm": {
        "policy": "pairing",
        "allowFrom": []
      },
      "groupPolicy": "disabled",
      "visibility": "home",
      "accounts": {
        "sub-account": {
          "host": "https://another.instance",
          "token": "${MISSKEY_TOKEN_2}"
        }
      }
    }
  }
}
```

### ResolvedMisskeyAccount 型

```typescript
type MisskeyDmConfig = {
  enabled?: boolean;
  policy?: DmPolicy;       // "pairing" | "allowlist" | "open" | "disabled"
  allowFrom?: Array<string | number>;
};

type MisskeyConfig = {
  name?: string;
  enabled?: boolean;
  accounts?: Record<string, Omit<MisskeyConfig, "accounts">>;
  host?: string;
  token?: string;
  dm?: MisskeyDmConfig;
  groupPolicy?: GroupPolicy;
  visibility?: "public" | "home" | "followers" | "specified";
  remoteUserPolicy?: "deny" | "allowlist" | "allow";
  allowedInstances?: string[];
  rateLimitPerMinute?: number;
  botFlag?: boolean;
  stripMfm?: boolean;
  tlsOnly?: boolean;
};

type ResolvedMisskeyAccount = {
  accountId: string;
  name?: string;
  enabled: boolean;
  configured: boolean;
  host?: string;
  token?: string;
  config: MisskeyConfig;
};
```

## 主要実装ポイント

### Phase 1: 最小動作（メンション受信 → ノート応答 + チャット DM）

1. **gateway.startAccount** — Misskey Streaming API (WebSocket) に接続、`main` channel subscribe
2. **misskey/monitor.ts** — メンション / チャットメッセージ受信 → `PluginRuntime` 経由でエージェント起動
3. **outbound.sendText** — `notes/create`（メンション応答）/ `chat/messages/create-to-user`（チャット応答）
4. **config** — `listAccountIds` / `resolveAccount`（単一/マルチアカウント対応）
5. **security** — `resolveDmPolicy`（pairing / allowlist / open）+ `collectWarnings`
6. **pairing** — `idLabel: "misskeyUserId"` + `normalizeAllowEntry`

**セキュリティ（Phase 1 必須）** — SECURITY.md の T1,T3-T9 対策:
- DM ポリシー（OpenClaw 標準の pairing/allowlist/open）
- visibility ダウングレード禁止（DM → パブリックへの漏洩防止）
- リモートユーザーポリシー（deny / allowlist / allow）
- 送信レートリミット
- bot フラグ自動設定
- TLS 強制（wss:// のみ）
- トークン権限チェック

### Phase 2: グループ対応

7. **groups** — チャットルーム / Misskey チャンネル監視
8. **directory** — ユーザー / グループ一覧
9. **threading** — リプライチェーン追跡
10. **MFM strip** — MFM → プレーンテキスト変換（T2 対策）

### Phase 3: リッチ機能

11. **actions** — リアクション（カスタム絵文字対応）
12. **media** — 画像 / ファイル添付
13. **onboarding** — セットアップウィザード
14. **サーキットブレーカー** — 出力暴走防止の強化
15. **監査ログ** — 全送受信の記録

## Misskey API で使うエンドポイント

### 受信（Streaming API）

- `wss://{host}/streaming?i={token}` — WebSocket 接続
  - `main` channel: 通知（メンション、メッセージ、フォロー等）
  - `homeTimeline` / `localTimeline`: タイムライン監視（Phase 2）

接続メッセージ形式:
```json
{"type": "connect", "body": {"channel": "main", "id": "unique-id"}}
```

### 送信（ノート）

- `POST /api/notes/create` — ノート投稿（メンション応答）
  - `visibility: "specified"` + `visibleUserIds` でダイレクトノートも可能
- `POST /api/notes/reactions/create` — リアクション（カスタム絵文字対応）

### 送信（チャット / メッセージ）— v2025.4.0+

- `POST /api/chat/messages/create-to-user` — ユーザーへの DM 送信
- `POST /api/chat/messages/create-to-room` — ルームへのメッセージ送信
- `POST /api/chat/messages/react` — メッセージへのリアクション
- `POST /api/chat/history` — チャット履歴取得
- `POST /api/chat/rooms/create` — ルーム作成
- `POST /api/chat/read-all` — 全既読

**注意**: 新チャット機能は現時点で **ローカルユーザー間のみ** 対応。リモートユーザーへの DM にはダイレクトノート（`visibility: "specified"`）を使用する。

### 情報取得

- `POST /api/i` — 自分の情報（接続テスト）
- `POST /api/users/show` — ユーザー情報
- `POST /api/meta` — インスタンス情報

## 依存パッケージ

- `misskey-js` — Misskey 公式 SDK（型安全な REST + Streaming API）
- `zod` — 設定スキーマ定義（OpenClaw 本体に合わせて Zod を使用）
- `openclaw` (devDependencies) — Plugin SDK 型定義 + PluginRuntime

### misskey-js について

- Misskey 本体モノレポ (`misskey-dev/misskey/packages/misskey-js`) で管理
- `api.APIClient` で全エンドポイントが型安全に呼べる
- `Stream` クラスで WebSocket 自動再接続対応
- ランタイム依存: `eventemitter3`, `reconnecting-websocket`, `autobind-decorator`

## アーキテクチャ判断

### MCP Server は不要

Misskey へのアクセスは全て Channel Plugin 経由で行う。
MCP Server（`misskey-mcp` 等）は作らない。理由：

- Channel Plugin は既に Misskey API トークンと misskey-js クライアントを保持している
- モデレーション等の管理系 API も Plugin の actions として追加すれば十分
- Claude Code 等からのアドホックなアクセスは `curl` で代替可能
- Misskey はストリーム型プラットフォームであり、常駐する Channel Plugin が最適なアクセス手段

### モデレーション等の拡張

将来的に自動モデレーションを行う場合の構成：

```
1. Channel Plugin（本リポ）— 通報・メンション受信 + 管理 API 呼び出し
2. Skills（プロンプトファイル）— モデレーションワークフロー定義
   → リポ不要。OpenClaw の設定ディレクトリに .md を置くだけ
```

### TypeBox → Zod

当初 TypeBox を予定していたが、OpenClaw 本体の Matrix / Discord 等のプラグインが Zod（v4）を使用していたため、Zod に変更。

## 注意事項

- 新チャット機能（メッセージ）はローカルユーザー間のみ。リモートユーザーにはダイレクトノートで対応
- カスタム絵文字リアクションは Misskey 固有機能
- `openclaw` の型定義は `devDependencies` に置くこと
- セキュリティモデルの詳細は SECURITY.md を参照

## マイルストーン

- [x] **M0**: PLAN.md + SECURITY.md + README + scaffold
- [ ] **M1**: 型定義 + config-schema + accounts
- [ ] **M2**: misskey-js ラッパー（client, stream, probe）
- [ ] **M3**: Inbound: Streaming 接続 + メンション/チャット受信 → エージェント起動
- [ ] **M4**: Outbound: notes/create + chat/messages/create-to-user
- [ ] **M5**: セキュリティ: DM ポリシー + visibility ガード + レートリミット
- [ ] **M6**: テスト（unit + integration）
- [ ] **M7**: チャットルーム / リプライチェーン（Phase 2）
- [ ] **M8**: リアクション + メディア + onboarding（Phase 3）

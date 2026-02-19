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
- [mzkri/claw-matrix](https://github.com/mzkri/claw-matrix) — Matrix channel plugin（E2E暗号化、スレッド対応）
- [soimy/openclaw-channel-dingtalk](https://github.com/soimy/openclaw-channel-dingtalk) — DingTalk（WebSocket Stream Mode）
- [ProofOfReach/openclaw-mattermost](https://github.com/ProofOfReach/openclaw-mattermost) — Mattermost

## Plugin SDK Interface

OpenClaw の `PluginDefinition` interface（推奨パターン）を使用する。

```typescript
import type { PluginDefinition } from 'openclaw/plugin-sdk'

const plugin: PluginDefinition = {
  slot: "channel",
  id: "misskey",

  schema: Type.Object({ /* ... */ }),

  async init(config, deps) {
    const { logger, configDir, workspaceDir, rpc } = deps;

    // Inbound: 受信メッセージを Gateway に転送
    // rpc.call('gateway.dispatchMessage', { provider: 'misskey', ... })

    return {
      async sendMessage(targets, message) { /* Outbound */ },
      async startAccount() { /* WebSocket 接続開始 */ },
      async stopAccount() { /* クリーンアップ */ },
    };
  },
};
```

### ライフサイクル

```
1. Discovery    → package.json の openclaw セクションをスキャン
2. Validation   → openclaw.plugin.json の configSchema で設定検証
3. Loading      → エントリポイントを import（jiti による動的ロード）
4. Init         → init(config, deps) 呼び出し
5. Runtime      → startAccount() で Streaming 接続、メッセージ送受信
6. Shutdown     → stopAccount() でクリーンアップ
```

### 必須ファイル構成

```
openclaw-misskey/
├── openclaw.plugin.json    # プラグインマニフェスト
├── package.json            # npm パッケージ定義
├── index.ts                # エントリポイント
├── tsconfig.json
└── src/
    ├── plugin.ts           # PluginDefinition 実装（メイン）
    ├── config-schema.ts    # 設定スキーマ（TypeBox）
    ├── types.ts            # 型定義
    ├── onboarding.ts       # セットアップウィザード
    ├── actions.ts          # メッセージアクション（リアクション等）
    ├── outbound.ts         # 送信処理
    ├── resolve-targets.ts  # 宛先解決
    └── misskey/
        ├── client.ts       # misskey-js APIClient ラッパー
        ├── stream.ts       # WebSocket Streaming 受信
        ├── chat.ts         # 新チャット API（メッセージ送受信）
        ├── notes.ts        # ノート投稿 / メンション応答
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

### package.json の openclaw セクション

```json
{
  "name": "@openclaw/misskey",
  "type": "module",
  "main": "./dist/index.js",
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

**依存関係ルール**: `openclaw/plugin-sdk` は `devDependencies` か `peerDependencies` に置く（`dependencies` に入れると npm 外部インストールが壊れる）。

## 主要実装ポイント

### Phase 1: 最小動作（メッセージ受信 → 応答）

1. **startAccount** — Misskey Streaming API (WebSocket) に接続し、メンション・メッセージを受信
2. **sendMessage** — `chat/messages/create-to-user` でメッセージ応答、`notes/create` でメンション応答
3. **config** — インスタンス URL + API トークンの管理
4. **security** — メッセージポリシー（pairing / allowlist / open）

### Phase 2: グループ対応

5. **groups** — チャットルーム / チャンネル / ハッシュタグ監視
6. **directory** — ユーザー / グループ一覧
7. **threading** — リプライチェーンの追跡

### Phase 3: リッチ機能

8. **actions** — リアクション（カスタム絵文字対応）
9. **media** — 画像 / ファイル添付
10. **onboarding** — セットアップウィザード

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

## 設定例（openclaw.json の channels.misskey セクション）

```json
{
  "channels": {
    "misskey": {
      "enabled": true,
      "host": "https://yami.ski",
      "token": "${MISSKEY_TOKEN}",
      "messagePolicy": "pairing",
      "groupPolicy": "disabled",
      "adminUsers": ["hitalin"],
      "visibility": "home"
    }
  }
}
```

## 依存パッケージ

- `misskey-js` — Misskey 公式 SDK（型安全な REST + Streaming API）
- `@sinclair/typebox` — 設定スキーマ定義（OpenClaw 推奨）
- `openclaw/plugin-sdk` (devDependencies) — Plugin SDK 型定義

### misskey-js について

- Misskey 本体モノレポ (`misskey-dev/misskey/packages/misskey-js`) で管理
- `api.APIClient` で全エンドポイントが型安全に呼べる
- `Stream` クラスで WebSocket 自動再接続対応
- ランタイム依存: `eventemitter3`, `reconnecting-websocket`, `autobind-decorator`

## 注意事項

- 新チャット機能（メッセージ）はローカルユーザー間のみ。リモートユーザーにはダイレクトノートで対応
- カスタム絵文字リアクションは Misskey 固有機能
- `openclaw/plugin-sdk` の型定義は `devDependencies` に置くこと

## マイルストーン

- [ ] **M1**: リポ scaffold + 型定義 + config-schema
- [ ] **M2**: Misskey Streaming 接続 + メンション受信
- [ ] **M3**: `notes/create` でメンション応答
- [ ] **M4**: 新チャット API でメッセージ送受信
- [ ] **M5**: セキュリティポリシー + onboarding
- [ ] **M6**: リアクション + メディア対応

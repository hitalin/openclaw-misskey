# OpenClaw Misskey Channel Plugin — 実装計画

## 概要

OpenClaw の channel plugin として Misskey を追加する。
Discord / Matrix と同様に、Misskey の DM・メンション・ノートを OpenClaw のエージェントが受信・応答できるようにする。

## 参考実装

- `/app/extensions/matrix/` — Matrix channel plugin（最も近い構造）
- `/app/extensions/bluebubbles/` — iMessage channel plugin

## Plugin SDK Interface

OpenClaw の `ChannelPlugin<T>` interface を実装する必要がある。

### 必須ファイル構成

```
openclaw-misskey/
├── openclaw.plugin.json    # プラグインマニフェスト
├── package.json            # npm パッケージ定義
├── index.ts                # エントリポイント
├── tsconfig.json
└── src/
    ├── channel.ts          # ChannelPlugin<T> 実装（メイン）
    ├── config-schema.ts    # Zod スキーマ（設定バリデーション）
    ├── types.ts            # 型定義
    ├── runtime.ts          # ランタイム参照の保持
    ├── onboarding.ts       # セットアップウィザード
    ├── actions.ts          # メッセージアクション（リアクション等）
    ├── outbound.ts         # 送信処理
    ├── resolve-targets.ts  # 宛先解決
    └── misskey/
        ├── index.ts        # Streaming 監視ループ
        ├── client.ts       # Misskey API クライアント
        ├── accounts.ts     # マルチアカウント管理
        ├── monitor.ts      # WebSocket Streaming 受信
        ├── send.ts         # ノート投稿 / DM 送信
        └── probe.ts        # 接続テスト
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
      "npmSpec": "@openclaw/misskey",
      "localPath": "extensions/misskey",
      "defaultChoice": "npm"
    }
  }
}
```

## ChannelPlugin<T> の主要実装ポイント

### Phase 1: 最小動作（DM 受信 → 応答）

1. **gateway.startAccount** — Misskey Streaming API (WebSocket) に接続し、メンション・DM を受信
2. **messaging** — `notes/create` で応答を投稿
3. **config** — インスタンス URL + API トークンの管理
4. **security** — DM ポリシー（pairing / allowlist / open）

### Phase 2: グループ対応

5. **groups** — チャンネル / ハッシュタグ監視
6. **directory** — ユーザー / グループ一覧
7. **threading** — リプライチェーンの追跡

### Phase 3: リッチ機能

8. **actions** — リアクション（カスタム絵文字対応）
9. **media** — 画像 / ファイル添付
10. **onboarding** — セットアップウィザード

## Misskey API で使うエンドポイント

### 受信（Streaming API）
- `wss://{host}/streaming` — WebSocket 接続
  - `main` channel: 自分宛の通知（メンション、DM、フォロー等）
  - `homeTimeline` / `localTimeline`: タイムライン監視（Phase 2）

### 送信
- `POST /api/notes/create` — ノート投稿
- `POST /api/messaging/messages/create` — DM 送信（Misskey v13+）
- `POST /api/notes/reactions/create` — リアクション

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
      "dmPolicy": "pairing",
      "groupPolicy": "disabled",
      "adminUsers": ["hitalin"],
      "visibility": "home"
    }
  }
}
```

## 依存パッケージ

- `misskey-js` — Misskey 公式 SDK（Streaming API 含む）
- `zod` — 設定スキーマバリデーション
- `openclaw` (devDependencies, workspace:*) — Plugin SDK 型定義

## 注意事項

- Misskey のバージョンによって API が異なる（v12 vs v13 vs v2024+）
- yami.ski は Misskey フォーク（Sharkey?）の可能性 → API 互換性要確認
- DM 機能は Misskey バージョンで実装が異なる（チャット vs ダイレクトノート）
- カスタム絵文字リアクションは Misskey 固有機能
- レートリミット: Misskey のデフォルトは概ね寛容だが、自前インスタンスなので問題なし

## マイルストーン

- [ ] **M1**: リポ scaffold + 型定義 + config-schema
- [ ] **M2**: Misskey Streaming 接続 + メンション受信
- [ ] **M3**: `notes/create` で応答投稿
- [ ] **M4**: DM 対応 + セキュリティポリシー
- [ ] **M5**: Ansible ロール統合（Dockerfile.j2 に追加）
- [ ] **M6**: リアクション + メディア対応

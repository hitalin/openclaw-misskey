# openclaw-misskey

[OpenClaw](https://github.com/openclaw/openclaw) channel plugin for [Misskey](https://misskey-hub.net/).

Misskey のメッセージ・メンション・ノートを OpenClaw エージェントで受信・応答できるようにする。

## Requirements

- Node.js >= 22
- OpenClaw >= 2026.x
- Misskey >= 2025.4.0（新チャット機能搭載）

## Installation

```bash
# npm からインストール（リリース後）
openclaw install @openclaw/misskey

# ローカル開発
git clone https://github.com/yamisskey-dev/openclaw-misskey.git
cd openclaw-misskey
pnpm install
```

## Configuration

`openclaw.json` に以下を追加:

```json
{
  "channels": {
    "misskey": {
      "enabled": true,
      "host": "https://your-instance.example.com",
      "token": "${MISSKEY_TOKEN}",
      "messagePolicy": "pairing",
      "visibility": "home"
    }
  }
}
```

### API トークンの取得

1. Misskey インスタンスの **設定 → API** からアクセストークンを生成
2. 必要な権限: `read:account`, `write:notes`, `read:notifications`, `write:chat`
3. 環境変数 `MISSKEY_TOKEN` にセットするか、設定に直接記述

### 設定項目

| 項目 | 型 | デフォルト | 説明 |
|------|-----|-----------|------|
| `host` | string | — | Misskey インスタンスの URL |
| `token` | string | — | API アクセストークン |
| `messagePolicy` | string | `"pairing"` | メッセージ受付ポリシー (`pairing` / `allowlist` / `open`) |
| `visibility` | string | `"home"` | ノート投稿時の公開範囲 (`public` / `home` / `followers` / `specified`) |
| `adminUsers` | string[] | `[]` | 管理者ユーザー名 |

## Features

### Phase 1（初期リリース）

- Streaming API (WebSocket) による常時接続
- メンション受信 → ノートで応答
- メッセージ（チャット）受信 → メッセージで応答
- メッセージポリシーによるアクセス制御

### Phase 2

- チャットルーム / チャンネル監視
- リプライチェーン追跡
- ユーザーディレクトリ

### Phase 3

- カスタム絵文字リアクション
- 画像 / ファイル添付
- セットアップウィザード

## Development

```bash
pnpm install
pnpm dev      # 開発モード
pnpm build    # ビルド
pnpm test     # テスト
```

## License

[MIT](LICENSE)

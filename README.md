# XIMP — eXtensible Internet Messaging Protocol

Version 1.0-draft（1.1 追補統合済み）。

仕様書は [`spec/`](spec/) に内容ごとに分割しています。目次は [spec/README.md](spec/README.md) を参照してください。

## クイックリンク

- [00-overview.md](spec/00-overview.md) — キーワード、範囲、用語、バイト列
- [01-wire-format.md](spec/01-wire-format.md) — 外枠、要素 TLV、ホスト名比較、証明書
- [02-identifiers-framing-types.md](spec/02-identifiers-framing-types.md) — 識別子
- [03-framing-types.md](spec/03-framing-types.md) — 外枠 Type
- [04-element-registry.md](spec/04-element-registry.md) — 要素レジストリ
- [05-errors-sessions.md](spec/05-errors-sessions.md) — エラーコード、セッション状態
- [06-auth.md](spec/06-auth.md) — 認証、チケット、S2S
- [07-community-channel.md](spec/07-community-channel.md) — コミュニティ、チャネル
- [08-messaging.md](spec/08-messaging.md) — メッセージ、同期、CHUNK、REACTION、Mention、検索、アバター、編集・削除
- [09-permissions.md](spec/09-permissions.md) — 権限
- [10-bot-commands.md](spec/10-bot-commands.md) — Bot、Slash Command、Webhook、サーバ間
- [11-server-minimum.md](spec/11-server-minimum.md) — 実装が持たないもの、最小実装チェック
- [12-integrated-additions.md](spec/12-integrated-additions.md) — 追加仕様（統合済み）
- [13-appendix.md](spec/13-appendix.md) — 付録 A/B/C

単一ファイル版: [eXtensible Internet Messaging Protocol.md](eXtensible%20Internet%20Messaging%20Protocol.md)
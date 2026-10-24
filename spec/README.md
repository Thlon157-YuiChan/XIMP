# XIMP 仕様書（分割版）

eXtensible Internet Messaging Protocol — Version 1.0-draft + 1.1 追補。

元ファイル: `../eXtensible Internet Messaging Protocol.md`（単一ファイル版）。

## 目次

1. [00-overview.md](00-overview.md) — キーワード、範囲、用語、バイト列、識別子
2. [01-wire-format.md](01-wire-format.md) — 外枠、要素 TLV、ホスト名比較、証明書
3. [02-framing-types.md](02-framing-types.md) — 外枠 Type、HELLO 要素
4. [03-element-registry.md](03-element-registry.md) — 要素レジストリ、Avatar 構造
5. [04-errors-sessions.md](04-errors-sessions.md) — エラーコード、セッション状態
6. [05-auth.md](05-auth.md) — 認証、チケット、S2S 確認
7. [06-community-channel.md](06-community-channel.md) — コミュニティ、チャネル、外部切り替え
8. [07-messaging.md](07-messaging.md) — メッセージ、同期、CHUNK、REACTION、Mention、検索、アバター、編集・削除
9. [08-permissions.md](08-permissions.md) — 権限（コミュニティ 32-bit + Host-Mask）
10. [09-bot-commands.md](09-bot-commands.md) — Bot、Slash Command、Webhook、サーバ間
11. [10-server-minimum.md](10-server-minimum.md) — 実装が持たないもの、最小実装チェック
12. [11-v1.1.md](11-v1.1.md) — 1.1 追補（ROSTER、Presence、DM、ban、host-role 等）
13. [12-appendix.md](12-appendix.md) — 付録 A/B/C

## 編集時の注意
- 各ファイルの `##` 見出しは元仕様の節番号を維持している。
- 要素レジストリ（03）と 1.1 追補（11）の要素コードは重複しないよう管理する。
- 単一ファイル版が必要な場合は `../eXtensible Internet Messaging Protocol.md` を参照。
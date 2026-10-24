# XIMP 仕様書（分割版）

eXtensible Internet Messaging Protocol — Version 1.0-draft（1.1 追補統合済み）。

元ファイル: `../eXtensible Internet Messaging Protocol.md`。

## 目次

1. [00-overview.md](00-overview.md) — キーワード、範囲、用語、バイト列（§1-3）
2. [01-wire-format.md](01-wire-format.md) — 外枠、要素 TLV、ホスト名比較、証明書（§3.1-3.4）
3. [02-identifiers-framing-types.md](02-identifiers-framing-types.md) — 識別子（§4）
4. [03-framing-types.md](03-framing-types.md) — 外枠 Type（§5）
5. [04-element-registry.md](04-element-registry.md) — 要素レジストリ（§6）
6. [05-errors-sessions.md](05-errors-sessions.md) — エラーコード、セッション状態（§7-8）
7. [06-auth.md](06-auth.md) — 認証、チケット、S2S（§9）
8. [07-community-channel.md](07-community-channel.md) — コミュニティ、チャネル（§10-11）
9. [08-messaging.md](08-messaging.md) — メッセージ、同期、CHUNK、REACTION、Mention、検索、アバター、編集・削除（§12）
10. [09-permissions.md](09-permissions.md) — 権限（§13）
11. [10-bot-commands.md](10-bot-commands.md) — Bot、Slash Command、Webhook、サーバ間（§14-15）
12. [11-server-minimum.md](11-server-minimum.md) — 実装が持たないもの、最小実装チェック（§16-17）
13. [12-integrated-additions.md](12-integrated-additions.md) — 追加仕様（ROSTER、Presence、クロスホスト、DM、ban、ホスト権限等）（§18）
14. [13-appendix.md](13-appendix.md) — 付録 A/B/C

## 編集時の注意
- 各ファイルの `##` 見出しは元仕様の節番号を維持している。
- 要素レジストリ（04）と追加仕様（12）の要素コードは重複しないよう管理する。
- 単一ファイル版が必要な場合は `../eXtensible Internet Messaging Protocol.md` を参照。
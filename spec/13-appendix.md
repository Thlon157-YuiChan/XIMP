## 付録 A. 予約項目（統合済み）

旧 1.1 で予約だった項目は §18 および各節に統合済み:
- ROSTER の詳細 — §18.1
- emoji-resolve の IQ 形式 — §18.3
- `Avatar-Ref` の `PRESENCE` 以外での利用 — §18.4
- アバターの一括取得 IQ — §18.5
- `contacts` の評価 — §18.6（撤廃、全公開）
- クロスホスト Presence — §18.8
- クロスホスト DM — §18.9
- dm-policy / ブロックリスト — §18.10
- コミュニティ ban — §18.11
- ホストロール — §18.12
- ホスト所有権譲渡 — §18.13

RECEIPT（`0x12`）は仕様から除去（§18.7）。

## 付録 B. 非規範の方式名

| Proto | 接続先の例 |
|---|---|
| `webrtc` | `wss://sfu.example/room` |
| `http` | `https://files.example/drop/abc` |
| `udp` | `203.0.113.10:9000` |

## 付録 C. 非規範の証明書運用

- パブリック CA（Let's Encrypt など）を使ってよい（MAY）。
- ACME で自動更新してよい（MAY）。
- OCSP stapling を使う SHOULD。
- Certificate Transparency の SCT を検証してよい（MAY）。
- クライアント証明書は使わない（MUST NOT）。mTLS を要求するサーバは XIMP 準拠ではない（MUST NOT）。


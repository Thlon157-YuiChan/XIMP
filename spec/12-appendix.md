## 付録 A. 1.0 で予約のままの項目

- presence-subscribe の IQ 形式（Event-Subscribe bit 2 で代替可）
- `Avatar-Ref` の `PRESENCE` 以外での利用（`ROSTER`、`MESSAGE`）— 1.1 §18.4 で許可
- `contacts` の評価で ROSTER を持たない場合の扱い — 1.1 §18.6
- ROSTER の詳細 — 1.1 §18.1 で定義
- emoji-resolve の IQ 形式 — 1.1 §18.3 で定義
- アバターの一括取得 IQ — 1.1 §18.5 で定義

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


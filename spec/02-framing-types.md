## 5. 外枠 Type

| 値 | 名前 | 意味 |
|---|---|---|
| `0x00` | RESERVED | 送信禁止（MUST NOT）。 |
| `0x01` | HELLO | 能力と上限。 |
| `0x02` | AUTH | ユーザ認証、またはチケット提示。 |
| `0x03` | AUTH_RESULT | 成否と正規 From。 |
| `0x04` | ERROR | エラー。 |
| `0x05` | PING | 生存確認。Value は任意クッキー。 |
| `0x06` | PONG | PING のクッキーを返す。 |
| `0x07` | BYE | 正常切断。 |
| `0x08` | S2S_AUTH | 場からホームへのチケット確認。 |
| `0x10` | MESSAGE | 本文。ライブも履歴もこの枠。 |
| `0x11` | ACK | ホストが正本として保存した。 |
| `0x12` | RESERVED | 本版では送信禁止（MUST NOT）。 |
| `0x13` | REACTION | リアクションの追加・削除・集約。保存する。履歴に出してよい。 |
| `0x20` | PRESENCE | 在席。保存しない。履歴に出さない。 |
| `0x21` | ROSTER | 連絡先。詳細は 1.1 で定義。 |
| `0x30` | IQ | 要求と応答。 |
| `0x40` | GROUP | 本版では送信禁止（MUST NOT）。受信したら unsupported-type。 |
| `0x50` | CHUNK | 64 KiB を超える断片。 |
| `0xF0`–`0xFE` | EXPERIMENTAL | 本番禁止（MUST NOT）。 |
| `0xFF` | RESERVED | 送信禁止（MUST NOT）。 |

- AUTH 前に送ってよいのは `HELLO / AUTH / ERROR / PING / PONG / BYE` だけでなければならない（MUST）。
- S2S で送ってよいのは `HELLO / S2S_AUTH / AUTH_RESULT / ERROR / PING / PONG / BYE` だけでなければならない（MUST）。
- REACTION は MESSAGE と同じ配送規則に従わなければならない（MUST）。ライブは READ がある接続にだけ push し（MUST）、履歴は sync-get の明示要求時だけ返す（MUST）。
- Version はメジャーのみでなければならない（MUST）。互換な追加は Capability で交渉しなければならない（MUST）。未知 Version は `unsupported-version` としなければならない（MUST）。HELLO の Capability は `v:<major>` と `feat:<name>` を含む SHOULD。受信側は未知の `feat:` を無視しなければならない（MUST）。
- HELLO の必須要素:
  - Version（外枠）: MUST
  - Capability（1 個以上、C=1）: MUST
  - Max-Size（C=1）: SHOULD
  - Software（C=0）: MAY
- クライアントとサーバは 1 接続につき 1 回だけ HELLO を送らなければならない（MUST）。再送は `malformed` としなければならない（MUST）。


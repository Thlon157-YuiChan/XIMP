## 10. コミュニティと入場

1 ホストが複数のコミュニティを持ってよい（MAY）。選択に SNI を使ってはならない（MUST NOT）。

作成時、サーバは作成者に owner ロールを付けなければならない（MUST）。ID を返さなければならない（MUST）。
- コミュニティ作成には CREATE_COMMUNITY または HOST_OWNER が必要である（MUST）。権限が無ければ `not-allowed` を返す（MUST）。

```
IQ community-create   Community-Name
IQ community-list     公開分、または自分が入れる分
IQ community-join     Community-ID, Admit-Data?
IQ community-leave    Community-ID
```

Admit-Mode:

| 値 | 意味 |
|---|---|
| 0 | 公開。Admit-Data 不要。 |
| 1 | パスフレーズ。 |
| 2 | 招待トークン。 |
| 3 以上 | 場の実装。外枠は同じ。 |

- list の各行は Community-ID、Community-Name、Admit-Mode でなければならない（MUST）。
- join の規則:
  1. 身分確定済みか確認しなければならない（MUST）。
  2. 既メンバーなら成功とする（MUST）。鍵を再要求しない（変更後だけ必須にするのは場の方針）。
  3. Mode 0 ならメンバー表に足し、member ロールを付けなければならない（MUST）。
  4. それ以外は Admit-Data を照合しなければならない（MUST）。成功なら同様に足す。失敗は `not-allowed` としなければならない（MUST）。
  5. 失敗回数を Community-ID + EID で制限しなければならない（MUST）。
  6. Admit-Data をログに出してはならない（MUST NOT）。ホームへ転送してはならない（MUST NOT）。
- 表示名はホスト内で一意とする SHOULD。衝突は `conflict` としなければならない（MUST）。
- コミュニティ作成は rate limit の対象としなければならない（MUST）。ユーザ単位・接続元単位で制限し、超過は `rate-limited` としなければならない（MUST）。
- ban:
  - `IQ set community-ban`  Community-ID, Target EID, Ban-Action (0=ban 1=unban), Ban-Reason?（MAY）
  - 権限: KICK または MANAGE_ADMIT（MUST）。
  - ban された EID は community-join が `not-allowed` となる（MUST）。既メンバーなら除名する（MUST）。
  - Ban-Reason は運用者向けであり、被 ban 者に返してはならない（MUST NOT）。

## 11. チャネル

```
IQ channel-create   Community-ID, Channel-Name
IQ channel-list     Community-ID
IQ channel-rename   Community-ID, Channel-ID, Channel-Name
IQ channel-join     Community-ID, Channel-ID
IQ channel-leave    Community-ID, Channel-ID
```

- create の応答に Channel-ID を付けなければならない（MUST）。
- 表示名はコミュニティ内で一意とする SHOULD。
- list は、呼び出し元の実効権限に READ が残るチャネルだけ返さなければならない（MUST）。
- join には JOIN_CHANNEL と READ が要る（MUST）。Kind=external では加えて EXTERNAL_JOIN が要る（MUST）。

### 11.1 別プロトコルへの切り替え

- XIMP は媒体の中身を解釈してはならない（MUST NOT）。
- 外部チャネルは、join 成功の応答で 1 個以上の External-Offer を返さなければならない（MUST）。1 チャネルに方式は複数あってよい（MAY）。
- External-Offer の中身:

| 要素 | 必須 | 意味 |
|---|---|---|
| External-Proto（`0x0065`） | はい（MUST） | 方式名。仕様は登録しない。例は付録を参照。 |
| External-Target（`0x0066`） | はい（MUST） | 接続先。`host:port` または URI。 |
| External-Token（`0x0067`） | いいえ（MAY） | その接続先だけに有効な短命なバイト列。ログに出さない。 |

- クライアントは HELLO で `ext:<proto>` を知っている方式ごとに広告する SHOULD。
- Offer のうち、広告した方式が一つも無ければ join は `unsupported-type` としなければならない（MUST）。一つでもあれば、クライアントが自分で 1 つ選ばなければならない（MUST）。サーバはどれを使うか指定してはならない（MUST NOT）。
- 規則:
  - SDP、コーデック、媒体パケットは XIMP に載せてはならない（MUST NOT）。
  - Offer、Target、Token を正本にしてはならない（MUST NOT）。`sync-get` で返してはならない（MUST NOT）。join のたびに発行しなければならない（MUST）。
  - 同じ Offer 内の Proto は 1 つでなければならない（MUST）。同名の Offer が複数あるときは、クライアントは先に出たものを使う SHOULD。
  - Token をログ、ERROR、ACK に出してはならない（MUST NOT）。
  - leave、切断、権限剥奪で、発行した Token を各接続先が拒否できるようにしなければならない（MUST）。拒否の仕組みは Proto の外。
  - 別ホストの参加者を S2S で媒体に引き込んではならない（MUST NOT）。そのホストへ自分で繋ぐ。
- create 時に、そのチャネルの Offer 雛形（Proto と Target）を複数保存してよい（MAY）。Token は join のたびに場が作る（MUST）。

方式名の例（非規範。この表に無い名前も送ってよい（MAY））:

| Proto | 接続先の例 |
|---|---|
| `webrtc` | `wss://sfu.example/room` |
| `http` | `https://files.example/drop/abc` |
| `udp` | `203.0.113.10:9000` |


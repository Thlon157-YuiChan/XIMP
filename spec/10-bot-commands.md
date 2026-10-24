## 14. Bot と Slash Command

### 13.1 ホストレベル権限（Host-Mask）

コミュニティのロールマスクとは独立した 32 ビットマスクをホストレベルで使う（MUST）。

| bit | 名前 | 意味 |
|---|---|---|
| 0 | CREATE_COMMUNITY | コミュニティを作成できる。 |
| 1 | MANAGE_HOST | ホスト設定・ポリシーを変更できる。 |
| 2 | GLOBAL_BAN | ホスト全体の ban を実行できる。 |
| 3 | VIEW_AUDIT | 監査ログを参照できる（実装が提供する場合）。 |
| 4–30 | 未割当 | 未知なら無視。 |
| 31 | HOST_OWNER | 全ビットを含む。OR では作れない。最後の 1 人は外せない。 |

- コミュニティのロールマスクと Host-Mask は別々に評価しなければならない（MUST）。混在してはならない（MUST NOT）。
- コミュニティ作成には CREATE_COMMUNITY または HOST_OWNER が必要である（MUST）。
- HOST_OWNER はデプロイ時に設定しなければならない（MUST）。プロトコルで付与してはならない（MUST NOT）。
- Host-Role の管理には MANAGE_HOST または HOST_OWNER が必要である（MUST）。
- Host-Role の付与は、自分が持つビットの範囲に限らなければならない（MUST NOT 超過）。
- Host-Mask をコミュニティのロールマスクに OR してはならない（MUST NOT）。

IQ（ホスト接続）:

```
IQ get  host-role-list
IQ set  host-role-set     Host-Role-EID, Host-Mask
IQ set  host-role-remove  Host-Role-EID
```

- `host-role-set` は Host-Mask を EID に付与する（MUST）。既存値は上書きする（MUST）。
- `host-role-remove` は EID の Host-Mask を 0 にする（MUST）。HOST_OWNER の EID は削除してはならない（MUST NOT）。

### 14.1 Bot の身分

- Bot も `bot@domain` のような EID を持たなければならない（MUST）。
- ホームが資格情報を発行しなければならない（MUST）。
- 認証は通常ユーザと同じ `home-ticket` でなければならない（MUST）。
- 場は「Bot である」ことを EID で判断してはならない（MUST NOT）。
- Bot であることを示すには `Bot` 要素（`0x00D0`）をメッセージに付けてよい（SHOULD）。
- 権限は通常のロールとマスクで扱わなければならない（MUST）。

### 14.2 イベント購読

```
IQ set
  Message-ID     = <iq-30>
  IQ-Kind        = 2
  <Event-Subscribe>
  Community-ID   = <cid>
  Channel-ID     = <chid>（任意）
  Event-Kind     = <ビットマスク>
```

応答:

```
IQ result
  Message-ID     = <iq-30>
  IQ-Kind        = 3
  <Subscription-ID>
```

イベント種別:

| bit | 名前 | 意味 |
|---|---|---|
| 0 | MESSAGE | 新規メッセージ |
| 1 | REACTION | リアクション |
| 2 | PRESENCE | 在席 |
| 3 | MEMBER | メンバー変更 |
| 4 | ROLE | ロール変更 |
| 5 | ACL | ACL 変更 |
| 6 | CHANNEL | チャネル変更 |
| 7–31 | 予約 | |

購読権限:

- MESSAGE の購読には READ が要る（MUST）。
- REACTION の購読には READ が要る（MUST）。
- PRESENCE の購読には READ が要る（MUST）。
- MEMBER の購読には KICK か MANAGE_ROLE が要る（MUST）。
- ROLE の購読には MANAGE_ROLE が要る（MUST）。
- ACL の購読には MANAGE_CHANNEL が要る（MUST）。
- CHANNEL の購読には READ が要る（MUST）。
- 購読は接続ごとでなければならない（MUST）。接続が切れたら消えなければならない（MUST）。
- 配送形式は通常の MESSAGE、REACTION、PRESENCE と同じでなければならない（MUST）。
- 配信時は購読時の権限に加え、配信時点の実効権限を再評価しなければならない（MUST）。権限を失った購読者へ配信してはならない（MUST NOT）。

### 14.3 Slash Command

構文:

```
/[namespace]:[command] [args]
```

- `namespace` は UTF-8 で 1..32 オクテットでなければならない（MUST）。`[a-z0-9_-]` とする SHOULD。
- 大文字は小文字に正規化する MUST。
- `command` は UTF-8 で 1..32 オクテットでなければならない（MUST）。`[a-z0-9_-]` とする SHOULD。
- 大文字は小文字に正規化する MUST。
- 予約名前空間は `ximp`、`system`、`server`、`local` でなければならない（MUST）。
- 予約名前空間を Bot が使ったら `not-allowed` としなければならない（MUST）。
- 同じコミュニティで同じ名前空間を持つ Bot は 1 つだけとする SHOULD。
- サーバは登録時に重複を検出し、`conflict` を返さなければならない（MUST）。
- `namespace` は Bot の EID の localpart とする SHOULD。
- 短縮形は定義してはならない（MUST NOT）。
- クライアントはオートコンプリートで補完してよい（MAY）。表示は短縮形でもよい（MAY）。ワイヤ上は完全修飾でなければならない（MUST）。

登録:

```
IQ set
  Message-ID         = <iq-31>
  IQ-Kind            = 2
  <Command-Register>
  Community-ID       = <cid>
  Command-Namespace  = "pollbot"
  Command-Name       = "poll"
  Command-Desc       = "Create a poll"
  Command-Scope      = 0（コミュニティ）/ 1（チャネル）
```

実行:

```
IQ set
  Message-ID         = <iq-32>
  IQ-Kind            = 2
  <Command-Invoke>
  Community-ID       = <cid>
  Channel-ID         = <chid>
  Command-Namespace  = "pollbot"
  Command-Name       = "poll"
  Command-Args       = "question='lunch?' options='ramen,sushi'"
```

- サーバはコマンドを登録した Bot に配送しなければならない（MUST）。
- Bot は結果を MESSAGE で返さなければならない（MUST）。
- クライアントは結果を表示する（MUST）。
- 登録には bit 11 COMMAND_REGISTER が要る（MUST）。
- 実行には READ と WRITE が要る（MUST）。
- `Command-Invoke` はユーザ単位・Bot 単位で rate limit の対象としてよい（MAY）。超過は `rate-limited` としなければならない（MUST）。

### 14.4 Webhook

- プロトコルに定義してはならない（MUST NOT）。
- サーバ実装が独自に提供してよい（MAY）。
- Webhook から投稿する場合、Bot 用の EID と `Bot` 要素を使う SHOULD。
- XIMP 側は通常の MESSAGE として受け取らなければならない（MUST）。


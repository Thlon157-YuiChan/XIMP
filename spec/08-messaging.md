## 12. メッセージと同期

チャネル投稿:

```
MESSAGE
  Message-ID  C=1
  From        C=1
  Community-ID C=1
  Channel-ID  C=1
  Body または Payload
  Timestamp 任意
  To を付けてはならない
```

1対1（同一ホスト内のみ）:

```
MESSAGE
  Message-ID, From, To
  Community-ID と Channel-ID を付けてはならない
```

- 両方ある、またはチャネル投稿なのに ID が無いものは `malformed` としなければならない（MUST）。
- サーバは MESSAGE の `From` が認証済み EID と一致することを確認しなければならない（MUST）。不一致は `malformed` または `auth-failed` としなければならない（MUST）。
- クライアント指定の `Timestamp` を権威的として信頼してはならない（MUST NOT）。サーバは受信時に自身の時刻で `Timestamp` を設定または検証しなければならない（MUST）。
- サーバは保存してから ACK を返さなければならない（MUST）。ACK の Reply-To-ID は対象 Message-ID でなければならない（MUST）。意味は「相手が読んだ」ではなく「このホストが正本として預かった」である。
- 同一接続で同一 Message-ID の再送は冪等に扱わなければならない（SHOULD）。重複保存してはならない（MUST NOT）。
- ホストはユーザ単位・チャネル単位で MESSAGE の rate limit を適用してよい（MAY）。超過は `rate-limited` としなければならない（MUST）。
- ライブは、そのコミュニティのメンバーで、そのチャネルを join し、READ がある接続にだけ push しなければならない（MUST）。ライブの MESSAGE は Reply-To-ID を付けてはならない（MUST NOT）。ライブの Reply-To-ID に sync の IQ ID を付けてはならない（MUST NOT）。
- 履歴は明示要求だけでなければならない（MUST）。AUTH や join の直後に流してはならない（MUST NOT）。

```
C → S  IQ get
         Sync-Scope = 2
         Community-ID, Channel-ID
         Sync-Limit 省略時 16、0 禁止
         Sync-Cursor 省略 = 最新から
S → C  MESSAGE × 最大 Limit 件（新しい順）
         Reply-To-ID = その IQ の Message-ID
         Stored を付けてよい
S → C  IQ result
         Sync-Cursor, More
```

- カーソルは不透明でなければならない（MUST）。無効なら `not-found` としなければならない（MUST）。クライアントはカーソル無しでやり直してよい（MAY）。
- チャネルの `Sync-Limit` 上限は 100 としなければならない（MUST）。超過は `malformed` としなければならない（MUST）。
- ライブが履歴の途中に割り込んでよい（MAY）。見分けは Reply-To-ID でなければならない（MUST）。
- inbox（Scope=1）は同じホスト上の 1対1 である。Scope=3 は定義しない（MUST NOT）。
- inbox（Scope=1）:
  - 同じホスト上の 1対1 メッセージのみ（MUST）。
  - Sync-Cursor はチャネルと同じ形式で不透明（MUST）。
  - 順序は Timestamp の降順。同時刻は Message-ID の昇順（MUST）。
  - Sync-Limit の既定は 16。上限は 100（MUST）。
  - Scope=3 以上は予約。受信したら `unsupported-type` としなければならない（MUST）。
- IQ には Message-ID を C=1 で付けなければならない（MUST）。クライアントは Reply-To-ID の有無でライブと履歴を分けなければならない（MUST）。Stored は履歴にだけ付けてよい（MAY）。

CHUNK:

```
Transfer-ID  C=1  16 オクテット
Offset       C=1  8 オクテット符号なし。バイト単位。
Total-Size   C=1  8 オクテット符号なし。全断片で同じ値。
Payload      C=1  この断片のバイト列
```

規則:

- Offset は 0 起点でなければならない（MUST）。Total-Size は 1 以上でなければならない（MUST）。Total-Size はサーバの受信上限を超えてはならない（MUST）。超過は `too-large` としなければならない（MUST）。
- Offset + len(Payload) <= Total-Size でなければ `malformed` としなければならない（MUST）。
- 同じ Transfer-ID の断片は、Offset の昇順で重複なく揃えなければならない（MUST）。重複または重なりは `conflict` としなければならない（MUST）。欠落は完了まで待つ（MUST）。
- 断片の合計が Total-Size に達したら完了とする（MUST）。それ以外は正本にしてはならない（MUST NOT）。
- 途中の断片をライブ配送してはならない（MUST NOT）。完了後に 1 回だけ配送しなければならない（MUST）。
- 同一 Transfer-ID の並行受信は 1 つまでとする（MUST）。別 Transfer-ID は接続あたり 4 つまで並行してよい（MAY）。それを超えたら `rate-limited` としなければならない（MUST）。
- 最初の断片から 10 分以内に完了しなければ破棄しなければならない（MUST）。
- 断片間隔が 60 秒を超えた転送は破棄してよい（MAY）。
- ホストはユーザ単位の保留中転送数（推奨 8）と保留中バイト数（推奨 64 MiB）の上限を持たなければならない（MUST）。超過は `rate-limited` または `mailbox-full` としなければならない（MUST）。
- 権限チェックは各断片の受信時に ATTACH で行い、完了時にもう一度 ATTACH と WRITE を見なければならない（MUST）。
- CHUNK は MESSAGE と同じく ACK の対象になる（MUST）。ACK は完了時だけ返さなければならない（MUST）。Reply-To-ID は完了したメッセージの Message-ID でなければならない（MUST）。

- PRESENCE は保存してはならない（MUST NOT）。同じコミュニティのメンバーで READ を持つ接続に push しなければならない（MUST）。最新を接続中の相手へ送るだけである（MUST）。購読要求は IQ presence-subscribe で行う（MUST）。ROSTER による追加フィルタは行わない（MUST NOT）。

### 12.1 REACTION

```
REACTION
  Message-ID       C=1
  From             C=1
  Community-ID     C=1
  Channel-ID       C=1
  Reaction-Target  C=1
  Reaction-Key     C=1
  Reaction-Op      C=1
  Timestamp        任意
```

- Reaction-Target は対象メッセージの Message-ID でなければならない（MUST）。
- Reaction-Key は不透明でなければならない（MUST）。Unicode 絵文字でも、`custom:12345` でも、`emoji:smile` でもよい（MAY）。
- Reaction-Op は 0=add 1=remove 2=clear でなければならない（MUST）。
- clear は Reaction-Key を省略してよい（MAY）（自分の全リアクションを消す）。
- サーバは保存してから ACK を返さなければならない（MUST）。ACK の Reply-To-ID は REACTION の Message-ID でなければならない（MUST）。
- ライブ配送:
  - そのコミュニティのメンバーで、そのチャネルを join し、READ がある接続にだけ push しなければならない（MUST）。
  - ライブの REACTION は Reply-To-ID を付けてはならない（MUST NOT）。
  - 削除されたメッセージへの REACTION は push してはならない（MUST NOT）。
- 履歴:
  - 既定では返してはならない（MUST NOT）。
  - Sync-Options の include-reactions があるときだけ、Reaction-Group で集約して返さなければならない（MUST）。
  - 削除されたメッセージのリアクションは返してはならない（MUST NOT）。
- 集約:
  - 同じメッセージに同じ Key のリアクションが複数ある場合、サーバが集約して Reaction-Count を返す SHOULD。
  - 誰が付けたかを見たい場合、Reaction-Group に Actor の EID を含めてよい（MAY）。
  - 全部の Actor を返すか、先頭 N 件だけかはコミュニティのポリシーが決めてよい（MAY）。
- カスタム絵文字:
  - Reaction-Key が `custom:` で始まる場合、続きはコミュニティ内の ID である（MUST）。
  - 解決は `IQ emoji-resolve`（予約）で行ってよい（MAY）。Community-ID と Emoji-ID を渡し、URL や画像データを得る。

### 12.2 Mention

```
MESSAGE
  Message-ID       C=1
  From             C=1
  Community-ID     C=1
  Channel-ID       C=1
  Body             C=1
  Mention          C=0 × N
    Mention-Kind   = 0（user）
    Mention-Target = alice@example.com
    Mention-Offset = 0
    Mention-Length = 6
    Mention-Text   = "@alice"
  Mention          C=0
    Mention-Kind   = 2（everyone）
    Mention-Text   = "@everyone"
```

- Mention は C=0 でなければならない（MUST）。未知の実装は飛ばさなければならない（MUST）。
- サーバは Mention を正本に保存しなければならない（MUST）。
- サーバは Mention を履歴でも返す SHOULD。`Sync-Options` で抑止できるようにしてもよい（MAY）。
- メンションの種類は user / role / everyone / here でなければならない（MUST）。
- `@here` はサーバが送信時点でオンラインのメンバーを特定しなければならない（MUST）。その人たちにだけ「here メンション」として配送する。保存される Mention は `Mention-Kind = 3`（here）のままでなければならない（MUST）。
- `@everyone` と `@here` を使うには bit 10 MENTION_EVERYONE が要る（MUST）。
- ロールメンションを使うには、そのロールを持つか bit 10 が要る（MUST）。
- 同じ Mention-Target が複数あってよい（MAY）。完全重複（Kind、Target、Offset、Length がすべて同じ）は `malformed` としなければならない（MUST）。
- 通知の重複排除はクライアントが行う SHOULD。

### 12.3 検索

```
IQ get
  Message-ID    = <iq-20>
  IQ-Kind       = 1
  Search-Scope  = 1（コミュニティ）/ 2（チャネル）
  Community-ID  = <cid>
  Channel-ID    = <chid>（Scope=2 のとき）
  Search-Query  = <UTF-8。クエリ言語はサーバ実装>
  Search-Limit  = 2 オクテット。省略時 32。0 禁止。
  Search-Cursor = 不透明。省略時は先頭から。
  Search-From   = 任意。EID。
  Search-After  = 任意。Unix ミリ秒。
  Search-Before = 任意。Unix ミリ秒。
```

応答:

```
IQ result
  Message-ID    = <iq-20>
  IQ-Kind       = 3
  Search-Result × N
    Message-ID  = <m-1>
    Channel-ID  = <chid>
    From        = <eid>
    Timestamp   = <ms>
    Snippet     = <UTF-8。抜粋>
  Search-Cursor = 次のカーソル
  More          = 0/1
```

- 検索には READ が要る（MUST）。
- `Search-Limit` の上限は 128 としなければならない（MUST）。超過は `malformed` としなければならない（MUST）。
- 結果は呼び出し元の実効権限に READ が残るチャネルだけ返さなければならない（MUST）。
- 権限が変わったら、次のページから新しい権限で返さなければならない（MUST）。
- クライアントが手元のキャッシュを検索するのは自由である（MAY）。
- 標準はサーバ側検索を前提としなければならない（MUST）。
- クエリ言語、ランキング、インデックス、全文検索エンジン、添付の中身はサーバ実装が決めてよい（MAY）。
- サーバは権限のないチャネルの内容をインデックスに含めてはならない（MUST NOT）。Snippet は 256 オクテットを超えてはならない（MUST）。

### 12.4 アバター

アバターはホームの既定とコミュニティの上書きの 2 段階で管理しなければならない（MUST）。

#### 解決の順序

1. コミュニティの上書きアバターがあるか確認しなければならない（MUST）。
2. なければホームの既定アバターを使わなければならない（MUST）。
3. どちらもなければ `not-found` としなければならない（MUST）。

- クライアントは場に IQ で問い合わせなければならない（MUST）。
- 場は上書きがあればそれを返さなければならない（MUST）。なければホームへ S2S で問い合わせて返す SHOULD。
- 場は結果を短期間キャッシュしてよい（MAY）。

#### 取得

```
IQ get
  Message-ID          = <iq-40>
  IQ-Kind             = 1
  <Avatar-EID>        = alice@example.com
  <Avatar-Known-Hash> = <hash-1>（任意）
```

応答:

```
IQ result
  Message-ID          = <iq-40>
  IQ-Kind             = 3
  <Avatar> × N
```

- `Avatar-Known-Hash` を送ると、場は変化がなければ `Avatar` を省略してよい（MAY）。
- `Avatar` は複数サイズを返してよい（MAY）。
- 見られない場合は `not-allowed` としなければならない（MUST）。

#### 登録

ホームの既定アバター:

```
IQ set
  Message-ID          = <iq-42>
  IQ-Kind             = 2
  <Avatar-Upload>
  <Avatar-MIME>       = "image/webp"
  <Payload>           = <画像のバイト列>
```

コミュニティの上書きアバター:

```
IQ set
  Message-ID          = <iq-43>
  IQ-Kind             = 2
  <Community-ID>      = <cid>
  <Avatar-Upload>
  <Avatar-MIME>       = "image/webp"
  <Payload>           = <画像のバイト列>
```

- ホーム全体のアバターはホームに送らなければならない（MUST）。
- コミュニティごとの上書きは場に送らなければならない（MUST）。
- 同じ `Avatar-Upload` の再送は上書きとしなければならない（MUST）。
- `Avatar-Updated` を更新しなければならない（MUST）。

#### 削除

```
IQ set
  Message-ID          = <iq-47>
  IQ-Kind             = 2
  <Avatar-Delete>
  <Community-ID>      = <cid>（上書きの削除のとき）
```

- ホームの既定を削除する場合、`Community-ID` を付けてはならない（MUST NOT）。
- コミュニティの上書きを削除する場合、`Community-ID` を付けなければならない（MUST）。
- 上書きを削除すると、ホームの既定が使われる（MUST）。

#### 実体の取得

- `Avatar-URL` は署名付きで短期間だけ有効にしなければならない（MUST）。
- アバターは全ユーザに公開する（MUST）。プライバシー段階は定義しない（MUST NOT）。
- 場は署名付き URL を発行してよい（MAY）。
- CDN が使えない場合、IQ で `Payload` として画像を返してよい（MAY）。

#### 更新の検出

- サーバは `Avatar-Updated` を更新しなければならない（MUST）。
- ホームからの通知は標準に含めてはならない（MUST NOT）。送り先が分からず、ファンアウトが肥大化するため。
- 代わりに、`PRESENCE` や `ROSTER` に `Avatar-Ref` を同梱してよい（MAY）。
- クライアントは `Avatar-Hash` が変わっていれば IQ で `Avatar` を取得する SHOULD。
- 場はホームのアバターを短期間キャッシュしてよい（MAY）。TTL は実装依存。

#### PRESENCE への同梱

```
PRESENCE
  From = alice@example.com
  Show = 0（online）
  <Avatar-Ref>
    Avatar-EID     = alice@example.com
    Avatar-Hash    = <32 オクテット>
    Avatar-Updated = 1700000000
```

- `PRESENCE` に `Avatar-Ref` を同梱してよい（MAY）。
- 受信側は `Avatar-Hash` が変わっていれば再取得する SHOULD。
- 同梱は任意。省略してもよい（MAY）。

#### キャッシュ

- クライアントは `Avatar-Hash` を鍵にキャッシュする SHOULD。
- `Avatar-Updated` が変わったら再取得しなければならない（MUST）。
- `Avatar-URL` が失効したら再取得しなければならない（MUST）。
- 場はホームから取得したアバターを短期間キャッシュしてよい（MAY）。
- 場はホームから取得したアバターを短期間キャッシュしてよい（MAY）。

## 12.5 メッセージの編集・削除

### 編集

```
MESSAGE
  Message-ID   C=1  （新しい ID）
  From         C=1
  Community-ID C=1
  Channel-ID   C=1
  Edit-Target  C=1  （編集対象の Message-ID）
  Body         C=1  （新しい本文）
  Timestamp    任意
```

- 編集は元の投稿者本人が行わなければならない（MUST）。MANAGE_CHANNEL を持つ者は他人のメッセージを編集してはならない（MUST NOT）。
- サーバは対象メッセージの Body を上書きし、`Edited` を現在時刻に更新しなければならない（MUST）。対象が存在しない、または権限がない場合は `not-found` / `not-allowed` を返す（MUST）。
- ACK の Reply-To-ID は編集メッセージの Message-ID でなければならない（MUST）。
- ライブ push は `Edit-Target` と `Edited` と新しい Body を含む MESSAGE を push しなければならない（MUST）。受信側は `Edit-Target` で対象を特定して更新する（MUST）。
- 履歴 sync は更新後の Body と `Edited` を返さなければならない（MUST）。
- 編集済みメッセージの `Edited` は last 勝ちとする（MUST）。
- 削除済みメッセージは編集してはならない（MUST NOT）。

### 削除

```
IQ set
  Message-ID      = <iq-...>
  IQ-Kind         = 2
  Delete-Target   = <message-id>
```

- 削除は元の投稿者本人、または MANAGE_CHANNEL を持つ者が行える（MUST）。
- サーバは対象メッセージを完全に削除しなければならない（MUST）。Message-ID は再利用防止のため記録し続ける（MUST）。
- 応答: IQ result（空）/ `not-found` / `not-allowed`。
- ライブ push は `Delete-Target` と `Deleted`（空）を含む MESSAGE を push しなければならない（MUST）。Body は含めてはならない（MUST NOT）。受信側は対象を除去する（MUST）。
- 履歴 sync は削除済みメッセージを返してはならない（MUST NOT）。
- 削除済みメッセージへの REACTION は拒否しなければならない（MUST）。
- 検索結果に削除済みメッセージを含めてはならない（MUST NOT）。

---

## 12.6 ファイル転送（pull モデル）

### メタデータ先行

- 大容量バイナリを伴う MESSAGE は `Payload` を空にし、メタデータのみを載せる（MUST）。
  - `File-Name`（`0x0115`、UTF-8）
  - `File-Size`（`0x0116`、8B）
  - `File-Hash`（`0x0117`、32B SHA-256）
  - `Content-Type`（`0x0006`）で MIME タイプ
- ライブ push でもメタデータのみ配送する（MUST）。実バイナリは送らない。
- 小容量（Max-Size 以内）のバイナリは `Payload` を直接載せてよい（MAY）。

### 要求ベースの取得

```
IQ get file-fetch
  Message-ID = <message-id>
→ 応答: IQ result, Transfer-ID = <16B>（新規発行）
  その後 CHUNK フレームでストリーム送信
```

- 取得には READ が必要（MUST）。権限喪失後は取得不可（MUST）。
- サーバは Transfer-ID を新規発行する（MUST）。MESSAGE には Transfer-ID を永続化しない（MUST NOT）。
- ホストはバイナリを正本として保存する（MUST）。送信者がオフラインでも取得可能。
- 保存期間・容量の上限は実装ポリシーで定義してよい（MAY）。期限切れは `not-found`。

### 区別

- テキスト: `Body` を使う。`Payload` を付けてはならない（MUST NOT）。
- バイナリ: `Payload` を使う。`Body` を付けてはならない（MUST NOT）。
- `Content-Type` で MIME を補助的に示す（MAY）。

---


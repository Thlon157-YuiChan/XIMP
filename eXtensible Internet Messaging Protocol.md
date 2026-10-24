# XIMP 仕様書（実装用草案）

eXtensible Internet Messaging Protocol
Version 1.0-draft
状態: 実装用草案。IANA ポートと一部の管理詳細は未確定。

本書は実装者が相互運用できる範囲だけを書く。認証方式の中身、パスフレーズのハッシュ、ロールの表示名はサーバ実装である。

## キーワード

本仕様は RFC 2119 と RFC 8174 のキーワードを使う。

- MUST / MUST NOT: 必須。守らなければ XIMP 準拠を名乗れない。
- SHOULD / SHOULD NOT: 推奨。守らない場合は理由を文書化する SHOULD。
- MAY: 任意。実装の自由。
- REQUIRED: MUST と同じ。
- SHALL: MUST と同じ。本仕様では使わない。
- SHOULD の例外: 実装が SHOULD を守らない場合、相互運用への影響を文書化する SHOULD。

キーワードは大文字で書いたときだけ RFC 2119 の意味を持つ。小文字の must / should / may は通常の英語として読む。

---

## 1. 範囲

XIMP はクライアントとサーバの間でテキストと小さなバイナリを交換するアプリケーションプロトコルである。

- 構成はクライアント—サーバでなければならない（MUST）。P2P は定義しない（MUST NOT）。
- 音声・映像そのものは運んではならない（MUST NOT）。チャネル join の応答で、別プロトコルの名前と接続先だけを返してよい（MAY）。
- エンドツーエンド暗号は本仕様では定義しない。実装してよい（MAY）。本文は TLS 区間でのみ守られる。
- 1 台のサーバ（ホスト）が複数のコミュニティを持ってよい（MAY）。
- サーバ間で運ぶのはユーザ認証のチケット確認だけでなければならない（MUST）。本文は転送してはならない（MUST NOT）。
- Bot は特別な存在ではなく、EID を持つ通常のクライアントとして扱わなければならない（MUST）。
- Webhook はプロトコルに定義してはならない（MUST NOT）。サーバ実装に委ねる。

## 2. 用語

| 用語 | 意味 |
|---|---|
| ホスト | TCP で接続する XIMP サーバ。証明書の名前を持つ。 |
| ホーム | `user@domain` の資格情報を持つサーバ。 |
| コミュニティ | ホスト上の場。不透明 ID で識別する。 |
| チャネル | コミュニティ内の会話の列。不透明 ID で識別する。 |
| EID | 利用者識別子。`user@domain`。 |
| 正本 | ホストが保存するメッセージ。クライアントの履歴はキャッシュ。 |
| チケット | ホームが発行する短命な身分証明。場への入場許可ではない。 |
| サーバ | 文脈によりホストまたはホームを指す。 |
| 場 | コミュニティを持つホスト。 |
| Bot | EID を持つ通常のクライアント。特別扱いしない。 |
| 名前空間 | Slash Command の帰属を示す識別子。 |
| アバター | EID に紐付く画像。ホームの既定とコミュニティの上書きがある。 |

## 3. バイト列

- 運び役は TCP 上の TLS 1.3 でなければならない（MUST）。HTTP / WebSocket / STARTTLS を使ってはならない（MUST NOT）。
- TLS 1.3 のみを使わなければならない（MUST）。TLS 1.2 以下は拒否しなければならない（MUST）。クライアント証明書を使ってはならない（MUST NOT）。ALPN は `ximp` でなければならない（MUST）。0-RTT を使ってはならない（MUST NOT）。セッション再開は PSK で許容してよい（MAY）。SNI は送ってよい（MAY）が、サーバは SNI をコミュニティ選択に使ってはならない（MUST NOT）。
- 既定ポートは `5777/tcp` とする SHOULD。実装は設定でポートを変更できなければならない（MUST）。IANA 登録までは 5777 を使う SHOULD。
- 多オクテット整数はすべてビッグエンディアンでなければならない（MUST）。
- 文字列は UTF-8 のみでなければならない（MUST）。BOM を付けてはならない（MUST NOT）。NUL 終端を付けてはならない（MUST NOT）。長さは親の Length で示す。
- 不正な UTF-8（過長符号化、途中切れ、孤立サロゲート）は `malformed` としなければならない（MUST）。
- EID、表示名、Community-Name、Channel-Name、Role-Name、Command-Namespace、Command-Name は比較の前に NFC を適用する MUST。Body、Status-Text、Error-Text は正規化してはならない（MUST NOT）。
- 実装は受信上限を持ち、HELLO の Max-Size で広告する SHOULD。受信上限を超える Length のメッセージは `too-large` で拒否しなければならない（MUST）。接続を閉じてよい（MAY）。Max-Size を送らない実装の既定は 1 MiB とする SHOULD。ワイヤ上の Length 最大は 16,777,215 だが、これをそのまま受け入れる実装は 16 MiB 超のバッファを確保してはならない（MUST NOT）。ストリーミングで読む SHOULD。

### 3.1 外枠

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+---------------+---------------+-------------------------------+
|    Version    |     Type      |            Length             |
+---------------+---------------+-------------------------------+
|                         Value (Length octets)                 |
+---------------------------------------------------------------+
```

| フィールド | 幅 | 意味 | 度合い |
|---|---|---|---|
| Version | 1 | 本仕様は `0x01`。`0x00` は予約。 | MUST |
| Type | 1 | メッセージ種類。 | MUST |
| Length | 3 | Value のオクテット数。ヘッダ 5 は含めない。 | MUST |
| Value | Length | 要素 TLV の連結。空でもよい。 | MAY |

- メッセージ全体は `5 + Length` でなければならない（MUST）。
- ストリーム上に隙間なく連結してよい（MAY）。
- 受信側は 5 オクテット読んでから Length 分を読まなければならない（MUST）。

### 3.2 要素 TLV

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|C|R|        Element Type       |            Length             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Value (Length octets)                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| フィールド | 幅 | 意味 | 度合い |
|---|---|---|---|
| C | 1 bit | 1 なら Critical。未知ならメッセージ全体を拒否。 | MUST |
| R | 1 bit | 予約。送信時 0。受信時は無視してよい。 | MUST / MAY |
| Element Type | 14 bit | 0 は予約。 | MUST |
| Length | 2 | この要素の Value の長さ。0..65535。ヘッダ 4 は含めない。 | MUST |
| Value | Length | 型依存。入れ子可。最大深さ 4（MUST）。 | MUST |

- C=1 の Type はワイヤ上 `0x8000 | type` になる（MUST）。例: Message-ID は `0x8001`。
- 未知要素の扱い:
  - C=0: Length だけ使って読み飛ばさなければならない（MUST）。
  - C=1: メッセージ全体を捨て、`ERROR unsupported-critical` を返さなければならない（MUST）。
- 要素 Length が親の残りを超える場合は `malformed` としなければならない（MUST）。接続を閉じてよい（MAY）。
- 同じ Type が複数ある場合の扱いは、要素ごとに次で定める（MUST）。
  - single: 複数あれば `malformed` としなければならない（MUST）。From、To、Message-ID、IQ-Kind、Auth-Status、Ticket、Home-Sig、Total-Size など。
  - list: 複数可（MAY）。順序を保たなければならない（MUST）。Capability、External-Offer、Role-ID、Channel-Allow、Channel-Deny、Mention、Reaction-Group、Avatar、Avatar-Ref など。
  - last: 最後が勝つ（MUST）。省略可のメタデータのみ。Timestamp、Show、Status-Text など。
- 要素レジストリの各行に single / list / last を明記しなければならない（MUST）。
- C=1 の single 要素が複数ある場合は `malformed` としなければならない（MUST）。Body は連結してはならない（MUST NOT）。

### 3.3 ホスト名の比較

ホスト名は DNS 名として比較する。比較の前に次を行わなければならない（MUST）。

1. ASCII 大文字を小文字に変換する。
2. 末尾のドットを 1 個だけ除去する。
3. IDN は A-label（Punycode）に変換する。U-label のまま送ってはならない（MUST NOT）。

Ticket-Aud、aud、Requester、証明書名、SRV の対象はすべてこの正規形で比較しなければならない（MUST）。正規形が空、または 255 オクテットを超えるものは `malformed` としなければならない（MUST）。

### 3.4 証明書

- X.509 v3 でなければならない（MUST）。
- SAN に DNS 名を入れなければならない（MUST）。CN を使ってはならない（MUST NOT）。
- ワイルドカードは RFC 9525 に従わなければならない（MUST）。
- チェーンはルート CA まで遡らなければならない（MUST）。
- 失効は OCSP または CRL で確認しなければならない（MUST）。OCSP stapling を使う SHOULD。
- 失効確認に失敗した場合、クライアントは接続を切らなければならない（MUST）。
- Certificate Transparency の SCT を検証してよい（MAY）。必須ではない。
- クライアント証明書を使ってはならない（MUST NOT）。mTLS を要求するサーバは XIMP 準拠ではない（MUST NOT）。

## 4. 識別子

```
EID = localpart "@" domain
```

- `@` はちょうど 1 個でなければならない（MUST）。スキーム、括弧、`/resource` を付けてはならない（MUST NOT）。
- localpart は 1..128 オクテットでなければならない（MUST）。NUL、空白、`@` を含んではならない（MUST NOT）。
- domain は 1..255 オクテットでなければならない（MUST）。ASCII LDH ラベルをドットで連結する。
- 全体の推奨最大は 256 オクテットとする SHOULD。
- ドメイン比較は ASCII の大小を無視しなければならない（MUST）。localpart はそのまま比較しなければならない（MUST）。
- 正規形はホームが決めて `From` に返さなければならない（MUST）。他ドメインの localpart を、ホームを通さず作ってはならない（MUST NOT）。
- Community-ID、Channel-ID、Role-ID は 16 オクテットでなければならない（MUST）。サーバが採番しなければならない（MUST）。再利用してはならない（MUST NOT）。表示名でこれらを指定してはならない（MUST NOT）。

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

## 6. 要素レジストリ

C の既定は、その型を必須とするメッセージでの推奨（SHOULD）。未知の C=0 は常に飛ばさなければならない（MUST）。

| 値 | 名前 | 多重度 | Value | 度合い |
|---|---|---|---|---|
| `0x0001` | Message-ID | single | 1..64 オクテット、または 16 オクテット UUID。 | MUST（対象メッセージで） |
| `0x0002` | From | single | EID の UTF-8。 | MUST（対象メッセージで） |
| `0x0003` | To | single | EID の UTF-8。1対1だけ。 | MUST（1対1で） |
| `0x0004` | Timestamp | last | Unix ミリ秒。8 オクテット符号なし。 | MAY |
| `0x0005` | Body | single | UTF-8 本文。 | MUST（Body を使うとき） |
| `0x0006` | Content-Type | single | UTF-8。RFC 9110 の media-type。charset は utf-8 のみ。 | MAY |
| `0x0007` | Payload | single | 生バイト。 | MUST（Payload を使うとき） |
| `0x0008` | Thread-ID | single | UTF-8。 | MAY |
| `0x0009` | Reply-To-ID | single | 返信元、または履歴なら親 IQ の Message-ID。ライブでは付けない。 | MUST（履歴で） / MUST NOT（ライブで） |
| `0x000A` | IQ-Kind | single | 1 オクテット。1=get 2=set 3=result 4=error。 | MUST（IQ で） |
| `0x000B` | Error-Code | single | 2 オクテット。 | MUST（ERROR で） |
| `0x000C` | Error-Text | single | UTF-8。機械判定に使わない。 | MAY |
| `0x000D` | Capability | list | UTF-8 トークン。複数可。 | MUST（HELLO で 1 個以上） |
| `0x000E` | Max-Size | single | 3 オクテット。受信できる最大 Length。HELLO に C=1 で付ける。省略時 1 MiB。 | SHOULD（HELLO で） |
| `0x000F` | Software | single | UTF-8。 | MAY |
| `0x0010` | Mechanism | single | UTF-8 の方式名。 | MUST（AUTH で） |
| `0x0011` | Auth-Data | single | 方式依存。ログに出さない。 | MUST（AUTH で） |
| `0x0012` | Auth-Status | single | 1 オクテット。0=続き 1=確定 2=失敗。 | MUST（AUTH_RESULT で） |
| `0x0020` | Show | last | 1 オクテット。0=online 1=away 2=dnd 3=xa 4=offline。 | MAY |
| `0x0021` | Status-Text | last | UTF-8。 | MAY |
| `0x0040` | Transfer-ID | single | 16 オクテット。 | MUST（CHUNK で） |
| `0x0041` | Offset | single | 8 オクテット符号なし。バイト単位。 | MUST（CHUNK で） |
| `0x0042` | Total-Size | single | 8 オクテット符号なし。 | MUST（CHUNK の先頭で） |
| `0x0050` | Sync-Cursor | single | 不透明。クライアントは書き換えない。 | MAY |
| `0x0051` | Sync-Limit | single | 2 オクテット。0 禁止。省略時 16。 | MAY |
| `0x0052` | Sync-Scope | single | 1 オクテット。2=チャネル。1=inbox。 | MUST（sync-get で） |
| `0x0053` | More | single | 1 オクテット。1=続きあり 0=終わり。 | MUST（sync-get の応答で） |
| `0x0054` | Stored | single | 空。保管庫から出した印。C=0。 | MAY |
| `0x0060` | Channel-ID | single | 16 オクテット。 | MUST（チャネル操作で） |
| `0x0061` | Channel-Topic | single | UTF-8。 | MAY |
| `0x0063` | Channel-Kind | single | 1 オクテット。0=text 1=external。 | MUST（channel-create で） |
| `0x0064` | External-Offer | list | 1 方式分の入れ子。複数可。 | MUST（外部チャネルの join で 1 個以上） |
| `0x0065` | External-Proto | single | Offer の中。UTF-8。登録しない。 | MUST（Offer で） |
| `0x0066` | External-Target | single | Offer の中。URI または host:port。 | MUST（Offer で） |
| `0x0067` | External-Token | single | Offer の中。任意。ログに出さない。 | MAY |
| `0x0070` | Ticket-Aud | single | 発行要求のホスト名。UTF-8。A-label 正規形。 | MUST（チケット発行で） |
| `0x0071` | Ticket | single | 第 9 節のバイト列。 | MUST（チケットで） |
| `0x0072` | Ticket-Exp | single | Unix 秒。8 オクテット。 | MUST（チケット応答で） |
| `0x0073` | Requester | single | S2S 確認元のホスト名。UTF-8。A-label 正規形。 | MUST（S2S_AUTH で） |
| `0x0074` | Home-Sig | single | Ed25519 署名 64 オクテット。S2S の AUTH_RESULT に付ける。 | MUST（成功時の AUTH_RESULT で） |
| `0x0075` | Act | single | 代理の EID。UTF-8。委任用チケットに含める。 | MAY（委任時） |
| `0x0076` | Scope | single | 入れ子。委任の範囲。 | MAY（委任時） |
| `0x0077` | Scope-Community | list | 16 オクテット。 | MAY |
| `0x0078` | Scope-Channel | list | 16 オクテット。 | MAY |
| `0x0079` | Scope-Mask | single | 4 オクテット。 | MAY |
| `0x007A` | Scope-Exp | single | Unix 秒 8 オクテット。 | MAY |
| `0x0080` | Community-ID | single | 16 オクテット。 | MUST（コミュニティ操作で） |
| `0x0081` | Community-Name | single | UTF-8 表示名。 | MUST（community-create で） |
| `0x0082` | Admit-Mode | single | 1 オクテット。 | MUST（community-list で） |
| `0x0083` | Admit-Data | single | 入場用バイト。ログに出さない。 | MAY（入場時） |
| `0x0090` | Role-ID | single | 16 オクテット。 | MUST（ロール操作で） |
| `0x0091` | Role-Name | single | UTF-8 表示名。 | MAY |
| `0x0092` | Role-Mask | single | 4 オクテット。 | MUST（role-create / role-set-mask で） |
| `0x0093` | Channel-Allow | list | 4 オクテット。 | MAY（ACL で） |
| `0x0094` | Channel-Deny | list | 4 オクテット。 | MAY（ACL で） |
| `0x00A0` | Reaction-Target | single | 対象の Message-ID。 | MUST（REACTION で） |
| `0x00A1` | Reaction-Key | single | 不透明なバイト列。1..64 オクテット。 | MUST（add / remove で） |
| `0x00A2` | Reaction-Op | single | 1 オクテット。0=add 1=remove 2=clear。 | MUST（REACTION で） |
| `0x00A3` | Reaction-Count | single | 2 オクテット符号なし。 | MAY（集約時） |
| `0x00A4` | Reaction-Group | list | 集約時の 1 グループ。 | MAY（集約時） |
| `0x00A5` | Sync-Options | single | 4 オクテットのビットマスク。bit 0 = include-reactions。bit 1 = include-mentions。 | MAY |
| `0x00B0` | Mention | list | 1 メンション分。入れ子。 | MAY |
| `0x00B1` | Mention-Kind | single | 1 オクテット。0=user 1=role 2=everyone 3=here。 | MUST（Mention で） |
| `0x00B2` | Mention-Target | single | EID または Role-ID。everyone / here では省略。 | MUST（user / role で） |
| `0x00B3` | Mention-Offset | single | 4 オクテット。Body 内のバイトオフセット。 | MAY |
| `0x00B4` | Mention-Length | single | 4 オクテット。Body 内のバイト長。 | MAY |
| `0x00B5` | Mention-Text | single | UTF-8。元の表記。 | MAY |
| `0x00B6` | Notify-Level | single | 1 オクテット。0=all 1=mentions 2=none。 | MAY |
| `0x00C0` | Search-Scope | single | 1 オクテット。1=コミュニティ 2=チャネル。 | MUST（検索で） |
| `0x00C1` | Search-Query | single | UTF-8。クエリ言語はサーバ実装。 | MUST（検索で） |
| `0x00C2` | Search-Limit | single | 2 オクテット。省略時 32。0 禁止。 | MAY |
| `0x00C3` | Search-Cursor | single | 不透明。 | MAY |
| `0x00C4` | Search-From | single | EID。任意。 | MAY |
| `0x00C5` | Search-After | single | Unix ミリ秒 8 オクテット。任意。 | MAY |
| `0x00C6` | Search-Before | single | Unix ミリ秒 8 オクテット。任意。 | MAY |
| `0x00C7` | Search-Result | list | Message-ID + Channel-ID + From + Timestamp + Snippet。 | MUST（検索応答で） |
| `0x00C8` | Snippet | single | UTF-8。抜粋。最大 256 オクテット。 | MAY |
| `0x00D0` | Bot | single | 空。Bot の発言に付ける。 | MAY |
| `0x00D1` | Event-Kind | single | 4 オクテットのビットマスク。 | MUST（Event-Subscribe で） |
| `0x00D2` | Event-Subscribe | single | 入れ子。Community-ID + Channel-ID + Event-Kind。 | MUST（購読で） |
| `0x00D3` | Subscription-ID | single | 16 オクテット。不透明。 | MUST（購読応答で） |
| `0x00E0` | Command-Name | single | UTF-8。1..32 オクテット。`[a-z0-9_-]` を SHOULD。 | MUST（Command で） |
| `0x00E1` | Command-Desc | single | UTF-8。 | MAY |
| `0x00E2` | Command-Args | single | UTF-8。最大 4096 オクテット。 | MAY |
| `0x00E3` | Command-Result | single | UTF-8 または要素 TLV。 | MAY |
| `0x00E4` | Command-Register | single | 入れ子。Community-ID + Command-Namespace + Command-Name + Command-Desc。 | MUST（登録で） |
| `0x00E5` | Command-Invoke | single | 入れ子。Community-ID + Channel-ID + Command-Namespace + Command-Name + Command-Args。 | MUST（実行で） |
| `0x00E6` | Command-Namespace | single | UTF-8。1..32 オクテット。`[a-z0-9_-]` を SHOULD。 | MUST（登録・実行で） |
| `0x00E7` | Command-Scope | single | 1 オクテット。0=コミュニティ 1=チャネル。 | MAY |
| `0x00F0` | Avatar | list | 1 アバター分。入れ子。 | MAY |
| `0x00F1` | Avatar-EID | single | 対象の EID。UTF-8。 | MUST（Avatar で） |
| `0x00F2` | Avatar-URL | single | UTF-8。取得先。HTTPS を SHOULD。 | MUST（Avatar で） |
| `0x00F3` | Avatar-MIME | single | UTF-8。MIME タイプ。 | MAY |
| `0x00F4` | Avatar-Width | single | 2 オクテット。 | MAY |
| `0x00F5` | Avatar-Height | single | 2 オクテット。 | MAY |
| `0x00F6` | Avatar-Hash | single | 32 オクテット。SHA-256。 | MAY |
| `0x00F7` | Avatar-Updated | single | Unix 秒 8 オクテット。 | MAY |
| `0x00F8` | RESERVED | 本版では送信禁止（MUST NOT）。アバター公開範囲は撤廃。 |
| `0x00F9` | Avatar-Upload | single | 入れ子。Avatar-MIME + Payload。 | MUST（登録で） |
| `0x00FA` | Avatar-Delete | single | 空。アバターを削除する。 | MUST（削除で） |
| `0x00FB` | Avatar-Ref | list | 軽量版。Avatar-EID + Avatar-Hash + Avatar-Updated。 | MAY |
| `0x00FC` | Avatar-Known-Hash | single | 32 オクテット。クライアントが既に持っているハッシュ。 | MAY |
| `0x0100` | Roster-Item | list | 1 件分。入れ子。1.1 で定義。 | MAY |
| `0x0101` | Roster-EID | single | EID。UTF-8。 | MUST（Roster-Item で） |
| `0x0102` | Roster-Sub | single | 1 オクテット。0=none 1=to 2=from 3=both。 | MUST（Roster-Item で） |
| `0x0103` | Roster-Name | single | UTF-8 ニックネーム。 | MAY |
| `0x0104` | Roster-Group | list | UTF-8 グループ名。 | MAY |
| `0x0105` | Emoji-URL | single | UTF-8。取得先。 | MUST（emoji-resolve で） |
| `0x0106` | Emoji-ID | single | UTF-8。`custom:` の後の ID。 | MUST（emoji-resolve で） |
| `0x0107` | Avatar-Bulk | single | 入れ子。Avatar-EID の繰り返し。 | MUST（一括取得で） |
| `0x0108` | Dm-Policy | single | 1 オクテット。0=open 1=contacts-only 2=closed。 | MAY |
| `0x0109` | Blocked-EID | single | EID。UTF-8。 | MAY |
| `0x010A` | Block-Action | single | 1 オクテット。0=add 1=remove。 | MAY |
| `0x010B` | Ban-Action | single | 1 オクテット。0=ban 1=unban。 | MUST（community-ban で） |
| `0x010C` | Ban-Reason | single | UTF-8。運用者向け。被 ban 者に返さない。 | MAY |
| `0x010D` | Host-Mask | single | 4 オクテット。ホストレベル権限ビット。 | MUST（host-role で） |
| `0x010E` | Host-Role-Item | list | 1 件分。Host-Role-EID + Host-Mask。入れ子。 | MAY |
| `0x010F` | Host-Role-EID | single | EID。UTF-8。 | MUST（Host-Role-Item で） |
| `0x0110` | Host-Owner-Target | single | EID。UTF-8。譲渡先。 | MUST（host-owner-transfer で） |
| `0x0111` | Edit-Target | single | 対象の Message-ID。 | MUST（編集で） |
| `0x0112` | Edited | single | Unix ミリ秒 8 オクテット。最終編集時刻。 | MUST（編集済みで） |
| `0x0113` | Deleted | single | 空。削除済みの印。 | MUST（削除通知で） |
| `0x0114` | Delete-Target | single | 対象の Message-ID。 | MUST（削除で） |
| `0x3F00` | Vendor | list | 先頭 4 オクテットが IANA PEN。続きはベンダ要素。C ビットはベンダが決める。未知の Vendor 要素は C=0 として飛ばせる。入れ子可。深さは 3.2 の制限に従う。 | MAY |

### 6.1 Avatar の構造

```
Avatar
  Avatar-EID      C=1   MUST
  Avatar-URL      C=1   MUST
  Avatar-MIME     C=0   MAY
  Avatar-Width    C=0   MAY
  Avatar-Height   C=0   MAY
  Avatar-Hash     C=0   MAY
  Avatar-Updated  C=0   MAY
```

### 6.2 Avatar-Ref の構造

```
Avatar-Ref
  Avatar-EID      C=1   MUST
  Avatar-Hash     C=1   MUST
  Avatar-Updated  C=0   MAY
```

- `Avatar-Ref` は `Avatar` の軽量版。`PRESENCE` や `ROSTER` に同梱してよい（MAY）。
- `Avatar-URL` や `Avatar-MIME` は含めない。クライアントは `Avatar-Hash` が変わっていれば IQ で `Avatar` を取得する SHOULD。

## 7. エラーコード

| 値 | 名前 | 接続 | 度合い |
|---|---|---|---|
| `0x0001` | malformed | 閉じてよい。 | MAY |
| `0x0002` | unsupported-version | 閉じてよい。 | MAY |
| `0x0003` | unsupported-type | 維持。 | MUST |
| `0x0004` | unsupported-critical | 維持。 | MUST |
| `0x0005` | too-large | 閉じてよい。 | MAY |
| `0x0006` | unauthorized | 維持。未 AUTH で業務を送った。 | MUST |
| `0x0007` | auth-failed | 維持。身分の失敗。正誤の詳細を返さない。 | MUST |
| `0x0008` | not-allowed | 維持。入場または権限の拒否。 | MUST |
| `0x0009` | not-found | 維持。 | MUST |
| `0x000A` | conflict | 維持。名前の衝突。 | MUST |
| `0x000B` | rate-limited | 維持。 | MUST |
| `0x000C` | mailbox-full | 維持。 | MUST |
| `0x00FF` | internal | 維持または切断。 | MAY |

- ERROR は可能なら元の Message-ID をコピーする SHOULD。
- Error-Text に秘密、チケット、Admit-Data を入れてはならない（MUST NOT）。
- IQ のエラーは IQ-Kind=4 で返さなければならない（MUST）。
- ERROR メッセージは外枠レベルの失敗に使う SHOULD。
- `not-found` はカーソル無効とリソース不在の両方に使ってよい（MAY）。Error-Text で区別してよい（MAY）。

## 8. セッション状態

```
未 AUTH
  許す: HELLO AUTH ERROR PING PONG BYE（MUST）
身分のみ
  上に加え、community-list と community-join（MUST）
メンバー
  そのコミュニティについて channel-* MESSAGE sync-get（MUST）
```

- AUTH_RESULT の Status=1 は「この TCP の身分は From」であり、コミュニティのメンバーではない（MUST）。
- キックはメンバー表から外し、そのコミュニティの購読を外さなければならない（MUST）。必要なら BYE を送ってよい（MAY）。ホームの資格情報は消えない（MUST NOT）。

## 9. 認証

AUTH はユーザ認証だけを行わなければならない（MUST）。方式は資格情報を持つホームの実装が決める。コミュニティはパスワードも OTP も検証してはならない（MUST NOT）。入場は第 10 節。

### 9.1 ホームでの身分

クライアントはホームへ別 TCP+TLS を張らなければならない（MUST）。

```
C → Home  HELLO
Home → C  HELLO  Capability: mech:<name> を方式ごとに
C → Home  AUTH  Mechanism + Auth-Data
Home → C  AUTH_RESULT  Auth-Status + From
```

- AUTH は同じ TCP 上で繰り返さなければならない（MUST）。新しい接続を張ってはならない（MUST NOT）。
- Status=0 の AUTH_RESULT を受けたら、クライアントは同じ Mechanism で Auth-Data だけ変えて AUTH を再送しなければならない（MUST）。
- Mechanism を途中で変えてはならない（MUST NOT）。変えたら `auth-failed` としなければならない（MUST）。
- Status=0 のあいだは身分未確定である（MUST）。
- ホームが広告してよい Mechanism は `mech:` で始まる UTF-8 トークンでなければならない（MUST）。場が受けてよいのは `home-ticket` だけである（MUST）。
- `mech:` の後の名前空間はホームのドメインで修飾する SHOULD。例: `mech:example.com/password`。
- この接続で `Mechanism=home-ticket` を受けてはならない（MUST NOT）。

### 9.2 チケット発行

身分確定後のホーム接続で IQ しなければならない（MUST）。

```
C → Home  IQ get
            Ticket-Aud = 入りたいホストの DNS 名（A-label 正規形）
Home → C  IQ result
            Ticket
            Ticket-Exp
```

- 未 AUTH なら `unauthorized` としなければならない（MUST）。
- `aud` の正規形が空、または 255 オクテット超なら拒否しなければならない（MUST）。汎用チケットを発行してはならない（MUST NOT）。
- TTL の既定は 5 分とする SHOULD。上限は 15 分としなければならない（MUST）。無期限は禁止する（MUST NOT）。
- ホームは `jti + aud + sub` を、exp + 60 秒まで保持しなければならない（MUST）。保持期間を過ぎたものは破棄してよい（MAY）。保持は永続化しなければならない（MUST）。再起動後も exp + 60 秒まで保持しなければならない（MUST）。

チケットのバイト列:

```
Version   1     0x01
KID       1     ホームの鍵番号
Sig       64    Ed25519
BodyLen   2
Body      BodyLen オクテットの要素 TLV
```

Body は次を C=1 で含まなければならない（MUST）。

| 要素名 | 中身 | 度合い |
|---|---|---|
| sub | 正規 EID | MUST |
| aud | ホスト名（A-label 正規形） | MUST |
| iat | Unix 秒 8 オクテット | MUST |
| exp | Unix 秒 8 オクテット | MUST |
| jti | 16 オクテット乱数 | MUST |

任意で次を含んでよい（MAY）。

| 要素名 | 中身 |
|---|---|
| act | 代理の EID。委任用チケット。C=0。 |
| scope | 委任の範囲。C=0。 |

- 署名対象は `Version || KID || BodyLen || Body` でなければならない（MUST）。コミュニティはこの中身を解釈しなくてよい（MAY）。
- KID は 1 オクテットでなければならない（MUST）。0 は予約する（MUST）。ホームは同時に最大 2 鍵を現用としてよい（MAY）。新しい鍵に切り替えるとき、旧鍵は旧鍵で発行済みのチケットの exp + 60 秒まで検証に使わなければならない（MUST）。KID 0 のチケットは `malformed` としなければならない（MUST）。

### 9.2.1 委任用チケット

- `act` は代理の EID でなければならない（MUST）。`sub` と異ならなければならない（MUST）。
- `scope` は委任の範囲である（MAY）。
- ホームは `sub` が `act` に委任する権限を持っていることを検証しなければならない（MUST）。
- `Scope-Mask` は `sub` の実効マスクを超えてはならない（MUST NOT）。
- `Scope-Community` は `sub` が属するコミュニティでなければならない（MUST）。
- `Scope-Channel` は `Scope-Community` のチャネルでなければならない（MUST）。
- 検証のポリシーはホーム実装が決めてよい（MAY）。

### 9.3 場での提示

```
C → Host  AUTH
            Mechanism = "home-ticket"
            From = EID
            Auth-Data = Ticket
Host → Home  S2S_AUTH
            Mechanism = "home-ticket"
            From = Subject EID
            Auth-Data = Ticket
            Requester = 自分のホスト名
Home → Host  AUTH_RESULT
            Auth-Status
            From（成功時）
            Home-Sig（0x0074、成功時は C=1）
Host → C     AUTH_RESULT を中継
```

- 場の HELLO がユーザ向けに広告してよい Mechanism は `home-ticket` だけでなければならない（MUST）。
- `password` 等が場に来たら転送せず `auth-failed` としなければならない（MUST）。
- 場は S2S の TLS で、ホームの証明書名がホームのドメイン（SNI の名前）と一致することを確認しなければならない（MUST）。一致しなければ S2S_AUTH を送ってはならない（MUST NOT）。SNI もホームの名前でなければならない（MUST）。aud は対象ホストの名であり、ホーム証明書名の比較対象ではない（MUST NOT）。

### 9.4 ホームの確認

次の順で確認しなければならない（MUST）。どれが失敗しても Status=2 としなければならない（MUST）。理由を分けてはならない（MUST NOT）。

1. 相手証明書の名前と Requester が一致する。
2. Requester とチケットの aud が、3.3 の正規化のあと一致する。
3. KID に対応する鍵が現用または移行中で、署名が正しい。未知の KID は Status=2。
4. 現在時刻が iat から exp の中（ずれは 30 秒まで）。
5. sub と Subject が正規化後に一致する。
6. jti が未使用。確認は原子的に行い、成功したら使用済みにする。同じ jti に対する並行確認は、1 つだけ成功し他は失敗する。

委任用チケットの場合、次も確認しなければならない（MUST）。

7. `act` がある場合、`act` が妥当な EID で、`sub` と異なる。
8. `scope` がある場合、`Scope-Mask` が `sub` の実効マスクを超えない。
9. `scope` の `Scope-Community` と `Scope-Channel` が `sub` の権限内である。

- ホームは同じ (jti, aud) の確認を 1 回だけ成功させなければならない（MUST）。同一チケットの複数回提示は 2 回目以降 Status=2 としなければならない（MUST）。

ホームは AUTH_RESULT を返すとき、次に署名しなければならない（MUST）。

```
"XIMP-S2S-AUTH-RESULT" || 0x00 ||
Auth-Status || 0x00 ||
Ticket の Body 全体 || 0x00 ||
Requester（正規形）
```

- 署名鍵は KID の現用鍵でなければならない（MUST）。
- 場は Home-Sig を検証してからクライアントへ中継しなければならない（MUST）。検証に失敗したら `auth-failed` としなければならない（MUST）。
- ホームの Ed25519 公開鍵の配布方法を実装は文書化しなければならない（MUST）。事前設定または信頼できる DNS/設定で取得しなければならない（MUST）。取得できない場合 S2S_AUTH を送ってはならない（MUST NOT）。
- 場はチケットをディスクに残してはならない（MUST NOT）。ログ、ERROR、ACK に出してはならない（MUST NOT）。
- 確認が終わるまで MESSAGE を受けてはならない（MUST NOT）。
- ホームに届かない場合、場はチケットを独自に信じてはならない（MUST NOT）。

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

## 13. 権限

判定に使うのは 32 ビットマスクだけでなければならない（MUST）。ロールはマスクを人に付ける管理用の名前でなければならない（MUST）。

| bit | 名前 | 意味 |
|---|---|---|
| 0 | READ | sync-get とライブ受信。 |
| 1 | WRITE | MESSAGE。 |
| 2 | ATTACH | CHUNK。 |
| 3 | JOIN_CHANNEL | channel-join。 |
| 4 | MANAGE_CHANNEL | チャネルの作成・改名・削除・ACL。 |
| 5 | INVITE | 招待トークンの発行。 |
| 6 | KICK | メンバー除名。 |
| 7 | MANAGE_ROLE | ロール作成と、自分が持つビット内の付与。 |
| 8 | MANAGE_ADMIT | 入場規則の変更。 |
| 9 | EXTERNAL_JOIN | Kind=external のチャネルに入る。 |
| 10 | MENTION_EVERYONE | `@everyone` と `@here` を使える。 |
| 11 | COMMAND_REGISTER | Slash Command を登録できる。 |
| 12 | EVENT_SUBSCRIBE | イベント購読を登録できる。 |
| 13–30 | 未割当 | 未知なら無視。 |
| 31 | OWNER | 全部を含む。OR では作れない。最後の 1 人は外せない。 |

- 作成時の owner ロールは bit 31 でなければならない（MUST）。
- join 時の member ロールは bit 0, 1, 3 でなければならない（MUST）。
- `base = その人のロールマスクの OR` でなければならない（MUST）。
- `実効 = (base | channel_allow) & ~channel_deny` でなければならない（MUST）。
- `許可 = (実効 & 必要なビット) != 0` でなければならない（MUST）。
- チャネル ACL はロール単位でなければならない（MUST）。
- `(community_id, channel_id, role_id) → allow, deny` でなければならない（MUST）。
- `allow = その人が持つロールの、このチャネルの allow の OR` でなければならない（MUST）。
- `deny = 同様に deny の OR` でなければならない（MUST）。
- ACL 行が無いロールは上書きしてはならない（MUST NOT）。
- deny が allow より強くなければならない（MUST）。
- ACL が全く無いチャネルは base だけでなければならない（MUST）。
- 人単位 ACL は定義してはならない（MUST NOT）。
- 個人指定は、その人用ロールを作って ACL に足さなければならない（MUST）。
- bit 31 は role mask と channel allow のどちらにも書いてよい（MAY）。channel deny には書いてはならない（MUST NOT）。書いた場合は `malformed` としなければならない（MUST）。
- base の計算で bit 31 を持つロールが複数あっても、結果は bit 31 が立つだけでなければならない（MUST）。
- MANAGE_ROLE でも、自分の実効マスクに無いビットを含むロールは作成・付与してはならない（MUST NOT）。
- OWNER 以外は bit 31 を立ててはならない（MUST NOT）。
- MANAGE_CHANNEL でも、自分が持たないビットを channel allow に書いてはならない（MUST NOT）。
- 判定は list、join、sync、ライブ、CHUNK、投稿の直前に同じ関数で行わなければならない（MUST）。
- ロール変更、ACL 変更、除名で接続のキャッシュを捨て、購読を外さなければならない（MUST）。
- 同じホスト上の全接続に即時反映しなければならない（MUST）。
- 他ホストの接続へは、次にその接続が IQ を送った時点で反映しなければならない（MUST）。
- ライブ配信時は配信直前に実効権限を再評価しなければならない（MUST）。取消済みの相手へ配信してはならない（MUST NOT）。
- 進行中の CHUNK は完了時チェックで拒否されうる（MAY）。
- 進行中の sync-get は次のページから新しい権限で返さなければならない（MUST）。
- リアクションの add / remove には READ を必須とし、WRITE を SHOULD とする。
- clear（自分の分を全部消す）には READ のみを要求しなければならない（MUST）。
- 委任用チケットの場合、実効マスクは次のように計算しなければならない（MUST）:

```
base = sub のロールマスクの OR
scope_mask = scope の Scope-Mask（あれば）
実効 = (base | channel_allow) & ~channel_deny
実効 = 実効 & scope_mask
```

- `scope_mask` がない場合、`sub` の実効マスクをそのまま使わなければならない（MUST）。
- `scope_mask` がある場合、それを上限としなければならない（MUST）。
- `Scope-Community` がある場合、そのコミュニティだけを対象としなければならない（MUST）。
- `Scope-Channel` がある場合、そのチャネルだけを対象としなければならない（MUST）。
- 対象外の操作は `not-allowed` としなければならない（MUST）。

```
IQ role-list / role-create / role-set-mask
IQ member-roles          Community-ID, To, Role-ID 複数
IQ channel-acl-set       Community-ID, Channel-ID, Role-ID, Allow, Deny
IQ channel-acl-list
```

- 名前で権限を判定してはならない（MUST NOT）。

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

## 15. サーバ間

- 本文は転送してはならない（MUST NOT）。
- S2S は第 9.3 節のチケット確認だけでなければならない（MUST）。
- Presence、ROSTER、アバターの cross-host 取得はクライアントが各ホームへ直接 pull する（MUST）。S2S で転送してはならない（MUST NOT）。

```
SRV _ximp._tcp.<domain> → host:port
無ければ A/AAAA <domain> の 5777
```

- SRV を引かなければならない（MUST）。
- 無ければ A/AAAA の 5777 を使わなければならない（MUST）。
- SRV の優先度・重みは RFC 2782 に従わなければならない（MUST）。

## 16. 実装が持たないもの

次に列挙するものは、本仕様では定義してはならない（MUST NOT）。実装が独自に提供するのは自由である（MAY）。

- SNI によるコミュニティ選択
- ログイン時の履歴自動配送
- コミュニティによるパスワードや OTP の中継
- AUTH 成功 = メンバー
- 表示名を宛先にすること
- サーバ間の MESSAGE
- ロール階層、人単位 deny
- エンドツーエンド暗号
- 絵文字の解釈（正規化、バリエーション、肌色）
- 絵文字リスト
- カスタム絵文字の形式（`custom:` の意味だけ標準、ID の解決はコミュニティ）
- リアクションの UI（表示順、集約、アニメーション）
- リアクションの集約上限
- リアクションの Actor 開示範囲
- 通知の方法（音、バッジ、プッシュ、メール）
- メンションのハイライト、色、アイコン、並び順
- ミュート、集中モード、夜間モード
- メンションの表示形式
- メンションのオートコンプリート
- 検索のクエリ言語、ランキング、インデックス、全文検索エンジン、添付の中身
- Bot の実行環境（ホスティング、言語、フレームワーク）
- Bot の登録 UI / 管理画面
- Bot のストア / マーケットプレイス
- Bot の課金
- Bot のレート制限の具体的な値
- Slash Command のオートコンプリート
- Bot の検証（公式 Bot バッジなど）
- Webhook（エンドポイント、認証、署名、再試行、冪等性、レート制限、SSRF 対策）
- 監査ログの形式、保存、取得、保持期間、改ざん防止
- 証明書の取得、更新、失効の運用
- ホームからのアバター更新通知（送り先が分からず、ファンアウトが肥大化するため）
- Pub/Sub によるアバター更新通知
- アバターのキャッシュ TTL の具体的な値
- アバター通知の頻度
- アバターの画像形式（PNG、JPEG、WebP、AVIF、SVG）
- アバターのリサイズ、圧縮、トリミング
- アバターの保存場所（サーバ、外部 URL、CDN）
- CDN の選択
- デフォルトアバター
- アバターのアップロード UI
- 管理者による他人のアバター変更（必要なら Vendor 要素で拡張する）

異なるホスト間の 1対1 は、§18.9 の cross-host DM（受信者ホームへの直接投稿）で定義する。それ以外の S2S による本文転送は禁止する（MUST NOT）。EID が他ドメインのユーザと会話するには、共通のコミュニティとチャネル、または cross-host DM を使う（MUST）。

## 17. 最小実装チェック

次に列挙する項目は、XIMP 準拠を名乗るための最小要件でなければならない（MUST）。

- 5 オクテット読んで Length 分を読む。整数はビッグエンディアン。
- C=0 未知は飛ばし、C=1 未知は拒否する。
- ホームで方式を照合し、場では `home-ticket` だけ受けて S2S 確認する。
- Requester、aud、証明書名、署名、期限、jti を確認する。
- 委任用チケットの場合、`act` と `scope` を確認する。
- join 成功まで MESSAGE を受けない。
- 履歴は sync-get だけ。新しい順。Reply-To-ID で履歴とライブを分ける。
- 権限はマスクの OR とチャネル deny。配信前にもう一度見る。
- Auth-Data、Ticket、Admit-Data をログに出さない。
- 証明書は X.509 v3、SAN、RFC 9525。ALPN は `ximp`。
- リアクションは READ 必須、WRITE SHOULD。履歴は include-reactions のときだけ。
- メンションは正本に保存。通知の重複排除はクライアント。
- 検索はサーバ側を前提。READ があるチャネルだけ返す。
- Slash Command は完全修飾のみ。予約名前空間を守る。
- アバターは `Avatar` と `Avatar-Ref` を扱う。`Avatar-Ref` は `PRESENCE` に同梱してよい。
- アバターは全公開（MUST）。`Avatar-Privacy` は撤廃。
- `Avatar-URL` は署名付きで短期間だけ有効にしなければならない（MUST）。
- `Avatar-Known-Hash` を送ると、変化がなければ `Avatar` を省略できる。
- ホームからのアバター更新通知はしない。`Avatar-Updated` と `Avatar-Hash` で更新を検出する。
- 場はホームのアバターを短期間キャッシュしてよい。

---

## 18. 1.1 で定義する項目

### 18.1 ROSTER（ホーム側の連絡先）

ROSTER はホームが保持する連絡先リストである（MUST）。場は ROSTER を保持してはならない（MUST NOT）。在席情報はクライアントがホームへ pull する（MUST）。ホームは在席を保存してはならない（MUST NOT）。

ROSTER は相互フレンドモデルとする（MUST）。`Roster-Sub=both` のみがフレンドである（MUST）。`to`/`from` は申請・承認の保留状態である（MUST）。
- Frame Type `0x21` ROSTER は、連絡先の追加・削除・購読状態の変化をホームがプッシュ通知するときに使う（MUST）。

Roster-Item（`0x0100`、list）の構造:

```
Roster-Item
  Roster-EID     (0x0101) C=1  MUST  EID
  Roster-Sub     (0x0102) C=1  MUST  0=none 1=to 2=from 3=both
  Roster-Name    (0x0103) C=0  MAY   UTF-8 ニックネーム
  Roster-Group   (0x0104) C=0 list MAY   UTF-8 グループ名
  Show           (0x0020) C=0  MAY   最終在席
  Status-Text    (0x0021) C=0  MAY
  Avatar-Ref     (0x00FB) C=0  MAY
```

IQ（ホーム接続）:

```
IQ get  roster-list      （カーソル、Limit 既定 64、上限 256）
IQ set  roster-add        Roster-EID, Roster-Group?
IQ set  roster-remove     Roster-EID
IQ set  roster-update     Roster-EID, Roster-Name?, Roster-Group?
IQ get  presence-get      Presence-Target × N（同じホームの連絡先のみ応答）
```

- `roster-add` は `to` を立て、相手が同じホームなら相手に `from` を立てる（MUST）。相手が別ホームの場合、購読状態は `to` のままとし、在席は取得できない（MUST）。
- `Roster-Sub=both` は相互追加で成立する（MUST）。
- `presence-get` の応答は Roster-Item 形式で、取得できた連絡先だけを返す（MUST）。取得できなかった EID は省略する（MUST）。
- グループ名とニックネームはホームが保存する（MUST）。

### 18.2 Presence の配送範囲（1.0 の補正）

- 場は ROSTER を参照できないため、PRESENCE はそのコミュニティのメンバーで READ を持つ接続に push する（MUST）。クライアントはローカルの連絡先リストで表示を絞る（MAY）。
- ホームからの在席 push は定義しない（MUST NOT）。クライアントは pull する。

### 18.3 カスタム絵文字解決

- `IQ get emoji-resolve`（場接続）: 必須 Community-ID, Emoji-ID（`custom:` の後の ID）。応答: Emoji-URL（`0x0105`）、Avatar-MIME（再利用）、Payload（MAY）。
- 権限: コミュニティメンバーで READ（MUST）。

### 18.4 Avatar-Ref の利用拡大

- `Avatar-Ref`（`0x00FB`）は PRESENCE、ROSTER、MESSAGE に同梱してよい（MAY）。

### 18.5 アバターの一括取得

- `IQ get`（場またはホーム接続）: `Avatar-Bulk`（`0x0107`）に `Avatar-EID`（`0x00F1`）を最大 32 個。応答: `Avatar` × N（取得できたもののみ。権限拒否は黙って省略（MUST））。
- `Avatar-Known-Hash` は一括取得では使わない（MUST NOT）。

### 18.6 アバターの公開範囲

- アバターは全ユーザに公開する（MUST）。プライバシー段階は定義しない（MUST NOT）。
- `Avatar-Privacy` 要素は撤廃した（MUST NOT 送信）。

### 18.7 RECEIPT

- `RECEIPT`（`0x12`）は本仕様から除去した。既読管理は標準機能としない（MUST NOT）。実装が独自に提供してよい（MAY）。

### 18.8 クロスホスト Presence

クロスホストの在席取得はクライアントが各ホームへ pull する（MUST）。S2S で在席を転送してはならない（MUST NOT）。

手順:

1. クライアントは連絡先の EID のドメイン（ホーム）を特定する（MUST）。
2. 自分のホームから `Ticket-Aud = 連絡先のホームの DNS 名` でチケットを発行する（MUST）。
3. 連絡先のホームへ TCP+TLS で接続し、`home-ticket` で認証する（MUST）。
4. `IQ get presence-get` で対象 EID を要求する（MUST）。
5. ホームは認証済み要求者であれば応答する（MUST）。ROSTER によるゲートは行わない（MUST NOT）。rate limit を適用してよい（MAY）。

- 応答は Roster-Item 形式（Show, Status-Text, Avatar-Ref を含む）とする（MUST）。
- 連絡先のホームが応答しない、または認証に失敗した場合、クライアントはその連絡先をオフライン扱いにしてよい（MAY）。
- クライアントは複数ホームへの接続を同時に維持してよい（MAY）。接続数の上限は実装依存。
- Presence の push は各ホーム内で完結する（MUST）。クライアントは pull の結果をローカルで集約する（MAY）。

### 18.9 クロスホスト 1対1（DM）

- 送信者が受信者ホームへ ticket 認証し、受信者の inbox へ直接投稿してよい（MAY）。§16 の禁止を撤廃する。
- 受信者ホームは以下を評価する（MUST）:
  1. 送信者の ticket が有効。
  2. 送信者が受信者のブロックリストに載っていない。
  3. 受信者の `dm-policy` に合致（open=全員、contacts-only=ROSTER の from/both のみ、closed=拒否）。
  4. rate limit 内。
- `dm-policy` のデフォルトは `open` とする（SHOULD）。ユーザは変更してよい（MAY）。ホームに保存する（MUST）。
- 受信拒否時は `not-allowed` を返す（MUST）。理由を詳細に返してはならない（MUST NOT）。
- ROSTER は DM のゲートに使わない（MUST NOT）。`contacts-only` の場合のみ ROSTER を参照する（MAY）。

`dm-policy` は `Dm-Policy`（`0x0108`）1 オクテットで表す: 0=open, 1=contacts-only, 2=closed。IQ set/get でホームに設定・取得する。

### 18.10 dm-policy とブロックリストの IQ 形式

dm-policy（ホーム接続）:

取得:
```
IQ get
  Message-ID = <iq-...>
  IQ-Kind    = 1
→ 応答: IQ result, Dm-Policy = <0/1/2>
```

設定:
```
IQ set
  Message-ID = <iq-...>
  IQ-Kind    = 2
  Dm-Policy  = <0/1/2>
→ 応答: IQ result（空）
```

ブロックリスト（ホーム接続）:

取得:
```
IQ get
  Message-ID = <iq-...>
  IQ-Kind    = 1
→ 応答: IQ result, Blocked-EID × N
```

追加/削除:
```
IQ set
  Message-ID   = <iq-...>
  IQ-Kind      = 2
  Blocked-EID  = <eid>
  Block-Action = 0 (add) / 1 (remove)
→ 応答: IQ result（空）
```

### 18.11 コミュニティ ban の IQ 形式

```
IQ set
  Message-ID   = <iq-...>
  IQ-Kind      = 2
  Community-ID = <cid>
  Target       = <eid>   （Ban 対象の EID。既存要素 To を再利用可）
  Ban-Action   = 0 (ban) / 1 (unban)
  Ban-Reason   = <UTF-8>（MAY, 運用者向け）
→ 応答: IQ result（空）/ not-allowed / not-found
```

- 権限: KICK または MANAGE_ADMIT（MUST）。
- ban 済み EID は community-join が `not-allowed`（MUST）。
- 既メンバーなら除名する（MUST）。

### 18.12 ホストロールの IQ 形式

取得:
```
IQ get
  Message-ID = <iq-...>
  IQ-Kind    = 1
→ 応答: IQ result, Host-Role-Item × N
```

設定:
```
IQ set
  Message-ID     = <iq-...>
  IQ-Kind        = 2
  Host-Role-EID  = <eid>
  Host-Mask      = <32-bit>
→ 応答: IQ result（空）/ not-allowed
```

削除:
```
IQ set
  Message-ID     = <iq-...>
  IQ-Kind        = 2
  Host-Role-EID  = <eid>
  Host-Mask      = 0
→ 応答: IQ result（空）/ not-allowed
```

- 権限: MANAGE_HOST または HOST_OWNER（MUST）。
- HOST_OWNER の EID は削除してはならない（MUST NOT）。

### 18.13 ホスト所有権の譲渡

```
IQ set
  Message-ID         = <iq-...>
  IQ-Kind            = 2
  Host-Owner-Target  = <eid>
→ 応答: IQ result（空）/ not-allowed
```

- 呼び出し元は HOST_OWNER でなければならない（MUST）。
- 譲渡後、呼び出し元の HOST_OWNER ビットを除去し（MUST）、Target に HOST_OWNER を付与する（MUST）。他のビットは呼び出し元に保持してよい（MAY）。
- Target への付与と呼び出し元からの除去は原子的に行わなければならない（MUST）。
- `host-role-set` で HOST_OWNER ビットを直接付与してはならない（MUST NOT）。
- HOST_OWNER が 1 人の場合でも譲渡は許可する（MUST）。譲渡後に 0 人にならないよう保証しなければならない（MUST）。

---

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


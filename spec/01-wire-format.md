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


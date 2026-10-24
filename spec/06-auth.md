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


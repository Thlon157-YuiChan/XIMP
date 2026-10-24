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


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


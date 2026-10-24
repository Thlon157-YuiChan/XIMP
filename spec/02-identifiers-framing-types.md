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


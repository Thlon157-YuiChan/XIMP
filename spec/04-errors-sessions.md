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


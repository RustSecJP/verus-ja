{% raw %}
# spec の量化子（`forall`、`exists`） {#spec-quantifiers-forall-exists}

量化子（quantifier）とトリガー（trigger）についての直感に沿った入門は、チュートリアルの[量化子](quants.html)、とくに [forall とトリガー](forall.html)と [exists と choose](exists.html) を参照してください。

`forall` と `exists` はどちらも、**spec モード専用**の式です。

### 構文 {#syntax}

```verus-grammar
V@[forall_expr] ::= forall |R@[binders...]| V@[spec_expr]
V@[exists_expr] ::= exists |R@[binders...]| V@[spec_expr]
```

本体の V@[spec_expr] には、[トリガー注釈](./trigger-annotations.html)を含めることができます。

### 型付け {#typing}

本体の V@[spec_expr] の型は `bool` でなければなりません。
束縛変数は、本体の中で spec モードの変数として使えます。
`forall` 式と `exists` 式は、どちらも型 `bool` を持ちます。

### 意味論 {#semantics}

`forall|x: T| P(x)` は、型 `T` のすべての値 `x` について `P(x)` が `true` であるとき、かつそのときに限り `true` です。

`exists|x: T| P(x)` は、`P(x)` が `true` になるような型 `T` の値 `x` が少なくとも 1 つ存在するとき、かつそのときに限り `true` です。

この 2 つは双対です。Verus は古典論理を使うので、次が成り立ちます。

```
exists|x: T| P(x)  ≡  !forall|x: T| !P(x)
```

どちらの量化子も、複数の変数を同時に束縛できます。これは、1 変数の量化子を入れ子にすることと等価です。

```rust
// These two are equivalent:
forall|i: int, j: int| i < j ==> f(i) <= f(j)
forall|i: int| forall|j: int| i < j ==> f(i) <= f(j)
```

### トリガー {#triggers}

量化子は無限の領域の上を動くので、SMT ソルバーは、ありうるすべての具体化（instantiation）を列挙することはしません。
代わりに*トリガー*を使います。トリガーとは構文上のパターンであり、証明の文脈にある項がこれに一致すると、一致した値で量化子が具体化されます。

すべての量化子には、少なくとも 1 つのトリガーグループが必要です。
Verus はトリガーを自動的に選ぶことができます。あるいは、量化子の本体に注釈を付けて、明示的に指定することもできます。

| 注釈 | 意味 |
|---|---|
| 部分式に付けた `#[trigger]` | その部分式がトリガーになります（ほかの `#[trigger]` 注釈とひとまとめのグループになります） |
| 部分式に付けた `#[trigger(n)]` | その部分式がトリガーグループ `n` の一部になります |
| 本体の先頭に置いた `#![trigger expr1, expr2, ...]` | `expr1, expr2, ...` が 1 つのトリガーグループになります |
| 本体の先頭に置いた `#![auto]` | トリガーの自動選択を使い、トリガーのログ出力の注記を抑止します |
| 本体の先頭に置いた `#![all_triggers]` | 積極的なトリガーの自動選択を使います |

Verus がトリガーグループをどのように選び、妥当性を確認するかの詳細は、[トリガー注釈](./trigger-annotations.html)を参照してください。
{% endraw %}

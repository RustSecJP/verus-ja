{% raw %}
# spec 等価性（`==`） {#spec-equality-}

> [!DIFF]
> spec モードでは、`==` は数学的な等しさです。exec コードの `==` と同じものではありません。
exec コードの `==` は、[`PartialEq`](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html) トレイトへ処理を振り分けます。

入門は、[等価性](./equality.html)を参照してください。

### 構文 {#syntax}

```verus-grammar
V@[equality_expr] ::= V@[spec_expr] == V@[spec_expr]
                | V@[spec_expr] != V@[spec_expr]
```

### 型付け {#typing}

この式は、左辺と右辺の型が同じであれば、型が正しく付きます。

場合によっては、両辺の[数学的な解釈](./reference-types.html)が同じであるときにも、Verus は `==` を型が正しく付くものとして受け入れることがあります。たとえば、次のとおりです。

 * 2 つの[整数型](./reference-types.html#integer-types)は、つねに比較できます。
 * `T` は `&T` と比較できます。

### 意味論 {#semantics}

`==` 演算子は、両辺が[数学的な解釈](./reference-types.html)において同値であるとき、かつそのときに限り `true` を返します。`!=` 演算子はその否定です。
{% endraw %}

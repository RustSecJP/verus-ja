{% raw %}
# spec のインデックス演算子 [] {#spec-index-operator-}

### 構文 {#syntax}

```verus-grammar
V@[spec_index_expr] ::= V@[spec_expr] [ V@[spec_expr] ]
```

### 脱糖 {#desugaring}

spec コードでは、式 `expr[i]` は `expr.spec_index(i)` へと脱糖（desugar）されます。これは、Rust の[メソッド解決](https://doc.rust-lang.org/reference/expressions/method-call-expr.html)によって通常どおりに解決されます。

> [!DIFF]
> これは、実行可能コードのインデックス演算子とは異なります。
> 実行可能コードでは、インデックス演算子は、スライスや配列にインデックスでアクセスする[場所式（place expression）](https://doc.rust-lang.org/reference/expressions.html#place-expressions-and-value-expressions)であるか、[`Index`](https://doc.rust-lang.org/std/ops/trait.Index.html) トレイトまたは [`IndexMut`](https://doc.rust-lang.org/std/ops/trait.IndexMut.html) トレイトによってオーバーロードされるかのどちらかです。

### 使用例 {#use-cases}

たとえば、次のものがあります。

 * [Seq の `spec_index`](https://verus-lang.github.io/verus/verusdoc/vstd/seq/struct.Seq.html#method.spec_index)
 * [Map の `spec_index`](https://verus-lang.github.io/verus/verusdoc/vstd/map/struct.Map.html#method.spec_index)
 * [スライスの `spec_index`](https://verus-lang.github.io/verus/verusdoc/vstd/slice/trait.SliceAdditionalSpecFns.html#tymethod.spec_index)
{% endraw %}

{% raw %}
# `has` 演算子 {#the-has-operator}

### 構文 {#syntax}

```verus-grammar
V@[has_expr] ::= V@[spec_expr]  has V@[spec_expr]
           | V@[spec_expr] !has V@[spec_expr]
```

### 脱糖 {#desugaring}

spec コードでは、式 `expr1 has expr2` は `expr1.spec_has(expr2)` へと脱糖（desugar）されます。これは、Rust 標準の[メソッド解決](https://doc.rust-lang.org/reference/expressions/method-call-expr.html)によって通常どおりに解決されます。

同様に、式 `expr1 !has expr2` は `!expr1.spec_has(expr2)` へと脱糖されます。

### 例 {#examples}

has 演算子は、ふつう、コレクションが特定の要素を含むことを示すために使います。
`vstd` ライブラリには、`spec_has` 関数が定義されたコンテナー型がいくつか用意されています。たとえば、[`Set::spec_has`](https://verus-lang.github.io/verus/verusdoc/vstd/set/struct.Set.html#method.spec_has) や [`Multiset::spec_has`](https://verus-lang.github.io/verus/verusdoc/vstd/multiset/struct.Multiset.html#method.spec_has) です。

例:

```rust
proof fn test_set() {
    let s: Set<int> = set![1, 2];
    assert(s has 1);
    assert(s has 2);
    assert(s !has 3);
}

proof fn test_multiset() {
    let s: Multiset<int> = Multiset::empty().insert(1).insert(1).insert(2);
    assert(s has 1);
    assert(s has 2);
    assert(s !has 3);
}
```
{% endraw %}

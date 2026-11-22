{% raw %}
# `is` 演算子 {#the-is-operator}

### 構文 {#syntax}

```verus-grammar
V@[is_expr] ::= V@[spec_expr]  is R@[ident]
          | V@[spec_expr] !is R@[ident]
```

### 型付け {#typing}

左辺は何らかの列挙型（enum）の式でなければならず、右辺はその列挙型のバリアントを表す（修飾なしの）識別子でなければなりません。
V@[is_expr] の型は `bool` です。

### 意味論 {#semantics}

`is` 式は、左辺が指定したバリアントに一致するとき `true` を返します。
`!is` 式は、その否定を返します。

### 例 {#example}

```rust
fn test() {
    let x = Some(5);
    assert(x is Some);
    assert(x !is None);
}
```
{% endraw %}

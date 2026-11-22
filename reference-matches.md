{% raw %}
# `matches` 演算子 {#the-matches-operator}

### 構文 {#syntax}

```verus-grammar
V@[matches_expr] ::= V@[spec_expr] matches R@[pattern] &&  V@[spec_expr]
               | V@[spec_expr] matches R@[pattern] ==> V@[spec_expr]
```

### 型付け {#typing}

左辺は R@[pattern] に一致する型でなければならず、右辺は `bool` 型でなければなりません。
右辺では、パターンが束縛した変数を参照できます。

V@[matches_expr] 全体の型は `bool` です。

### 意味論 {#semantics}

「matches-and」式 `expr matches pat && condition` は、次と等価です。

```rust
match expr {
    pat => condition,
    _ => false,
}
```

「matches-implies」式 `expr matches pat ==> condition` は、次と等価です。

```rust
match expr {
    pat => condition,
    _ => true,
}
```
{% endraw %}

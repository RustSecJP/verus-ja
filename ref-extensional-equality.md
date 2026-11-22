{% raw %}
# 外延的等価性（`=~=` と `=~~=`） {#extensional-equality--and-}

**浅い外延的等価性演算子**（shallow extensional equality operator）`=~=` と**深い外延的等価性演算子**（deep extensional equality operator）`=~~=` は、どちらも [spec 等価性（`==`）](./spec-equality.html)と等価です。

これらの演算子の違いは、証明の探索に与える影響だけです。
具体的には、`=~=` 演算子と `=~~=` 演算子を使うと、「外延的等価性」の演算子の適用が引き起こされます。

動機を含めた入門的な説明は、[ガイドのページ](extensional_equality.html)を参照してください。

### 構文 {#syntax}

```verus-grammar
V@[ext_eq_expr] ::= V@[spec_expr] =~=  V@[spec_expr]
              | V@[spec_expr] =~~= V@[spec_expr]
```

### 型付け {#typing}

外延的等価性演算子は、左辺と右辺の型が同じであることを求めます。
この式は [`bool`](./reference-types.html#bool) を返します。

### 意味論 {#semantics}

この演算子は [spec 等価性](./spec-equality.html)と等価です。
{% endraw %}

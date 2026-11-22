{% raw %}
# 連鎖演算子 {#chained-operators}

### 構文 {#syntax}

```verus-grammar
V@[chained_expr] ::= V@[spec_expr] V@[cmp_op] V@[spec_expr] (V@[cmp_op] V@[spec_expr])+
V@[cmp_op]       ::= < | <= | > | >= | ==
```

### 型付け {#typing}

連鎖式のオペランドは、すべて[整数型](./reference-types.html#integer-types)でなければなりません。
この式は `bool` を返します。

### 意味論 {#semantics}

連鎖した比較は、隣り合うすべての組の連言へと脱糖（desugar）されます。途中の値は、それぞれ前後の比較で共有されます。

```
a op1 b op2 c op3 d  ≡  a op1 b && b op2 c && c op3 d
```

たとえば、`a <= b < c` は `a <= b && b < c` と等価です。
{% endraw %}

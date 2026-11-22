{% raw %}
# `assert` {#assert}

### 構文 {#syntax}

```verus-grammar
V@[assert] ::= assert (V@[spec_expr]) ;
```

### 証明上の動作 {#proof-operation}

与えられた式を既定のソルバーで証明し、それ以降のコードではその述語を仮定します。
{% endraw %}

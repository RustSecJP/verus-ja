{% raw %}
# 含意（`==>`、`<==`、`<==>`） {#implication---and-}
### 構文 {#syntax}

```verus-grammar
V@[implies_expr] ::= V@[spec_expr] ==>  V@[spec_expr]
V@[explies_expr] ::= V@[spec_expr] <==  V@[spec_expr]
V@[iff_expr]     ::= V@[spec_expr] <==> V@[spec_expr]
```

### 型付け {#typing}

3 つの演算子はどれも、両辺の型が `bool` であることを求め、`bool` を返します。

### 意味論 {#semantics}

`P ==> Q` は「*P ならば Q*」と読み、`P` が `false` であるか `Q` が `true` であるときはつねに `true` です。

```
P ==> Q  ≡  !P || Q
```

`P <== Q`（*P は Q から導かれる*）はその逆で、`Q ==> P` と等価です。
帰結を先に書くほうが自然な場合に、読みやすさのために役立つことがあります。

`P <==> Q`（*P であるのは Q であるとき、かつそのときに限る*）は、両辺の真理値が同じときに真になります。

```
P <==> Q  ≡  P == Q
```

`<==>` の構文上の[優先順位](./spec-operator-precedence.html)は、`==>` や `<==` よりも低く、`==` よりもずっと低いことに注意してください。
そのため、`==` では括弧が必要になる文脈でも、多くの場合は余分な括弧なしで使えます。
{% endraw %}

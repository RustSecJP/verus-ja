{% raw %}
# assert forall ... by {#assert-forall--by}

`assert forall ... by` 文は、量化された変数を文脈に導入しながら、`forall` 式の証明を書くために使います。

### 構文 {#syntax}

```verus-grammar
V@[assert_by_forall_stmt] ::=
    assert forall |R@[binders]| (V@[spec_expr] implies)? V@[spec_expr] by { V@[proof_stmt]* }
```

通常の `assert` 文と違って、丸括弧が無いことに注意してください。

### 型付け {#typing}

spec 式と証明の本体では、R@[binders] で束縛された変数を参照できます。
spec 式の型は `bool` でなければなりません。

### 証明上の動作 {#proof-operation}

`assert forall |binders| P implies Q` という形の `assert forall` 文では、
述語 `P` が証明の本体に対する仮定として導入され、証明の本体の目標は `Q` を証明することになります。
最終的に、述語 `forall |binders| P ==> Q` が証明され、
証明の残りの部分の文脈で使えるようになります。

簡略化した形 `assert forall |binders| Q` は、
`assert forall |binders| true implies Q` と等価です。

通常の [`assert ... by`](./reference-assert-by.html) 文と同じように、
本体の中の証明は、証明の残りの部分の文脈には入りません。
{% endraw %}

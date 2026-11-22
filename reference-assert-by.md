{% raw %}
# assert ... by {#assert--by}

`assert ... by` 文は、証明をカプセル化するために使います。

### 構文 {#syntax}

```verus-grammar
V@[assert_by_stmt] ::= assert ( V@[spec_expr] ) by { V@[proof_stmt]* }
```

### 証明上の動作 {#proof-operation}

真偽値の述語 `P` について、次のように書いたとします。

```rust
assert(P) by {
    // ... proof here
}
// ... remainder
```

Verus はこの証明を検証し、それを使って `P` を証明しようとします。
ただし、証明の中身は、残りの部分を証明するときの文脈には含まれません。
残りの部分の文脈に持ち込まれるのは `P` だけです。
{% endraw %}

{% raw %}
# opens_invariants {#opens_invariants}

`opens_invariants` 節は、任意の `proof` 関数または `exec` 関数に付けられます。

これは、その関数が開いてよい tracked な不変条件の*名前*の集合を表します。
現時点では、3 つの形があります。Verus がこうした制限を課す理由について詳しくは、[`open_local_invariant` のドキュメント](https://verus-lang.github.io/verus/verusdoc/vstd/macro.open_local_invariant.html#avoiding-reentrancy)を参照してください。

```
fn example()
    opens_invariants any
{
    // Any invariant may be opened here
}
```

または、

```
fn example()
    opens_invariants none
{
    // No invariant may be opened here
}
```

または、

```
fn example()
    opens_invariants [ $EXPR1, $EXPR2, ... ]
{
    // Only invariants with names in [ $EXPR1, $EXPR2, ... ] may be opened.
}
```

### 既定値 {#defaults}

`exec` 関数では、既定は `opens_invariants any` です。

`proof` 関数では、既定は `opens_invariants none` です。
{% endraw %}

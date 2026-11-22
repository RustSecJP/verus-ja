{% raw %}
# 辞書式の decreases 節 {#lexicographic-decreases-clauses}

再帰関数によっては、
すべての再帰呼び出しで減少する値を 1 つだけ指定するのが難しいことがあります。
たとえば[アッカーマン関数](https://en.wikipedia.org/wiki/Ackermann_function)には
2 つの引数 `m` と `n` があり、
3 つの再帰呼び出しのすべてで減少するものは、`m` と `n` のどちらでもありません。

```rust
spec fn ackermann(m: nat, n: nat) -> nat
    decreases m, n,
{
    if m == 0 {
        n + 1
    } else if n == 0 {
        ackermann((m - 1) as nat, 1)
    } else {
        ackermann((m - 1) as nat, ackermann(m, (n - 1) as nat))
    }
}

proof fn test_ackermann() {
    reveal_with_fuel(ackermann, 9);
    assert(ackermann(2, 3) == 9);
}
```

このような場合のために、Verus では `decreases` 節に複数の式を書くことができ、
これらの式を[辞書式順序](https://en.wikipedia.org/wiki/Lexicographic_order)（lexicographic order）で
並んでいるものとして扱います。
たとえば `decreases m, n` は、次のどちらかが真でなければならないことを意味します。
- m が同じままで、n が減少する。
  これは呼び出し `ackermann(m, (n - 1) as nat)` で起こります
- m が減少し、n は任意に増えても減ってもよい。
  これは `ackermann((m - 1) as nat, ...)` の形をした 2 つの呼び出しで起こります

# 相互再帰 {#mutual-recursion}

関数は相互再帰的であってもかまいません。
次の例では、`is_even` が `is_odd` を再帰的に呼び出し、
`is_odd` が `is_even` を再帰的に呼び出します。

```rust
spec fn abs(i: int) -> int {
    if i < 0 {
        -i
    } else {
        i
    }
}

spec fn is_even(i: int) -> bool
    decreases abs(i),
{
    if i == 0 {
        true
    } else if i > 0 {
        is_odd(i - 1)
    } else {
        is_odd(i + 1)
    }
}

spec fn is_odd(i: int) -> bool
    decreases abs(i),
{
    if i == 0 {
        false
    } else if i > 0 {
        is_even(i - 1)
    } else {
        is_even(i + 1)
    }
}

proof fn even_odd_mod2(i: int)
    ensures
        is_even(i) <==> i % 2 == 0,
        is_odd(i) <==> i % 2 == 1,
    decreases abs(i),
{
    if i < 0 {
        even_odd_mod2(i + 1);
    }
    if i > 0 {
        even_odd_mod2(i - 1);
    }
}

fn test_even() {
    proof {
        reveal_with_fuel(is_even, 11);
    }
    assert(is_even(10));
}

fn test_odd() {
    proof {
        reveal_with_fuel(is_odd, 11);
    }
    assert(!is_odd(10));
}
```

ここでの再帰は、`i` が正でも負でも働きます。
どちらの場合も、再帰によって `i` の絶対値である `abs(i)` が減少します。

この相互再帰は、次のように書くこともできます。

```rust
spec fn is_even(i: int) -> bool
    decreases abs(i), 0int,
{
    if i == 0 {
        true
    } else if i > 0 {
        is_odd(i - 1)
    } else {
        is_odd(i + 1)
    }
}

spec fn is_odd(i: int) -> bool
    decreases abs(i), 1int,
{
    !is_even(i)
}

proof fn even_odd_mod2(i: int)
    ensures
        is_even(i) <==> i % 2 == 0,
        is_odd(i) <==> i % 2 == 1,
    decreases abs(i),
{
    reveal_with_fuel(is_odd, 2);
    if i < 0 {
        even_odd_mod2(i + 1);
    }
    if i > 0 {
        even_odd_mod2(i - 1);
    }
}

fn test_even() {
    proof {
        reveal_with_fuel(is_even, 21);
    }
    assert(is_even(10));
}

fn test_odd() {
    proof {
        reveal_with_fuel(is_odd, 22);
    }
    assert(!is_odd(10));
}
```

この別の版では、再帰呼び出し `!is_even(i)` は
`abs(i)` を減少させないので、`abs(i)` だけを `decreases` 節として使うことはできません。
しかし、辞書式順序を使った工夫ができます。
`decreases abs(i), 1` と書けば、
`!is_even(i)` の呼び出しは 1 番目の式 `abs(i)` を同じままに保ちつつ、
2 番目の式を `1` から `0` に減少させるので、
辞書式に減少するという要求を満たします。
呼び出し `is_odd(i - 1)` も辞書式順序に従っています。
1 番目の式 `abs(i)` を減少させるので、
2 番目の式が `0` から `1` に増えてもかまわないからです。
{% endraw %}

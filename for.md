{% raw %}
# for ループ {#for-loops}

前の節では、`triangle` の `while` ループによる実装を紹介しました。

```rust
fn loop_triangle(n: u32) -> (sum: u32)
    requires
        triangle(n as nat) <= u32::MAX,
    ensures
        sum == triangle(n as nat),
{
    let mut sum: u32 = 0;
    let mut idx: u32 = 0;
    while idx < n
        invariant
            idx <= n,
            sum == triangle(idx as nat),
            triangle(n as nat) <= u32::MAX,
        decreases n - idx,
    {
        idx = idx + 1;
        assert(sum + idx <= u32::MAX) by {
            triangle_is_monotonic(idx as nat, n as nat);
        }
        sum = sum + idx;
    }
    sum
}
```

これは、次のように `for` ループとして書き直せます。

```rust
fn for_loop_triangle(n: u32) -> (sum: u32)
    requires
        triangle(n as nat) <= u32::MAX,
    ensures
        sum == triangle(n as nat),
{
    let mut sum: u32 = 0;

    for idx in 0..n
        invariant
            sum == triangle(idx as nat),
            triangle(n as nat) <= u32::MAX,
    {
        assert(sum + idx + 1 <= u32::MAX) by {
            triangle_is_monotonic((idx + 1) as nat, n as nat);
        }
        sum = sum + idx + 1;
    }
    sum
}
```

この `for` ループと `while` ループの違いは、
`idx` が各反復の終わりに自動的に 1 ずつ増やされることと、
不変条件 `idx <= n` がもう必要ないことだけです。

より複雑な例のために、Verus は `for` ループで使われるイテレータについての
ゴースト仕様も提供しています。これは次の節で説明します。
{% endraw %}

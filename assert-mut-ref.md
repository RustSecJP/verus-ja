{% raw %}
# 可変参照についてのアサーション {#assertions-about-mutable-references}

可変参照を含むアサーションを使うと、可変参照が指すメモリについて、*以前の*（関数呼び出しの開始時点の）状態と*現在の*状態を述べることができます。次の例を考えてみましょう。

```rust
fn check_and_assert(a: &mut u32)
    requires *old(a) == 0
{
    assert(*old(a) == 0);
    *a = *a + 1;
    assert(*a == 1);
    *a = *a + 1;
    assert(*a == 2);
    assert(*old(a) == 0);
}

fn asserts() 
{
    let mut x: u32 = 0;
    check_and_assert(&mut x);
}
```

`check_and_assert` の関数呼び出しの中で、`*old(a)` は、関数呼び出しの*開始時点*で可変参照 `a` を参照外しした値を指し、`*a` は、可変参照 `a` を参照外しした*現在の*値を指します。

次は、`assert`、`requires`、`ensures` をすべて使う、もう少し複雑な例です。

```rust
fn decrease(b: &mut u32)
    requires
        *old(b) == 10,
    ensures
        *final(b) == 0,
{
    let mut i: u32 = 0;
    while (*b > 0) 
        invariant
            *b == (10 - i),
        decreases *b,
    {
        *b = *b - 1;
        i = i + 1;
        assert(*b == (10 - i));
    }
    assert(*b == 0);
    assert(*old(b) == 10);
}

fn complex_example()
{
    let mut d: u32 = 10;
    decrease(&mut d);
    assert(d == 0);
}
```
{% endraw %}

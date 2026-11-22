{% raw %}
# break を伴うループ {#loops-with-break}

ループは `return` や `break` を使って早めに抜けることができます。
たとえば、`loop_triangle` 関数から
`triangle(n as nat) < 0x1_0000_0000` という要求を取り除き、
代わりに実行時にオーバーフローを検査したいとします。
次の版の関数は、実行時にオーバーフローを検出した場合に、
`return` を使って特別な値 `0xffff_ffff` を返します。

```rust
fn loop_triangle_return(n: u32) -> (sum: u32)
    ensures
        sum == triangle(n as nat) || (sum == 0xffff_ffff && triangle(n as nat) > u32::MAX),
{
    let mut sum: u32 = 0;
    let mut idx: u32 = 0;
    while idx < n
        invariant
            idx <= n,
            sum == triangle(idx as nat),
        decreases n - idx,
    {
        idx = idx + 1;
        if sum as u64 + idx as u64 > u32::MAX as u64 {
            proof {
                triangle_is_monotonic(idx as nat, n as nat);
            }
            return 0xffff_ffff;
        }
        sum = sum + idx;
    }
    sum
}
```

ループを早めに抜けるもう 1 つの方法は、ループ本体の中で `break` を使うことです。
ただし、`break` はループの仕様を少し複雑にします。
`break` の無い単純な `while` ループでは、
ループを抜けた後にループ条件（たとえば `idx < n`）が
偽になっているはずだと Verus は知っています。
しかし `break` がある場合、ループの後でループ条件が偽であるとは限りません。
ループ条件が真であっても、`break` によってループを抜けるかもしれないからです。
これに対処するため、`break` を伴う `while` ループと、
Rust の `loop` 式（条件の無いループ）では、
次のコードのように、ループを抜けた後に何が真であるかを
`ensures` 節で明示的に指定しなければなりません。
さらに、`break` の後では成り立たない不変条件は、
`invariant` ではなく `invariant_except_break` と印を付けなければなりません。

```rust
fn loop_triangle_break(n: u32) -> (sum: u32)
    ensures
        sum == triangle(n as nat) || (sum == 0xffff_ffff && triangle(n as nat) > u32::MAX),
{
    let mut sum: u32 = 0;
    let mut idx: u32 = 0;
    while idx < n
        invariant_except_break
            idx <= n,
            sum == triangle(idx as nat),
        ensures
            sum == triangle(n as nat) || (sum == 0xffff_ffff && triangle(n as nat) > u32::MAX),
        decreases n - idx,
    {
        idx = idx + 1;
        if sum as u64 + idx as u64 > u32::MAX as u64 {
            proof {
                triangle_is_monotonic(idx as nat, n as nat);
            }
            sum = 0xffff_ffff;
            break;
        }
        sum = sum + idx;
    }
    sum
}
```
{% endraw %}

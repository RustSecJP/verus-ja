{% raw %}
# オーバーフローが起きないことを証明する {#proving-absence-of-overflow}

Verus の実行可能コードが、具体的な（ゴーストでない）変数に対して数学的な演算を行うときは必ず、Verus はそれがオーバーフローしないことを確かめます。
これは役に立ちます。予期しないオーバーフロー（よくあるバグ）を防げるうえ、推論が単純になるからです。たとえば、`x + y` の結果が `(x + y) % pow(2, 32)` になる可能性について SMT ソルバーに推論させずに済みます。
しかしその代わりに、開発者には、すべての演算がオーバーフローし得ないことを証明する義務が生じます。
前の節の[ループ不変条件を考え出す](invariants.html)で見たように、このために追加の証明作業が必要になります。

これに対処する 1 つの方法は、変数の値に明示的な範囲の制約を付けて、コードがオーバーフローし得ないことをソルバーが推論できるようにすることです。
たとえば、和を計算する次の単純な関数は、オーバーフローするかもしれないので検証に失敗します。
```rust
fn compute_sum_fails(x: u64, y: u64) -> (result: u64)
    ensures
        result == x + y,
{
    x + y  // error: possible arithmetic underflow/overflow
}
```
しかし次の版は、範囲の制約によってオーバーフローが防がれることをソルバーが簡単に見て取れるので、検証に成功します。
```rust
fn compute_sum_limited(x: u64, y: u64) -> (result: u64)
    requires
        x < 1000000,
        y < 1000000,
    ensures
        result == x + y,
{
    x + y
}
```

オーバーフローに対処するもう 1 つの方法は、実行時に明示的に検査することです。
これには、`checked_add` や `checked_mul` といった Rust 標準ライブラリの演算を使えます。
これらの演算は `Option` を返し、値 `None` は、オーバーフローが起きるはずだったことを示します。
Verus にはこれらの仕様が含まれているので、そのまま使えます。
次の例は、その使い方を示しています。
```rust
fn compute_sum_runtime_check(x: u64, y: u64) -> (result: Option<u64>)
    ensures
        match result {
            Some(z) => z == x + y,
            None => x + y > u64::MAX,
        },
{
    x.checked_add(y)
}

fn main() {
}

} // verus!


```

これは単独の演算にはうまく働きますが、加算や乗算をいくつも連ねて行うのはもっと厄介です。
オーバーフローが起きたらその時点で止まらなければならず、演算の連鎖全体としてオーバーフローしていたのかどうかがはっきりしないからです。
このような状況のために、Verus の標準ライブラリには `CheckedU8`、`CheckedU16` などの名前の構造体が含まれています。
これらはいずれも、オーバーフローした後でも演算を続けられ、オーバーフローしていない本当の値をゴースト状態に保持します。
この価値を理解するために、前の節の[ループ不変条件を考え出す](invariants.html)に出てきたフィボナッチの例を考えましょう。
次の例のように `CheckedU64` を使えば、結果がオーバーフローし得ないことを証明するために `lemma_fib_is_monotonic` を呼び出す必要はありません。
```rust
fn fib_checked(n: u64) -> (result: u64)
    requires
        fib(n as nat) <= u64::MAX
    ensures
        result == fib(n as nat),
{
    if n == 0 {
        return 0;
    }
    let mut prev: CheckedU64 = CheckedU64::new(0);
    let mut cur: CheckedU64 = CheckedU64::new(1);
    let mut i: u64 = 1;
    while i < n
        invariant
            0 < i <= n,
            fib(n as nat) <= u64::MAX,
            cur@ == fib(i as nat),
            prev@ == fib((i - 1) as nat),
        decreases n - i,
    {
        i = i + 1;
        let new_cur = cur.add_checked(&prev);
        prev = cur;
        cur = new_cur;
    }
    cur.unwrap()
}
```
checked 版は `u64` の代わりに実行時の `Option<u64>` からなるので、性能とメモリ使用量の面でわずかなコストがかかりますが、その見返りとしてコードは単純になります。
もう 1 つの利点は、結果が `u64` に収まらなければならないという事前条件を取り除き、オーバーフローの可能性を正しく扱えることです。
```rust
fn fib_checked_no_precondition(n: u64) -> (result: Option<u64>)
    ensures
        match result {
            Some(x) => x == fib(n as nat),
            None => fib(n as nat) > u64::MAX,
        },
{
    if n == 0 {
        return Some(0);
    }
    let mut prev: CheckedU64 = CheckedU64::new(0);
    let mut cur: CheckedU64 = CheckedU64::new(1);
    let mut i: u64 = 1;
    while i < n
        invariant
            0 < i <= n,
            cur@ == fib(i as nat),
            prev@ == fib((i - 1) as nat),
        decreases n - i,
    {
        i = i + 1;
        let new_cur = cur.add_checked(&prev);
        prev = cur;
        cur = new_cur;
    }
    cur.to_option()
}
```
{% endraw %}

{% raw %}
# exists と choose {#exists-and-choose}

`exists` 式は `forall` 式の双対です。
`forall|i: int| f(i)` が、すべての `i` について `f(i)` が真であることを意味するのに対し、
`exists|i: int| f(i)` は、少なくとも 1 つの `i` について `f(i)` が真であることを意味します。
`exists|i: int| f(i)` を証明するには、
SMT ソルバーは、`f(i)` が真になる `i` の値を 1 つ見つけなければなりません。
この値は、`exists|i: int| f(i)` の *証拠*（witness）と呼ばれます。
`forall` 式と同じように、`exists` 式についての証明もトリガーに基づきます。
具体的には、`exists` 式を証明するために、
SMT ソルバーは `exists` 式のトリガーを使って証拠を見つけようとします。

次の例では、トリガーは `is_even(i)` です。

```rust
proof fn test_exists_succeeds() {
    assert(is_even(4));
    assert(!is_even(5));
    assert(is_even(6));
    assert(exists|i: int| #[trigger] is_even(i));  // succeeds with witness i = 4 or i = 6
}
```

このトリガーに一致する式は 3 つあります。
`is_even(4)`、`is_even(5)`、`is_even(6)` です。
そのうちの 2 つ、`is_even(4)` と `is_even(6)` が、
`exists|i: int| #[trigger] is_even(i)` の証拠になりえます。
これらに基づき、`i = 4` か `i = 6` のどちらかを証拠として、表明は成功します。

対照的に、次のコードでは同じ表明が失敗します。
`is_even(i)` に一致する式がどこにも無いからです。

```rust
proof fn test_exists_fails() {
    assert(exists|i: int| #[trigger] is_even(i)); // FAILS, no match for trigger
}
```

## choose {#choose}

上の証明は、`exists` 式が真であることを証明しようとするものでした。
しかし、`exists` 式が真であることをすでに知っているとします。
たとえば、関数の事前条件として仮定している場合です。
これは、その `exists` 式の証拠が何か存在するはずだということです。
その証拠を手に入れたいなら、`choose` 式を使えます。

`choose` 式 `choose|i: int| f(i)` は、
ヒルベルトの選択演算子
（[イプシロン](https://en.wikipedia.org/wiki/Epsilon_calculus)と呼ばれることもあります）を実装しています。
`f(i)` を満たす値 `i` が存在するなら、そのような値をどれか選びます。
存在しなければ、`i` として任意の値を選びます。

次の例は、事前条件として `exists|i: int| f(i)` を仮定しています。
これに基づいて、SMT ソルバーは、`f(i)` を真にする証拠 `i` が
少なくとも 1 つあることを知っており、
`choose` はこれらの証拠のうちの 1 つを任意に選びます。

```rust
spec fn f(i: int) -> bool;

proof fn test_choose_succeeds()
    requires
        exists|i: int| f(i),
{
    let i_witness = choose|i: int| f(i);
    assert(f(i_witness));
}
```

一方、`exists|i: int| f(i)` であることを知らない場合、
`choose` は、`f(i)` を満たさないかもしれない任意の値を返すだけです
（[ゴーストコードと exec コード](./ghost_vs_exec.html)で説明したとおり、
ゴーストコードはどんな型の任意の値でも作り出せます）。

```rust
proof fn test_choose_fails() {
    let i_witness = choose|i: int| f(i);
    assert(i_witness < 0 || i_witness >= 0); // i_witness is some integer
    assert(f(i_witness)); // FAILS because we don't know exists|i: int| f(i)
}
```

`exists|i: int| f(i)` であることを知っているかどうかにかかわらず、
`choose|i: int| f(i)` 式は常に同じ値を返します。

```rust
proof fn test_choose_same() {
    let x = choose|i: int| f(i);
    let y = choose|i: int| f(i);
    assert(x == y);
}
```

複数の値をまとめて選び、
その値をタプルに集めることもできます。

```rust
spec fn less_than(x: int, y: int) -> bool {
    x < y
}

proof fn test_choose_succeeds2() {
    assert(less_than(3, 7));  // promote i = 3, i = 7 as a witness
    let (x, y) = choose|i: int, j: int| less_than(i, j);
    assert(x < y);
}
```

この例では、SMT ソルバーは
`exists|i: int, j: int| less_than(i, j)` を証明できます。
式 `less_than(3, 7)` が、`i = 3` かつ `j = 7` のときに、
自動的に選ばれたトリガー `less_than(i, j)` に一致し、
`i = 3, j = 7` が証拠の役目を果たすからです。
{% endraw %}

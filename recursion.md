{% raw %}
# 再帰関数、decreases、燃料 {#recursive-functions-decreases-fuel}

再帰関数とは、自分自身を呼び出す関数のことです。
健全性（soundness）を保証するために、再帰的な `spec` 関数は、すべての入力に対して停止しなければなりません。
無限に続く再帰呼び出しは許されません。
停止性がなぜ重要なのかを理解するために、次の停止しない関数定義を考えてみましょう。

```rust
spec fn bogus(i: int) -> int {
    bogus(i) + 1 // FAILS, error due to nontermination
}
```

Verus はこの定義を拒否します。再帰呼び出しが無限に繰り返され、決して停止しないからです。
もし Verus がこの定義を受け入れたら、とても簡単に偽を証明できてしまいます。
たとえば、この定義は `bogus(3) == bogus(3) + 1` であると主張しており、そこから `0 == 1` が導かれますが、これは偽だからです。

```rust
proof fn exploit_bogus()
    ensures
        false,
{
    assert(bogus(3) == bogus(3) + 1);
}
```

停止性の証明を助けるために、Verus は、再帰的な `spec` 関数の定義のそれぞれに `decreases` 節を含めることを要求します。

```rust
spec fn triangle(n: nat) -> nat
    decreases n,
{
    if n == 0 {
        0
    } else {
        n + triangle((n - 1) as nat)
    }
}

```

再帰呼び出しのたびに、`decreases` 節の式は少なくとも 1 減らなければなりません。
さらに、呼び出しによって、その式が 0 を下回ってはいけません。
これらの制限があるので、`decreases` 節の式は、`triangle` が自分自身を呼び出す深さの上限として働き、停止性が保証されます。

Verus はこうした停止性の証明を自動的に完了できることが多いのですが、証明に追加の助けが必要になることもあります。
そのような証明は、いくつかの方法で与えられます。[このページ](./reference-decreases.html#helping-verus-prove-termination)を参照してください。

# 燃料と、再帰関数についての推論 {#fuel-and-reasoning-about-recursive-functions}

上の `triangle` の定義があれば、それについてのアサーションをいくつか書けます。

```rust
fn test_triangle_fail() {
    assert(triangle(0) == 0); // succeeds
    assert(triangle(10) == 55); // FAILS
}
```

`triangle(0)` についての 1 つ目のアサーションは成功します。
しかし、少し意外なことに、アサーション `assert(triangle(10) == 55)` は失敗します。
`triangle(10)` は実際に [55 に等しい](https://en.wikipedia.org/wiki/Triangular_number)にもかかわらずです。
ここで私たちは、自動推論の限界にぶつかったのです。
SMT ソルバーは、すべての再帰関数についてのすべての真の事実を自動的に証明できるわけではありません。

再帰的でない関数については、SMT ソルバーは、単に関数をインライン展開することで、その関数について推論できます。
たとえば、[`min` 関数](spec_functions.html)への呼び出し `min(a + 1, 5)` があるとします。

```rust
spec fn min(x: int, y: int) -> int {
    if x <= y {
        x
    } else {
        y
    }
}
```

SMT ソルバーは、`min(a + 1, 5)` を次のものに置き換えられます。

```
    if a + 1 <= 5 {
        a + 1
    } else {
        5
    }
```

これで、呼び出しが無くなります。
しかし、この戦略は、再帰関数に対しては完全にはうまくいきません。
関数をインライン展開すると、同じ関数への呼び出しを含む別の式ができるからです。

```
triangle(x) = if x == 0 { 0 } else { x + triangle(x - 1) }
```

素朴に考えると、ソルバーは何度も何度もインライン展開を続け、次々と式を作り出していけますが、この戦略は決して停止しません。

```
triangle(x) = if x == 0 { 0 } else { x + triangle(x - 1) }
triangle(x) = if x == 0 { 0 } else { x + (if x - 1 == 0 { 0 } else { x - 1 + triangle(x - 2) }) }
triangle(x) = if x == 0 { 0 } else { x + (if x - 1 == 0 { 0 } else { x - 1 + (if x - 2 == 0 { 0 } else { x - 2 + triangle(x - 3) }) }) }
```

この無限のインライン展開を避けるために、Verus は、ある 1 つの呼び出しが SMT ソルバーの中で生み出せる再帰呼び出しの数を制限します。
この制限を*燃料*（fuel）と呼びます。
入れ子になった再帰的なインライン展開のたびに、燃料を 1 単位消費します。
既定では、燃料は 1 です。
これは、`assert(triangle(0) == 0)` が成功するのにちょうど足りる量ですが、`assert(triangle(10) == 55)` が成功するには足りません。
燃料をもっと多い量に増やすには、`reveal_with_fuel` という指示を使います。

```rust
fn test_triangle_reveal() {
    proof {
        reveal_with_fuel(triangle, 11);
    }
    assert(triangle(10) == 55);
}
```

ここでは、11 単位の燃料があれば、`triangle(0)`, ..., `triangle(10)` の 11 個の呼び出しをインライン展開するのに足ります。
なお、たとえ燃料を 1 単位しか与えたくない場合でも、アサーションを長く連ねることで、`assert(triangle(10) == 55)` を証明できます。

```rust
fn test_triangle_step_by_step() {
    assert(triangle(0) == 0);
    assert(triangle(1) == 1);
    assert(triangle(2) == 3);
    assert(triangle(3) == 6);
    assert(triangle(4) == 10);
    assert(triangle(5) == 15);
    assert(triangle(6) == 21);
    assert(triangle(7) == 28);
    assert(triangle(8) == 36);
    assert(triangle(9) == 45);
    assert(triangle(10) == 55);  // succeeds
}
```

これがうまくいくのは、1 単位の燃料で `assert(triangle(0) == 0)` を証明するのに足りるからです。
そして、ひとたび `triangle(0) == 0` が分かれば、`triangle(1)` を 1 回インライン展開するだけで、次の式が得られます。

```
triangle(1) = if 1 == 0 { 0 } else { 1 + triangle(0) }
```

ここで SMT ソルバーは、先に計算した `triangle(0)` を使って、これを次のように単純化できます。

```
triangle(1) = if 1 == 0 { 0 } else { 1 + 0 }
```

そして、`triangle(1) == 1` を導きます。
同じように、SMT ソルバーは次に、1 単位の燃料を使って `triangle(2)` を `triangle(1)` を使った形に書き換え、`triangle(2) == 3` を証明できます。以下同様です。
しかし、できることなら、アサーションを長く連ねるのは避けるのがおそらく最善です。
その代わりに、SMT による証明が既定ではなぜ失敗するのか（燃料が足りない）を明確にし、まさにその問題だけを直す証明を書くのがよいでしょう。

```rust
fn test_triangle_assert_by() {
    assert(triangle(10) == 55) by {
        reveal_with_fuel(triangle, 11);
    }
}
```
{% endraw %}

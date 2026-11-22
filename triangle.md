{% raw %}
# すべてをまとめる {#putting-it-all-together}

`spec`、`proof`、`exec` のコードがどのように協調して働くかを示すために、n 番目の[三角数](https://en.wikipedia.org/wiki/Triangular_number)を計算する下の例を考えてみましょう。
この例と、そこで使われている機能については、[次の章](recursion_loops.html)でさらに詳しく扱います。
ですから、ここでは、コードの大まかな構造に注目しましょう。

まず、`spec` 関数 `triangle` を使って、自然数（`nat`）と再帰的な記述により、仕様を数学的に定義します。
次に、もっと効率のよい反復による実装を、`exec` 関数 `loop_triangle` として書きます（`exec` が関数の既定のモードであることを思い出してください）。
そして、`loop_triangle` の `ensures` 節の中で、`loop_triangle` の戻り値の正しさを、数学的な仕様に結び付けます。

しかし、`loop_triangle` の検証を成功させるには、さらにいくつか必要なものがあります。
1 つ目に、実行可能コードでは、算術オーバーフローの可能性を気にしなければなりません。
オーバーフローについての推論を単純に保つために、結果が `u32` に収まる必要があるという事前条件を `loop_triangle` に追加します。

また、最終的な `triangle` の結果が `u32` に収まるという知識を、結果を計算する個々のステップがオーバーフローしないという知識に変換する必要もあります。
つまり、`sum = sum + idx` の計算がオーバーフローしないという知識です。
これは、`triangle` が単調であること、つまり、`triangle` の引数を大きくすれば結果も大きくなることを示せば実現できます。
この性質を示すには、[帰納法による証明](induction.html)が必要です。
帰納法による証明は後で扱います。
ここで大事なのは、この証明を `proof` 関数（`triangle_is_monotonic`）を使って行えるということです。
`exec` の実装の中でこの証明の結果を使うには、新しい合計が収まることを[アサート](proof_functions.html#assert-by)し、その根拠として、関係する引数を渡して証明を呼び出します。
呼び出しの場所で、Verus は、`triangle_is_monotonic` の事前条件が成り立つことを検査し、そのうえで事後条件が成り立つと仮定します。

最後に、この実装は while ループを使っています。
そのため、[ループ不変条件](while.html)がいくつか必要になります。
これについても後で扱います。

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

```rust
proof fn triangle_is_monotonic(i: nat, j: nat)
    ensures
        i <= j ==> triangle(i) <= triangle(j),
    decreases j,
{
    // We prove the statement `i <= j ==> triangle(i) <= triangle(j)`
    // by induction on `j`.

    if j == 0 {
        // The base case (`j == 0`) is trivial since it's only
        // necessary to reason about when `i` and `j` are both 0.
        // So no proof lines are needed for this case.
    }
    else {
        // In the induction step, we can assume the statement is true
        // for `j - 1`. In Verus, we can get that fact into scope with
        // a recursive call substituting `j - 1` for `j`.

        triangle_is_monotonic(i, (j - 1) as nat);

        // Once we know it's true for `j - 1`, the rest of the proof
        // is trivial.
    }
}

```

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
{% endraw %}

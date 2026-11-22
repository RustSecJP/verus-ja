{% raw %}
# 再帰的な exec 関数と proof 関数、帰納法による証明 {#recursive-exec-and-proof-functions-proofs-by-induction}

前の節では、三角数の仕様を導入しました。
これをもとに、三角数の実行可能な実装をいくつか順に試してみましょう。
まずは単純な再帰による実装です。

```rust
fn rec_triangle(n: u32) -> (sum: u32)
    ensures
        sum == triangle(n as nat),
{
    if n == 0 {
        0
    } else {
        n + rec_triangle(n - 1) // FAILS: possible overflow
    }
}
```

すると、すぐに実用上の小さな問題にぶつかります。
実装は結果を保持するために有限幅の整数を使う必要があり、
この整数はオーバーフローするかもしれません。

```
error: possible arithmetic underflow/overflow
   |
   |         n + rec_triangle(n - 1) // FAILS: possible overflow
   |         ^^^^^^^^^^^^^^^^^^^^^^^
```

実際、結果が有限幅の整数型に収まらないのであれば、
実装が正しく動くことは期待できません。
そこで、結果が収まらなければならないという事前条件（precondition）を
加えるのが理にかなっています。
結果が `u32` の場合、これは `triangle(n) < 0x1_0000_0000` を意味します。

```rust
fn rec_triangle(n: u32) -> (sum: u32)
    requires
        triangle(n as nat) <= u32::MAX,
    ensures
        sum == triangle(n as nat),
    decreases n,
{
    if n == 0 {
        0
    } else {
        n + rec_triangle(n - 1)
    }
}
```

今度は検証が成功します。
ただし、ここで少し立ち止まって、検証が *なぜ* 成功するのかを理解しておく価値があります。
たとえば `rec_triangle(10)` を実行すると、
10 回の加算が別々に行われ、そのどれもがオーバーフローする可能性があります。
最初の事前条件 `triangle(10) < 0x1_0000_0000` だけから、
この 10 回の加算が *どれも* オーバーフローしないことを、Verus はどうやって知るのでしょうか。

答えは次のとおりです。`n != 0` のときの `triangle(n)` はそれぞれ
`triangle(n - 1)` を再帰的に呼び出し、
この再帰呼び出しは事前条件 `triangle(n - 1) < 0x1_0000_0000` を満たさなければなりません。
これがどのように証明されるのかを見てみましょう。
`rec_triangle` の事前条件から `triangle(n) < 0x1_0000_0000` が分かっていて、
燃料（fuel）を 1 単位使って `triangle` の定義を 1 回インライン展開すると、
次が得られます。

```
triangle(n) < 0x1_0000_0000
triangle(n) = if n == 0 { 0 } else { n + triangle(n - 1) }
```

`n != 0` の場合、これは次のように単純化されます。

```
triangle(n) < 0x1_0000_0000
triangle(n) = n + triangle(n - 1)
```

ここから `n + triangle(n - 1) < 0x1_0000_0000` が導かれます。
`n` の型は非負の `u32` なので `0 <= n` であり、
したがって `triangle(n - 1) < 0x1_0000_0000` となります。

直感的には、`rec_triangle` の実行につれて、
`triangle(n) < 0x1_0000_0000` についての証明がスタックを下って再帰呼び出しへと渡されていく、
と想像できます。
最初の呼び出しで `triangle(10) < 0x1_0000_0000` を証明し、
2 回目の呼び出しで `triangle(9) < 0x1_0000_0000` を、
3 回目の呼び出しで `triangle(8) < 0x1_0000_0000` を証明し、
以下同様に続きます。
（もちろん、証明は実行時には実際には存在しません。
証明は純粋に静的なもので、コンパイルの前に消去されます。
それでも、このように考えるのは理にかなっています。）

## 命令的な実装に向けて: 変更と末尾再帰 {#towards-an-imperative-implementation-mutation-and-tail-recursion}

上で示した再帰による実装は、書くのも検証するのも簡単です。
しかし、再帰のために多くのスタック領域を必要とするので、あまり効率的ではありません。
while ループに基づく、より効率的で命令的な実装に向けて、小さな一歩を 2 つほど進めましょう。
まず、while ループで使うことになる可変の変数に備えて、
`sum` を戻り値から、可変に更新される変数へと切り替えます。

```rust
fn mut_triangle(n: u32, sum: &mut u32)
    requires
        triangle(n as nat) <= u32::MAX,
    ensures
        *final(sum) == triangle(n as nat),
    decreases n,
{
    if n == 0 {
        *sum = 0;
    } else {
        mut_triangle(n - 1, sum);
        *sum = *sum + n;
    }
}
```

検証の観点からは、これで重要な点は何も変わりません。
検証を行うとき、Verus は内部で `*sum` の最終的な値を単に戻り値として表現します。
そのため、`mut_triangle` の検証は `rec_triangle` の検証と本質的に同じになります。

次に、関数を[末尾再帰](https://en.wikipedia.org/wiki/Tail_call)（tail recursive）にして、
スタックの使いすぎをなくしてみましょう。
そのために、while ループがそうするのと同じように、
`0` から `n` まで数え上げる添字の変数 `idx` を導入します。

```rust
fn tail_triangle(n: u32, idx: u32, sum: &mut u32)
    requires
        idx <= n,
        *old(sum) == triangle(idx as nat),
        triangle(n as nat) <= u32::MAX,
    ensures
        *final(sum) == triangle(n as nat),
    decreases n - idx,
{
    if idx < n {
        let idx = idx + 1;
        *sum = *sum + idx;
        tail_triangle(n, idx, sum);
    }
}
```

事前条件と事後条件（postcondition）の中で、
式 `*old(sum)` は関数に入った時点での `*sum` の初期値を指し、
`*sum` は関数から出る時点での最終的な値を指します。
事前条件 `*old(sum) == triangle(idx as nat)` は、
`tail_triangle` が再帰呼び出しを重ねるにつれて、
`sum` に和 `0 + 1 + ... + idx` が蓄積されていくことを指定しています。
再帰呼び出しのたびに `idx` は 1 ずつ増え、やがて `idx` が `n` に達します。
その時点で `sum` は `0 + 1 + ... + n` に等しく、関数は `sum` を変更せずにそのまま返ります。

ところが、`tail_triangle` を検証しようとすると、Verus はオーバーフローの可能性についてのエラーを報告します。

```
error: possible arithmetic underflow/overflow
    |
    |         *sum = *sum + idx;
    |                ^^^^^^^^^^
```

これは最初は不可解に思えるかもしれません。
`rec_triangle` や `mut_triangle` のときのように、
事前条件 `triangle(n as nat) < 0x1_0000_0000` が
オーバーフローの問題を自動的に片付けてくれないのはなぜでしょうか。

問題は、加算と再帰呼び出しの順序を逆にしたことにあります。
`rec_triangle` と `mut_triangle` は、先に再帰呼び出しを行い、
その後で加算を行っていました。
そのおかげで、オーバーフローについて必要な事実をすべて、
一連の再帰呼び出しの中で先に証明し
（たとえば `triangle(10) < 0x1_0000_0000`、`triangle(9) < 0x1_0000_0000`、
……、`triangle(0) < 0x1_0000_0000` を証明し）、
その後で、これらの事実に依存する算術を行うことができました。
しかし `tail_triangle` は、再帰の前に、先に算術を行おうとします。
そのため、元の仮定 `triangle(n) < 0x1_0000_0000` から
これらの事実を導く機会がありません。

## 帰納法による証明 {#proofs-by-induction}

`triangle(10)` を計算する例では、
`triangle(0) < 0x1_0000_0000`、
次に `triangle(1) < 0x1_0000_0000`、
と順に知る必要がありますが、最初に分かっているのは `triangle(10) < 0x1_0000_0000` だけです。
もし何らかの方法で
`triangle(0) <= triangle(10)`、
`triangle(1) <= triangle(10)`、
などが分かっていれば、
`triangle(10) < 0x1_0000_0000` から必要な事実を導けます。
必要なのは、`i <= j` ならば `triangle(i) <= triangle(j)` であることを証明する
*補題*（lemma）です。
言い換えると、`triangle` が単調であることを証明する必要があります。

この補題は `proof` 関数を使って実装できます。

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

証明は j についての帰納法（induction）によります。
帰納法の基底ケースは `i == j` で、
帰納ステップは `j - 1` を `j` に関係づけます。
Verus では、帰納ステップは、証明から自分自身への再帰呼び出しとして実装します
（この例では、`triangle_is_monotonic(i, (j - 1) as nat)` の行がその再帰呼び出しです）。

再帰的な `spec` 関数と同じように、
再帰的な `proof` 関数も停止しなければならず、`decreases` 節が必要です。
そうでなければ、次の停止しない「証明」のように、
`false` を簡単に証明できてしまいます。

```rust
proof fn circular_reasoning()
    ensures
        false,
{
    circular_reasoning(); // FAILS, does not terminate
}
```

補題 `triangle_is_monotonic` を使うと、`tail_triangle` の検証を完成させられます。

```rust
fn tail_triangle(n: u32, idx: u32, sum: &mut u32)
    requires
        idx <= n,
        *old(sum) == triangle(idx as nat),
        triangle(n as nat) <= u32::MAX,
    ensures
        *final(sum) == triangle(n as nat),
    decreases n - idx,
{
    if idx < n {
        let idx = idx + 1;
        assert(*sum + idx <= u32::MAX) by {
            triangle_is_monotonic(idx as nat, n as nat);
        }
        *sum = *sum + idx;
        tail_triangle(n, idx, sum);
    }
}
```

直感的には、`tail_triangle` から `triangle_is_monotonic` への呼び出しは、
`rec_triangle` と `mut_triangle` が再帰呼び出しの中で
事前条件 `triangle(n) < 0x1_0000_0000` を証明するときに行っていたのと
同様の再帰的な証明を行っている、と考えられます。
`rec_triangle` と `mut_triangle` から `tail_triangle` へ移る際に、
この再帰的な推論を、実行可能コードから別の再帰的な補題へと移しただけなのです。
{% endraw %}

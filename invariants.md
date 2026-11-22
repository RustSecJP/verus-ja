{% raw %}
# ループ不変条件を考え出す {#devising-loop-invariants}

以下では、ループの不変条件を考え出す過程をどう進めればよいかを、いくつかの例を作り上げながら示します。

## 例 1: フィボナッチ {#example-1-fibonacci}

[フィボナッチ数列](https://en.wikipedia.org/wiki/Fibonacci_sequence)の値を計算することが目標だとしましょう。
`spec` 関数 `fib` を使い、`nat` と再帰的な記述によって、仕様を数学的に定義します。
```rust
spec fn fib(n: nat) -> nat
    decreases n,
{
    if n == 0 {
        0
    } else if n == 1 {
        1
    } else {
        fib((n - 2) as nat) + fib((n - 1) as nat)
    }
}
```

目標は、もっと効率のよい反復的な実装を、`exec` 関数 `fib_impl` として書くことです。
話を簡単にするために、結果が `u64` の値に収まらなければならないという事前条件（precondition）を `fib_impl` に付けます。
`fib_impl` の戻り値の正しさは、`fib_impl` の `ensures` 節で数学的な仕様と結び付けます。
```rust
fn fib_impl(n: u64) -> (result: u64)
    requires
        fib(n as nat) <= u64::MAX
    ensures
        result == fib(n as nat),
{
    if n == 0 {
        return 0;
    }
    let mut prev: u64 = 0;
    let mut cur: u64 = 1;
    let mut i: u64 = 1;
    while i < n
    {
        i = i + 1;
        let new_cur = cur + prev;
        prev = cur;
        cur = new_cur;
    }
    cur
}
```
しかし、Verus にこのコードの検証を頼むと、エラーが 2 つ報告されます。
```
error: postcondition not satisfied
   |
   |           result == fib(n as nat),
   |           ----------------------- failed this postcondition
```
と
```
error: possible arithmetic underflow/overflow
   |
   |         let new_cur = cur + prev;
   |                       ^^^^^^^^^^
```

1 つ目の失敗から取り組みましょう。
この while ループには不変条件が 1 つもないので、事後条件（postcondition）を満たしていることが Verus に分からなくても不思議ではありません。
つまり、while ループが終わった後、Verus は、ループの条件（`i < n`）がもう成り立たないということ以外、ループの中で何が起きたのかを何も知らないのです。
これを直すために、ループの中で行っている作業のうち、事後条件を満たすことにつながると思われるものを要約する不変条件を作ってみましょう。
この場合、`cur` を返し、それが `fib(n as nat)` であることを期待しているので、`cur == fib(i as nat)` という不変条件を追加しましょう。

Verus を実行すると、これでもまだ事後条件を満たすには足りないことが分かります。
なぜでしょうか。
ループが終わった後に Verus が何を知っているのかを考えてみましょう。
Verus は不変条件を知っており、`!(i < n)` も知っています。これは `i >= n` と同値です。
このように考えると、何が問題なのかは明らかです。Verus は `i == n` であることを知らないのです。
したがって、ループが進むあいだの `i` と `n` の関係を表す不変条件がもう 1 つ必要です。
`i` は 1 から始まり、`n` で終わるはずなので、不変条件 `1 <= i <= n` を追加しましょう。
`i <= n` を使う必要があることに注意してください。ループの最後の反復は `i == n - 1` で始まり、その後 `i` を 1 増やします。そして不変条件は、ループ本体の実行後に常に真でなければならないからです。

この新しい不変条件によって、Verus は事後条件についてのエラーを報告しなくなりました。前進です。
どう前進したのかをはっきりさせておきましょう。
ループが終わった後、Verus は（新しい不変条件のおかげで）`i <= n` を知っています。また、ループが終わったという事実から `i >= n` も知っています。
そのため `i == n` だと結論でき、したがって不変条件 `cur == fib(i as nat)` から、`cur == fib(n as nat)` だと分かるのです。

残念ながら、Verus は依然として算術のアンダーフロー／オーバーフローを懸念しています。
さらに、`cur` についての新しい不変条件が成り立たないという、新しいエラーも報告します。
まずアンダーフロー／オーバーフローの問題に取り組みましょう。
2 つの非負の数を足しているので、問題はオーバーフローの可能性のほうだと推測できます。
また、事前条件から `fib(n as nat)` が `u64` に収まることも分かっています。しかし、この情報を while ループの中で使うには、これも不変条件として追加する必要があります。
なぜそうなのかは、前に出てきた[ループと不変条件についての議論](while.html)を参照してください。
しかし、これでもまだ足りません。
`cur + prev` がオーバーフローしないことを示したいなら、`prev` の値についても何かを知っている必要があります。
`prev` はフィボナッチ数列の 1 つ前の値を追うために使っているので、`prev == fib((i - 1) as nat)` を不変条件として追加しましょう。

これらの新しい不変条件を加えても、Verus は同じエラーを報告します。
なぜでしょうか。
問題は、`fib(n as nat)` がオーバーフローしないと分かっていても、それが必ずしも `fib(i as nat)` がオーバーフローしないことを意味しない点にあります。
人間である私たちには、これが真であることが分かります。`i < n` であり、`fib` は単調だからです（つまり、引数が大きくなるにつれて常に大きくなります）。
しかし、この性質を証明するには[帰納法による証明](induction.html)が必要です。
証明そのものを書く前に、まず欲しい性質を補題として述べるだけにして、それで証明が通るようになるかどうかを見てみましょう。
欲しい性質をくだけた形で述べたものを、Verus で書くと次のようになります。
この性質を確立することだけが目的なので、`proof` モードの関数として書きます。
```rust
proof fn lemma_fib_is_monotonic(i: nat, j: nat)
    requires
        i <= j,
    ensures
        fib(i) <= fib(j),
{
}
```
Verus はまだこれを証明できませんが、while ループの中で呼び出してみましょう。
この補題を `exec` の実装の中で呼び出すには、`proof {}` ブロックを使い、該当する引数を渡します。
呼び出し箇所で、Verus は `lemma_fib_is_monotonic` の事前条件が成り立つことを検査し、その後、事後条件が成り立つと仮定します。
```rust
fn fib_impl(n: u64) -> (result: u64)
    requires
        fib(n as nat) <= u64::MAX
    ensures
        result == fib(n as nat),
{
    if n == 0 {
        return 0;
    }
    let mut prev: u64 = 0;
    let mut cur: u64 = 1;
    let mut i: u64 = 1;
    while i < n
        invariant
            0 < i <= n,
            fib(n as nat) <= u64::MAX,
            cur == fib(i as nat),
            prev == fib((i - 1) as nat),
        decreases n - i,
    {
        i = i + 1;
        proof {
            lemma_fib_is_monotonic(i as nat, n as nat);
        }
        let new_cur = cur + prev;
        prev = cur;
        cur = new_cur;
    }
    cur
}
```
補題の呼び出しは、`i` を 1 増やした後に置きます。
こうすると、これから `cur` と `prev` を足して計算しようとしている新しい `i` の値について、`fib(i as nat) <= fib(n as nat)` が確立されます。
この補題の呼び出しによって、`fib_impl` は検証を通るようになります。

しかし、まだ終わりではありません。
`lemma_fib_is_monotonic` を証明する必要が残っています。
帰納法による証明を組み立てるには、基底の場合を並べ、それから帰納の段階で Verus を助ける必要があります。
基本的な骨組みは次のとおりです。
```rust
proof fn lemma_fib_is_monotonic(i: nat, j: nat)
    requires
        i <= j,
    ensures
        fib(i) <= fib(j),
{
    if j < 2 {
    } else if i == j {
    } else if i == j - 1 {
    } else {
        assume(false);
    }

}
```
最後の厄介な場合に `assume(false)` を追加していることに注意してください（証明を組み立てるための、[assert と assume を使う](assert_assume.html)方法に従っています）。
Verus を実行すると成功します。これは、基底の場合については Verus に助けが要らないことを示しています。
しかし、証明の最後の部分にはもっと助けが必要です（`assume(false)` を取り除いて、証明が失敗するのを見れば確かめられます）。
ここでの鍵となる考えは、帰納法の仮定を使って、`fib(i)` が `fib(j - 1)` と `fib(j - 2)` のどちらよりも小さいことを示すことです。
これは、`lemma_fib_is_monotonic` を 2 回再帰的に呼び出すことで行います。
再帰呼び出しを追加すると、decreases 節を追加する必要が出てくることに注意してください。
この場合、`j` と `i` の間の距離を縮めているので、`j - i` でうまくいきます。
```rust
proof fn lemma_fib_is_monotonic(i: nat, j: nat)
    requires
        i <= j,
    ensures
        fib(i) <= fib(j),
    decreases j - i,
{
    if j < 2 {
    } else if i == j {
    } else if i == j - 1 {
    } else {
        lemma_fib_is_monotonic(i, (j - 1) as nat);
        lemma_fib_is_monotonic(i, (j - 2) as nat);
    }
}
```
これらを追加すると証明は成功し、プログラム全体が検証を通るようになります。


## 例 2: 口座の残高 {#example-2-account-balance}

この例では、銀行口座への一連の入金と出金を表す `i64` の値のスライスが与えられます。
目標は、口座の残高が 0 を下回ることがあるかどうかを判定することです。
この要求を、spec 関数 `always_non_negative` で形式化します。この関数自体は、`i64` の値の列の最初の `i` 個の要素の和を計算することを使って定義されています。
```rust
spec fn always_non_negative(s: Seq<i64>) -> bool
{
    forall|i: int| 0 <= i <= s.len() ==> sum(#[trigger] s[..i]) >= 0
}

spec fn sum(s: Seq<i64>) -> int
    decreases s.len(),
{
    if s.len() == 0 {
        0
    } else {
        sum(s.drop_last()) + s.last()
    }
}
```

実装では、いつものように、具体的な結果 `r` を ensures 節で仕様と結び付けます。
```rust
fn non_negative(operations: &[i64]) -> (r: bool)
    ensures
        r == always_non_negative(operations@),
{
    let mut s = 0i128;
    for i in 0usize..operations.len()
    {
        s = s + operations[i] as i128;
        if s < 0 {
            return false;
        }
    }
    true
}
```
口座の累計を計算するのに `i128` を使っていることに注意してください。こうすれば、オーバーフローせずに十分大きな数を扱えるからです。
練習として、和に `i64` を使うように実装を変え、必要になる不変条件や証明を追加してみるのもよいでしょう。
ここでは `while` ループの代わりに `for` ループを使っているので、`0 <= i <= operations.len()` という不変条件がただで手に入ります。
しかし前と同じように、それだけでは事後条件を証明するにも、オーバーフローが起きないと言い切るにも足りません。Verus は次のように報告します。
```rust
error: postcondition not satisfied
   |
   |           r == always_non_negative(operations@),
   |           ------------------------------------- failed this postcondition
   | / {
   | |     let mut s = 0i128;
   | |     for i in 0usize..operations.len()
   | |     {
...  |
   | |     true
   | | }
   | |_^ at the end of the function body

error: possible arithmetic underflow/overflow
   |
   |         s = s + operations[i] as i128;
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^

```

まず、アンダーフロー／オーバーフローの可能性に取り組みましょう。
なぜ、私たちはこれが起きないと考えるのでしょうか。
アンダーフローが起きないと考えるのは、`s = 0` から始めて、`s` が 0 を下回ったらその時点ですぐに return するからです。
`s` は 0 をどこまで下回り得るでしょうか。
最悪の場合、0 から `i64::MIN` を引くことになるかもしれないので、`i64::MIN <= s` が不変条件だと論じられます。
オーバーフローが起きないと言い切るには、`s` がどこまで大きくなり得るかを考える必要があります。
簡単な上界は、`i` 個の `i64` の値を足し合わせたなら、その和は `i64::MAX * i` で抑えられる、というものです。
これらをまとめると、`i64::MIN <= s <= i64::MAX * i` というループ不変条件を追加できます。
この不変条件があれば、Verus は算術のアンダーフロー／オーバーフローの可能性について文句を言わなくなります。

次に、事後条件が成り立つことを証明できるように、機能的な正しさに取り組みましょう。
前と同じように、ループ不変条件には、事後条件に向けてどこまで進んだのかを要約させたいところです。理想的には、ループが終わったときに、不変条件が事後条件の式とうまく噛み合う形にします。
この方向への第一歩は、`s` が `i` までの値の和を表すことを明示することです。つまり `s == sum(operations@.take(i as int))` です。
また、個々の和をそれぞれ検査して、非負であることを確かめたという事実も覚えておく必要があります。
言い換えると、`forall|j: int| 0 <= j <= i ==> sum(#[trigger] operations@.take(j)) >= 0` が必要です。
これらを追加すると、Verus は新しい文句を言います。
```rust
error: invariant not satisfied at end of loop body
   |
   |             s == sum(operations@.take(i as int)),
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

error: postcondition not satisfied
   |
   |         r == always_non_negative(operations@),
   |         ------------------------------------- failed this postcondition
...
   |             return false;
   |             ^^^^^^^^^^^^ at this exit
```
これは 2 つの別々の問題でしょうか。それとも、根底にある同じ問題の 2 つの症状でしょうか。
これを確かめる 1 つの方法は、1 つ目が真だと仮定して、2 つ目のエラーがどうなるかを見ることです。
つまり、ループ本体を次のように書き換えます。
```rust
s = s + operations[i] as i128;
assume(s == sum(operations@.take((i + 1) as int)));
if s < 0 {
    return false;
}
```
この仮定によって両方のエラーが消えることが分かります。
したがって、本当に必要なのは、最初の `i+1` 個の操作の和を正しく計算したことを Verus に納得させることだけです。
`ops_i_plus_1` を `operations@.take((i + 1) as int)` とし、`sum` の定義を展開すると、`sum(ops_i_plus_1) == sum(ops_i_plus_1.drop_last()) + ops_i_plus_1.last()` であることが分かります。
つまり Verus は、この右辺が、計算した値（すなわち `s + operations[i]`）と一致すると判断するのに手こずっているのです。
2 つの和の第 2 項が一致することを確かめてみましょう。すなわち、次のとおりです。
```rust
assert(operations[i as int] == ops_i_plus_1.last());
```
これは成功します。
したがって問題は、`s`（不変条件から、`sum(operations@.take(i as int))` であると分かっています）が `sum(ops_i_plus_1.drop_last())` に等しいことを示す部分にあるはずです。
sum の引数が等しければ、この 2 つの値が等しいのは明らかです。そこで、Verus がそれを証明できるかどうかを見てみましょう。
次のように書けば確かめられます。
```rust
assert(operations@.take(i as int) == ops_i_plus_1.drop_last());
```
Verus はこの表明を検証できます。それだけでなく、その後、先ほどの仮定 `s == sum(operations@.take((i + 1) as int))` も証明でき、これで証明が完成します。
なぜこれでうまくいくのでしょうか。
この表明を述べると、Verus は[列の外延性](spec_lib.html)についての公理、つまり 2 つの列が等しくなる条件についての公理を呼び出します。
それによって 2 つの列が等しいことを証明でき、そこから和が等しいことが証明され、証明が完成するのです。

検証を通る実装の完全版は次のとおりです。
```rust
fn non_negative(operations: &[i64]) -> (r: bool)
    ensures
        r == always_non_negative(operations@),
{
    let mut s = 0i128;
    for i in 0usize..operations.len()
        invariant
            s == sum(operations@[..i]),
            forall|j: int| 0 <= j <= i ==> sum(#[trigger] operations@[..j]) >= 0,
            i64::MIN <= s <= i64::MAX * i,
    {
        assert(operations@[..i] =~= operations@.take(
            (i + 1) as int,
        ).drop_last());
        s = s + operations[i] as i128;
        if s < 0 {
            return false;
        }
    }
    true
}
```
{% endraw %}

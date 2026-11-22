{% raw %}
# 量化子 {#quantifiers}

列のすべての要素が偶数である、と指定したいとします。
列の大きさが小さく固定されているなら、
要素ごとに別々に仕様を書くことができます。

```rust
spec fn is_even(i: int) -> bool {
    i % 2 == 0
}

proof fn test_seq_5_is_evens(s: Seq<int>)
    requires
        s.len() == 5,
        is_even(s[0]),
        is_even(s[1]),
        is_even(s[3]),
        is_even(s[3]),
        is_even(s[4]),
{
    assert(is_even(s[3]));
}
```

しかし明らかに、これは、もっと大きな列や長さの分からない列にはうまく対応できません。

再帰的な仕様を書くこともできます。

```rust
spec fn all_evens(s: Seq<int>) -> bool
    decreases s.len(),
{
    if s.len() == 0 {
        true
    } else {
        is_even(s.last()) && all_evens(s.drop_last())
    }
}

proof fn test_seq_recursive(s: Seq<int>)
    requires
        s.len() == 5,
        all_evens(s),
{
    assert(is_even(s[3])) by {
        reveal_with_fuel(all_evens, 2);
    }
}
```

しかし、再帰的な定義を使うと、帰納法による証明がたくさん必要になり、
それを書くのにプログラマの多くの労力がかかることがあります。

幸い、Verus と SMT ソルバーは、
[全称量化子と存在量化子](https://en.wikipedia.org/wiki/Quantifier_(logic))（universal and existential quantifiers）である
`forall` と `exists` に対応しています。
これらは、無限個の連言または選言と考えることができます。

```
(forall|i: int| f(i)) = ... f(-2) && f(-1) && f(0) && f(1) && f(2) && ...
(exists|i: int| f(i)) = ... f(-2) || f(-1) || f(0) || f(1) || f(2) || ...
```

これを使うと、列のすべての要素についての仕様を、ずっと手軽に書けます。

```rust
proof fn test_use_forall(s: Seq<int>)
    requires
        5 <= s.len(),
        forall|i: int| 0 <= i < s.len() ==> #[trigger] is_even(s[i]),
{
    assert(is_even(s[3]));
}
```

量化子はとても強力ですが、いくらか注意が必要です。
量化子についての SMT ソルバーの推論は不完全だからです。
これは SMT ソルバーの実装の欠陥ではなく、
もっと根の深い問題です。
量化子・関数・算術を含む論理式が妥当かどうかを判定するのは
決定不能な問題なので、
SMT ソルバーが実装できる完全なアルゴリズムは存在しません。
その代わりに SMT ソルバーは、*トリガー*（trigger）に基づく不完全な戦略を使います。
これは、式がトリガーのパターンに一致したときに量化子を具体化（instantiate）するものです。

この章では、`forall` と `exists` の使い方、
トリガーの仕組み、
そして `choose` 式とクロージャに関連するいくつかの話題を説明します。
{% endraw %}

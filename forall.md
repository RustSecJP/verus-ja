{% raw %}
# forall とトリガー {#forall-and-triggers}

次のコードを詳しく見てみましょう。
このコードは、`requires` 節の `forall` 式を使って
表明を証明しています。

```rust
proof fn test_use_forall(s: Seq<int>)
    requires
        5 <= s.len(),
        forall|i: int| 0 <= i < s.len() ==> #[trigger] is_even(s[i]),
{
    assert(is_even(s[3]));
}
```

この `forall` 式は、ありうるすべての整数 `i` について
`0 <= i < s.len() ==> is_even(s[i])` が成り立つことを意味します。

```
...
0 <= -3 < s.len() ==> is_even(s[-3])
0 <= -2 < s.len() ==> is_even(s[-2])
0 <= -1 < s.len() ==> is_even(s[-1])
0 <= 0 < s.len() ==> is_even(s[0])
0 <= 1 < s.len() ==> is_even(s[1])
0 <= 2 < s.len() ==> is_even(s[2])
0 <= 3 < s.len() ==> is_even(s[3])
...
```

整数 `i` は無限にあるので、上に示した一覧は無限に長くなります。
SMT ソルバーが `forall` を文字どおり無限個の式の一覧に展開することは
期待できません。
しかも、この例で関心があるのは、そのうちの 1 つの式、
つまり `i = 3` の式だけです。
`assert(is_even(s[3]))` を証明するのに必要なのは、これだけだからです。

```rust
0 <= 3 < s.len() ==> is_even(s[3])
```

理想的には、SMT ソルバーは、特定のプログラムの検証に関係しそうな
`i` だけを選びます。
関係しそうな `i` を選ぶために SMT ソルバーが使う最も一般的な手法は、
*トリガー*（trigger）に基づくものです
（SMT パターン、あるいは単に
[パターン](https://microsoft.github.io/z3guide/docs/logic/Quantifiers)とも呼ばれます）。

*トリガー* とは、SMT ソルバーが照合の対象のパターンとして使う、
1 つの式、または式の集まりにすぎません。
上の例では、`#[trigger]` 属性が、式 `is_even(s[i])` を
`forall` 式のトリガーとして印付けています。
この属性に基づいて、
SMT ソルバーは `is_even(s[...])` という形の式を探します。
上に示した `test_use_forall` 関数の検証の間に、
この形をした式が 1 つあります。`is_even(s[3])` です。
これは `i = 3` のとき、トリガー `is_even(s[i])` にぴったり一致します。
このパターンの一致に基づいて、SMT ソルバーは `i = 3` を選び、次の事実を導入します。

```rust
0 <= 3 < s.len() ==> is_even(s[3])
```

この事実によって、SMT ソルバーは表明
`assert(is_even(s[3]))` についての証明を完了できます。

トリガーは、`forall` 式の具体化（instantiation）をプログラムするための手段です
（また、後の節で説明するとおり、`exists` 式の証明をプログラムするための手段でもあります）。
異なるトリガーを選ぶことで、`forall` 式が
上の例の `i = 3` のような、どの値で具体化されるかに影響を与えられます。
たとえば、表明を少し変えて、
`is_even(s[3])` の代わりに `s[3] % 2 == 0` を表明するとします。
数学的には、この 2 つは同値です。
ところが、`s[3] % 2 == 0` についての表明は失敗します。

```rust
spec fn is_even(i: int) -> bool {
    i % 2 == 0
}

proof fn test_use_forall_fail(s: Seq<int>)
    requires
        5 <= s.len(),
        forall|i: int| 0 <= i < s.len() ==> #[trigger] is_even(s[i]),
{
    assert(s[3] % 2 == 0); // FAILS: doesn't trigger is_even(s[i])
}
```

これが失敗するのは、パターン `is_even(s[...])` に一致する式が無いからです。
式 `s[3] % 2 == 0` は、`is_even` にまったく触れていません。
`s[3] % 2 == 0` を証明するには、
先に `is_even(s[3])` に明示的に触れなければなりません。

```rust
proof fn test_use_forall_succeeds1(s: Seq<int>)
    requires
        5 <= s.len(),
        forall|i: int| 0 <= i < s.len() ==> #[trigger] is_even(s[i]),
{
    assert(is_even(s[3]));  // triggers is_even(s[3])
    assert(s[3] % 2 == 0);  // succeeds, because previous line already instantiated the forall
}
```

式 `is_even(s[3])` に促されて、SMT ソルバーが
`forall` 式を `i = 3` で具体化すれば、
SMT ソルバーは、その結果として得られる `0 <= 3 < s.len() ==> is_even(s[3])` を使って
`s[3] % 2 == 0` を証明できます。

あるいは、もっとえり好みの少ないトリガーを選ぶこともできます。
たとえば、トリガー `s[i]` は `s[...]` という形のどんな式にも一致します。
これには、`s[3] % 2 == 0` の中の `s[3]` も、
`is_even(s[3])` の中の `s[3]` も含まれます。

```rust
proof fn test_use_forall_succeeds2(s: Seq<int>)
    requires
        5 <= s.len(),
        forall|i: int| 0 <= i < s.len() ==> is_even(#[trigger] s[i]),
{
    assert(s[3] % 2 == 0);  // succeeds by triggering s[3]
}
```

実は、`#[trigger]` 属性を完全に省略すると、
Verus はトリガー `s[i]` を自動的に選びます。

```rust
proof fn test_use_forall_succeeds3(s: Seq<int>)
    requires
        5 <= s.len(),
        forall|i: int| 0 <= i < s.len() ==> is_even(s[i]), // Verus chooses s[i] as the trigger and prints a note
{
    assert(s[3] % 2 == 0); // succeeds by triggering s[3]
}
```

実際、Verus は、このトリガーを選んだことを伝える注記を表示します。

```
note: automatically chose triggers for this expression:
   |
   |         forall|i: int| 0 <= i < s.len() ==> is_even(s[i]), // Verus chooses s[i] as the trigger
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

note:   trigger 1 of 1:
   |
   |         forall|i: int| 0 <= i < s.len() ==> is_even(s[i]), // Verus chooses s[i] as the trigger
   |                                                     ^^^^

note: Verus printed one or more automatically chosen quantifier triggers
      because it had low confidence in the chosen triggers.
```

もっとも Verus には、
プログラマがトリガーとして `s[i]` を望んでいるのか、`is_even(s[i])` を望んでいるのかは分かりません。
`s[i]` は `is_even(s[i])` より小さいので、Verus は `s[i]` のほうをやや優先し、
`s[i]` を選びます。
しかし同時に、この決定を見直すようプログラマに促す注記も表示します。
プログラマは、量化子の本体の前に `#![auto]` と書くことで、この決定を受け入れられます。
こうすると注記は表示されなくなります。

```rust
proof fn test_use_forall_succeeds4(s: Seq<int>)
    requires
        5 <= s.len(),
        forall|i: int|
            #![auto]
            0 <= i < s.len() ==> is_even(s[i]),  // Verus chooses s[i] as the trigger
{
    assert(s[3] % 2 == 0);  // succeeds by triggering s[3]
}
```

## 有効なトリガーと無効なトリガー {#valid-triggers-and-invalid-triggers}

実際には、有効なトリガーは 2 つの規則に従う必要があります。

1. ある文に対するトリガーは、その文の自由でない変数、つまり `forall` や `exists` によって具体化される変数をすべて含んでいる必要があります。
    これを実現するために、トリガーを複数の部分に分けることができます。これについては[次の節](./multitriggers.html)で詳しく学べます。
2. トリガーには、等しいか等しくないかの比較（`==`、`===`、`!=`、`!==`）、基本的な整数の算術演算子（`<=` や `+` など）、基本的なブール演算子（`&&` など）を含めることができません

たとえば、次の無効なトリガー `0 <= i` を選びたいとします。

```rust
proof fn test_use_forall_bad1(s: Seq<int>)
    requires
        5 <= s.len(),
        forall|i: int| (#[trigger](0 <= i)) && i < s.len() ==> is_even(s[i]),
{
    assert(s[3] % 2 == 0);
}
```

すると、次のエラーになります。

```
error: trigger must be a function call, a field access, or a bitwise operator
    |
    |         forall|i: int| (#[trigger](0 <= i)) && i < s.len() ==> is_even(s[i]),
    |                        ^^^^^^^^^^^^^^^^^^^^
```
どうしてもそうしたいなら、関数を 1 つ余分に導入することで、有効なトリガーにできます。

```rust
spec fn nonnegative(i: int) -> bool {
    0 <= i
}

proof fn test_use_forall_bad2(s: Seq<int>)
    requires
        5 <= s.len(),
        forall|i: int| #[trigger] nonnegative(i) && i < s.len() ==> is_even(s[i]),
{
    assert(is_even(s[3])); // FAILS: doesn't trigger nonnegative(i)
}
```

しかし、コードが `nonnegative(3)` に明示的に触れていないので、このトリガーは一致しません
（コードを動かすには、明示的な `assert(nonnegative(3))` を加えなければなりません）。

## 良いトリガーと悪いトリガー {#good-triggers-and-bad-triggers}

最初の例に戻りましょう。

```rust
proof fn test_use_forall_succeeds3(s: Seq<int>)
    requires
        5 <= s.len(),
        forall|i: int| 0 <= i < s.len() ==> is_even(s[i]), // Verus chooses s[i] as the trigger and prints a note
{
    assert(s[3] % 2 == 0); // succeeds by triggering s[3]
}
```

では……`s[i]` と `is_even(s[i])` のどちらが良いトリガーなのでしょうか。
残念ながら、この種の問いに唯一の最善の答えはありません。
2 つの選択の間にはトレードオフがあります。
トリガー `s[i]` は、`is_even(s[i])` よりも多くのパターン一致を引き起こします。
一致が多いということは、証明の成功を助ける、関係のある具体化を
SMT ソルバーが見つけやすくなるということです。
しかし、一致が多いということは、関係の無い具体化を
SMT ソルバーが生成しやすくなるということでもあります。
そうした具体化は役に立たない情報で SMT ソルバーを詰まらせ、証明を遅くします。

この場合は、おそらく `s[i]` が、選ぶのに良いトリガーです。
このトリガーは、関数 `test_use_forall_succeeds3` が
列 `s` の要素に触れるたびに一致し、
`s` についての推論に役立ちそうな事実をもたらします。

トリガーの悪い選び方の例は、上の節に出てきた `0 <= i` と `nonnegative(i)` です。

- もし `0 <= i` が有効だったとすると、0 以上のどんな値にも一致します。
そこには、`s` と何の関係も無く、`s` に関係しそうにない値も
含まれます。

- さらに、`nonnegative(i)` は `s` に触れていません。
`forall|i: int| 0 <= i < s.len() ==> is_even(s[i])` の
肝心な点は、`s` の要素について何かを述べることであって、
非負の数について何かを述べることではありません。
{% endraw %}

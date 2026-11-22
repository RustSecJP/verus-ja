{% raw %}
# 複数の変数、複数のトリガー、マッチングループ {#multiple-variables-multiple-triggers-matching-loops}

`i` と `j` という複数の変数を持つ `forall` 式があるとします。

```rust
spec fn is_distinct(x: int, y: int) -> bool {
    x != y
}

proof fn test_distinct1(s: Seq<int>)
    requires
        5 <= s.len(),
        forall|i: int, j: int| 0 <= i < j < s.len() ==> #[trigger] is_distinct(s[i], s[j]),
{
    assert(is_distinct(s[2], s[4]));
}
```

上に示した `forall` 式は、`s` のすべての要素が互いに異なることを述べています。
（注: `0 <= i < j < s.len()` の代わりに
`0 <= i < s.len() && 0 <= j < s.len() && i != j`
と書くこともできましたが、
前者のほうが簡潔で、一般性も変わりません。
異なる 2 つの整数が与えられたら、小さいほうを `i`、
大きいほうを `j` とすれば `i < j` となるからです。）

上の例では、トリガー `is_distinct(s[i], s[j])` が
変数 `i` と `j` の両方を含んでおり、
式 `is_distinct(s[2], s[4])` が `i = 2, j = 4` でこのトリガーに一致します。

```
0 <= 2 < 4 < s.len() ==> is_distinct(s[2], s[4])
```

関数呼び出し `is_distinct(s[i], s[j])` を使う代わりに、
単に `s[i] != s[j]` と直接書くこともできます。
しかしその場合、式 `s[i] != s[j]` をトリガーとして使うことはできません。
[前の節](./forall.html)で説明したとおり、
トリガーには `!=` のような、等しいか等しくないかの比較を含められないからです。
ただし、トリガーは単一の式である必要はありません。
次のコードのように、複数の式に分けることができます。
このコードは、トリガーを式の組
`s[i]`、`s[j]` として定義しています。

```rust
proof fn test_distinct2(s: Seq<int>)
    requires
        5 <= s.len(),
        forall|i: int, j: int| 0 <= i < j < s.len() ==> #[trigger] s[i] != #[trigger] s[j],
{
    assert(s[4] != s[2]);
}
```

Verus は、これと同等の別の構文 `#![trigger ...]` にも対応しています。
`#![trigger ...]` は `forall|...|` の直後に置きます。
組 `s[i]`、`s[j]` を直接書きたい場合に使えます。

```rust
proof fn test_distinct3(s: Seq<int>)
    requires
        5 <= s.len(),
        forall|i: int, j: int| #![trigger s[i], s[j]] 0 <= i < j < s.len() ==> s[i] != s[j],
{
    assert(s[4] != s[2]);
}
```

トリガーが組 `s[i]`、`s[j]` のとき、
一致は 4 つあります。`i = 2, j = 2` と `i = 2, j = 4` と `i = 4, j = 2` と `i = 4, j = 4` です。

```
0 <= 2 < 2 < s.len() ==> s[2] != s[2]
0 <= 2 < 4 < s.len() ==> s[2] != s[4]
0 <= 4 < 2 < s.len() ==> s[4] != s[2]
0 <= 4 < 4 < s.len() ==> s[4] != s[4]
```

`i = 2, j = 4` の具体化は s[2] != s[4] を証明し、
これは s[4] != s[2] と同値です。
ほかの具体化は行き止まりです。`2 < 2`、`4 < 2`、`4 < 4` はどれも成り立たないからです。

トリガーは、量化子の変数 `i` と `j` のそれぞれに、少なくとも 1 回は触れなければなりません。
そうでなければ、Verus は文句を言います。

```rust
proof fn test_distinct_fail1(s: Seq<int>)
    requires
        5 <= s.len(),
        forall|i: int, j: int|
            0 <= i < j < s.len() ==> s[i] != #[trigger] s[j], // error: trigger fails to mention i
{
    assert(s[4] != s[2]);
}
```
```
error: trigger does not cover variable i
    |
    | / ...   forall|i: int, j: int|
    | | ...       0 <= i < j < s.len() ==> s[i] != #[trigger] s[j], // error: trigger fails to ment...
    | |__________________________________________________________^
```

複数の式を持つトリガーに一致させるには、
SMT ソルバーはトリガーの中の *すべての* 式について一致を見つけなければなりません。
したがって、
トリガーに式を足せば、いつでもトリガーをより制限の強いものにできます。
たとえば、3 つ目の式 `is_even(i)` を
わざわざトリガーに足すこともできます。そうすると、
`is_even(i)` に一致する式が無いので、一致は失敗します。

```rust
proof fn test_distinct_fail2(s: Seq<int>)
    requires
        5 <= s.len(),
        forall|i: int, j: int| #![trigger s[i], s[j], is_even(i)]
            0 <= i < j < s.len() ==> s[i] != s[j],
{
    assert(s[4] != s[2]); // FAILS, because nothing matches is_even(i)
}
```

この例を成功させるには、`is_even(2)` に明示的に触れなければなりません。
```rust
    assert(is_even(2));
    assert(s[4] != s[2]); // succeeds; we've matched s[2], s[4], is_even(2)
```

# 複数のトリガー {#multiple-triggers}

これまでの例ではすべて、
各量化子が含むトリガーはちょうど 1 つでした
（ただし、そのトリガーが 2 つ以上の式を含むことはありました）。
あまり多くはありませんが、
1 つの量化子に複数のトリガーを指定することもできます。
SMT ソルバーは、トリガーの *どれか* が一致すれば量化子を具体化します。
したがって、トリガーを増やすと、量化子の具体化は *増えます*。
（これは、トリガーに *式* を足す場合とは対照的です。
トリガーに式を足すと、トリガーの制限は強くなり、
量化子の具体化は *減ります*。）

次の例では、`#![trigger a[i], b[j]]` と `#![trigger a[i], c[j]]` の両方を
トリガーとして指定しています。どちらかが明らかに優れているわけではないからです。

```rust
proof fn test_multitriggers(a: Seq<int>, b: Seq<int>, c: Seq<int>)
    requires
        5 <= a.len(),
        a.len() == b.len(),
        a.len() == c.len(),
        forall|i: int, j: int|
            #![trigger a[i], b[j]]
            #![trigger a[i], c[j]]
            0 <= i < j < a.len() ==> a[i] != b[j] && a[i] != c[j],
{
    assert(a[2] != c[4]);  // succeeds, matches a[i], c[j]
}
```

（注: 複数のトリガーを指定するには、`#[trigger]` 構文ではなく
`#![trigger ...]` 構文を使わなければなりません。）

もし量化子が単一のトリガー `#![trigger a[i], b[j]]` だけを挙げていたなら、
上の表明は失敗していたでしょう。`a[2] != c[4]` は `b` に触れていないからです。
単一のトリガー `#![trigger a[i], b[j], c[j]]` はさらに制限が強く、
`b` と `c` の両方が現れることを要求するので、表明はやはり失敗します。

上の例では、明示的なトリガーを省略できます。その場合、
Verus はちょうどこの 2 つのトリガー
`#![trigger a[i], b[j]]` と `#![trigger a[i], c[j]]` を自動的に推論します。
しかし、ほとんどの場合、Verus は 2 つ以上のトリガーを推論することを意図的に避けます。
複数のトリガーは量化子の具体化を増やし、
SMT ソルバーを遅くするおそれがあるからです。
たいていはトリガー 1 つで十分です。

1 つのトリガーのほうが複数のトリガーより安全である例として、
列 `s` の要素 `j` を更新しても要素 `i` は影響を受けない、
と述べる表明を考えましょう。

```rust
proof fn seq_update_different<A>(s: Seq<A>, i: int, j: int, a: A) {
    assert(forall|i: int, j: int|
        0 <= i < s.len() && 0 <= j < s.len() && i != j ==> s.update(j, a)[i] == s[i]);
}
```

これには、実はありうるトリガーが 2 つあります。

```
#![trigger s.update(j, a)[i]]
#![trigger s.update(j, a), s[i]]
```

しかし Verus は、量化子の具体化が多くなりすぎるのを避けるため、
1 つ目だけを選び、2 つ目は退けます。

```
note: automatically chose triggers for this expression:
    |
    |       assert(forall|i: int, j: int|
    |  ____________^
    | |         0 <= i < s.len() && 0 <= j < s.len() && i != j ==> s.update(j, a)[i] === s[i]
    | |_____________________________________________________________________________________^

note:   trigger 1 of 1:
   --> .\rust_verify\example\guide\quants.rs:243:60
    |
    |         0 <= i < s.len() && 0 <= j < s.len() && i != j ==> s.update(j, a)[i] === s[i]
    |                                                            ^^^^^^^^^^^^^^^^^
```

（注: コマンドラインのオプション `--triggers` を使うと、上のメッセージを表示できます。）

# マッチングループ: その正体と避け方 {#matching-loops-what-they-are-and-how-to-avoid-them}

列が整列済みである、と指定したいとします。
これは、列の要素が互いに異なることについての先ほどの `forall` 式と
似た形で書けます。
`s[i] != s[j]` の代わりに `s[i] <= s[j]` と書きます。

```rust
proof fn test_sorted_good(s: Seq<int>)
    requires
        5 <= s.len(),
        forall|i: int, j: int| 0 <= i <= j < s.len() ==> s[i] <= s[j],
{
    assert(s[2] <= s[4]);
}
```

Verus では、これが整列済みであることを表す最善の方法です。
トリガー `s[i], s[j]` がとてもうまく働くからです。
しかし、別のやり方もあります。
`i` と `j` の両方について量化する代わりに、
1 つの変数 `i` だけについて量化し、
`s[i]` を `s[i + 1]` と比べることを試せます。

```rust
proof fn test_sorted_bad(s: Seq<int>)
    requires
        5 <= s.len(),
        forall|i: int|
            0 <= i < s.len() - 1 ==> s[i] <= s[i + 1],
{
    assert(s[2] <= s[4]);
}
```

ところが Verus は、良いトリガーを見つけられなかったと文句を言います。

```
error: Could not automatically infer triggers for this quantifer.  Use #[trigger] annotations to manually mark trigger terms instead.
    |
    | /         forall|i: int|
    | |             0 <= i < s.len() - 1 ==> s[i] <= s[i + 1],
    | |_____________________________________________________^
```

Verus は、式 `0 <= i`、`i < s.len() - 1`、`s[i]`、`s[i + 1]` を
トリガーの候補として検討します。
しかし、`s[i]` を除くすべてが整数の算術を含んでおり、これはトリガーでは許されません。
残った候補である `s[i]` は、一見すると妥当に見えます。
それでも Verus はこれを退けます。無限の *マッチングループ*（matching loop）につながるおそれがあるからです。
トリガーは、SMT ソルバーによる量化子の具体化をプログラムするための手段であり、
注意しないと、無限ループをプログラムしてしまうことがあります。
これがどのように起こるのかを見てみましょう。
`s[i]` をトリガーとして使うことにこだわったとします。

```
forall|i: int|
    0 <= i < s.len() - 1 ==> #[trigger] s[i] <= s[i + 1],
```

（TODO: Verus はここで、マッチングループの可能性についての警告を表示すべきです。）

これは実際に、表明 `s[2] <= s[4]` の検証に成功しますが、
必ずしも良い形での成功ではありません。
SMT ソルバーは `i = 2` と `i = 4` で一致させます。
`i = 2` については、次が得られます。

```
0 <= 2 < s.len() - 1 ==> s[2] <= s[3]
```

これによって新しい式 `s[3]` が生まれ、SMT ソルバーは今度はこれに `i = 3` で一致させられます。

```
0 <= 3 < s.len() - 1 ==> s[3] <= s[4]
```

これで `s[2] <= s[3]` と `s[3] <= s[4]` が分かり、
`s[2] <= s[4]` を証明するのに十分です。
問題は、具体化がここで止まるとは限らないことです。
`s[4]` があれば `i = 4` で一致させることができ、それが `s[5]` を生み、
さらに `i = 5` での一致につながり、と続いていきます。

```
0 <= 4 < s.len() - 1 ==> s[4] <= s[5]
0 <= 5 < s.len() - 1 ==> s[5] <= s[6]
0 <= 6 < s.len() - 1 ==> s[6] <= s[7]
...
```

原理的には、SMT ソルバーは `i = 6`、`i = 7` と、いつまでもループし続ける可能性があります。
実際には、SMT ソルバーは量化子の具体化に打ち切りの上限を設けており、多くの場合
（常にではありませんが）それが無限ループを止めます。
しかし、たとえ SMT ソルバーがループを止めるとしても、
これは依然として非効率な過程であり、
マッチングループは避けるべきです。
（SMT ソルバーに無限の時間とメモリを使わせるマッチングループの例は、
[この節](./profiling.html)を参照してください。）
{% endraw %}

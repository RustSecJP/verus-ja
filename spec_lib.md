{% raw %}
# 仕様ライブラリ: Seq、Set、ISet、Map、IMap {#specification-libraries-seq-set-iset-map-imap}

Verus のライブラリには、`Seq<T>`、`Set<T>`、`ISet<T>`、
`Map<Key, Value>`、`IMap<Key, Value>` という型があります。
これらは仕様の中で、列、有限集合、無限かもしれない集合、
有限のマップ、無限かもしれないマップを表します。
[std::collections](https://doc.rust-lang.org/std/collections/) にある
Rust の実行可能なコレクション型とは対照的に、
`Seq`、`Set`、`ISet`、`Map`、`IMap` の各型は
任意の大きさのコレクションを表します。
たとえば、
[`std::collections::HashSet`](https://doc.rust-lang.org/std/collections/hash_set/struct.HashSet.html) の
`len()` メソッドは、
上限のある `usize` 型の長さを返します。
これに対し、
[`Seq`](https://verus-lang.github.io/verus/verusdoc/vstd/seq/struct.Seq.html#method.len)、
[`Set`](https://verus-lang.github.io/verus/verusdoc/vstd/set/struct.Set.html#method.len)、
[`ISet`](https://verus-lang.github.io/verus/verusdoc/vstd/iset/struct.ISet.html#method.len) の
`len()` メソッドはそれぞれ、
上限の無い `nat` 型の長さを返します。

`Seq`、`Set`、`Map` のオブジェクトは常に有限です。
`ISet` と `IMap` は、無限かもしれない集合とマップを表します。
これによって仕様は、コンピュータの物理メモリには収まらないほど
大きなコレクションについても語ることができます。

無限かもしれない版には、大きな利点が 1 つあります。有限であることを
証明しなくても構築できることです。有限の版には、大きな利点が 2 つあります。
1 つ目は、再帰的な型の中で使えることです。たとえば `enum T` は、
`Set<T>` 型や `Map<T, U>` 型のフィールドを含むことはできますが、`ISet<T>` や `IMap<T, U>` は含められません。
2 つ目は、有限であるという性質が常に成り立つことを検証器が知っていることです。これによって、
SMT ソルバーの段階で証明が思いがけず失敗するのを防げることがあります。たとえば、`Set` に新しい
要素を加えると `len()` は 1 増えますが、`ISet` は大きさが無限かもしれないので、
これが常に成り立つとは限りません。

## Seq、Set、ISet、Map、IMap の構築と利用 {#constructing-and-using-seq-set-iset-map-and-imap}

`seq!`、`set!`、`map!` の各マクロは、指定した中身を持つ `Seq`、`Set`、
`Map` 型の値を構築します。`iset!` と `imap!` のマクロは、
無限かもしれない型である `ISet` と `IMap` の有限の値を構築します。

```rust
proof fn test_seq1() {
    let s: Seq<int> = seq![0, 10, 20, 30, 40];
    assert(s.len() == 5);
    assert(s[2] == 20);
    assert(s[3] == 30);
    assert(s[2..4] == seq![20, 30]);
    assert(s[2..] == seq![20, 30, 40]);
    assert(s[..4] == seq![0, 10, 20, 30]);
}

proof fn test_set1() {
    let s: Set<int> = set![0, 10, 20, 30, 40];
    assert(s.contains(20));
    assert(s.contains(30));
    assert(!s.contains(60));

    let s: ISet<int> = iset![0, 10, 20, 30, 40];
    assert(s.finite());
    assert(s.contains(20));
    assert(s.contains(30));
    assert(!s.contains(60));
}

proof fn test_map1() {
    let m: Map<int, int> = map![0 => 0, 10 => 100, 20 => 200, 30 => 300, 40 => 400];
    assert(m.dom().contains(20));
    assert(m.dom().contains(30));
    assert(!m.dom().contains(60));
    assert(m[20] == 200);
    assert(m[30] == 300);
}

```

上のマクロで構築できるのは、有限の（リテラルの）列・集合・マップだけです。
列、無限かもしれない集合、無限かもしれないマップは、
`Seq::new`、`ISet::new`、`IMap::new` でも構築できます。

`Set` は、有限であることを証明できる場合にしか構築できません。
その方法はいくつかあり、たとえば次のものがあります。
* 数値型 `T`（たとえば `int`、`usize`、`i32`、`u64` など）について、
  `Set::<T>::range(lo, hi)` は、`lo <= i <
  hi` を満たす `i: T` の集合を作ります。（`FiniteRange` トレイトを参照してください。）
* `int` と `nat` 以外の数値型 `T` について、
  `Set::<T>::full().unwrap()` は、`T` 型のすべての値の集合を作ります。
  （`FiniteFull` トレイトを参照してください。）
* 上の方法のどれかを使ったうえで、`Set::filter` や `Set::map` を使って
  `Set` を望みどおりに変形できます。ただし注意があります。`Set::filter` は問題ありませんが、
  `Set::map` は厄介なことがあります。`Set::map` の意味が、
  式
  `self.map(f).contains(b) <==> exists|a: A| self.contains(a) && b == f(a)`
  によって定義されているからです。
  このように `exists` を使っていると、たいていの場合、結果の集合についての証明が
  難しくなり、自動化もされにくくなります。可能であれば、`Set::map_by` や
  `set_build!` など、`Set::map` の代わりになるものを検討してください。
* コントリビュートされたライブラリ [set_build.rs](https://github.com/verus-lang/verus/tree/main/source/builtin_macros/src/contrib/set_build.rs) で
定義されている `set_build!` マクロを使って `Set` を構築できます。

`Map` を構築するには、その定義域を `Set` として構築し、それを
`Map::new` の第 1 引数として渡します。

```rust
proof fn test_seq2() {
    let s: Seq<int> = Seq::new(5, |i: int| 10 * i);
    assert(s.len() == 5);
    assert(s[2] == 20);
    assert(s[3] == 30);
}

proof fn test_set2() {
    let s_finite: Set<int> = Set::range(0, 41).filter(|i: int| i % 10 == 0);
    assert(s_finite.contains(20));
    assert(s_finite.contains(30));
    assert(!s_finite.contains(60));

    let s: ISet<int> = ISet::new(|i: int| 0 <= i <= 40 && i % 10 == 0);
    assert(s.contains(20));
    assert(s.contains(30));
    assert(!s.contains(60));

    let s_infinite: ISet<int> = ISet::new(|i: int| i % 10 == 0);
    assert(s_infinite.contains(20));
    assert(s_infinite.contains(30));
    assert(!s_infinite.contains(35));
}

proof fn test_map2() {
    let m: IMap<int, int> = IMap::new(|i: int| 0 <= i <= 40 && i % 10 == 0, |i: int| 10 * i);
    assert(m[20] == 200);
    assert(m[30] == 300);

    let m_infinite: IMap<int, int> = IMap::new(|i: int| i % 10 == 0, |i: int| 10 * i);
    assert(m_infinite[20] == 200);
    assert(m_infinite[30] == 300);
    assert(m_infinite[90] == 900);

    let m_finite: Map<int, int> = Map::new(Set::range(0, 41), |i: int| 10 * i);
    assert(m_finite[20] == 200);
    assert(m_finite[30] == 300);
}
```

`Map<Key, Value>` の値はそれぞれ、`.dom()` で得られる `Set<Key>` 型の定義域を持ちます。
同様に、`IMap<Key, Value>` はそれぞれ `ISet<Key>` 型の定義域を持ちます。
上の `test_map2` の例では、`m` と `m_finite` はどちらも有限の
定義域 `{0, 10, 20, 30, 40}` を持ち、
`m_infinite` の定義域は無限集合 `{..., -20, -10, 0, 10, 20, ...}` です。

`Set<T>` のコンストラクタの中には、有限集合を作るとは限らないものがあります。それらは
`Option<Set<T>>` を作り、集合が有限のときだけ `Some` になります。したがって、
これらが役に立つのは、有限であることを証明できる場合だけです（通常は再帰を使い、
`ISet::empty()` が有限であることと、`ISet::insert` が有限性を保つことを
利用します）。集合が有限であることを証明できない（あるいは証明する必要が無い）なら、
代わりに `ISet` を使ってください。`Option<Set<T>>` を作る関数の例は
次のとおりです。
* `Set::<T>::new(f)` は、述語 `f: spec_fn(T) -> bool` を満たす
  `T` のすべての要素からなる集合を構築します。
* `Set::<T>::full()` は、`T` のすべての要素からなる集合を構築します。`T` がトレイト
  `FiniteFull` を持つなら、環境に置かれた broadcast 補題だけで、これが `Some` であると分かります。
* `Set::<T>::complement(self)` は、`self` に含まれない
  `T` のすべての要素からなる集合を構築します。
* `Set::<T>::new_from_iset(s)` は、`ISet` である s に含まれる
  `T` のすべての要素からなる集合を構築します。

列の連結（`.add` または `+`）、
列の更新、
集合の和（`.union` または `+`）、
集合の共通部分（`.intersect`）
など、
そのほかの操作については次を参照してください。

- [seq.rs](https://github.com/verus-lang/verus/tree/main/source/vstd/seq.rs)
- [seq_lib.rs](https://github.com/verus-lang/verus/tree/main/source/vstd/seq_lib.rs)
- [set.rs](https://github.com/verus-lang/verus/tree/main/source/vstd/set.rs)
- [set_lib.rs](https://github.com/verus-lang/verus/tree/main/source/vstd/set_lib.rs)
- [iset.rs](https://github.com/verus-lang/verus/tree/main/source/vstd/iset.rs)
- [iset_lib.rs](https://github.com/verus-lang/verus/tree/main/source/vstd/iset_lib.rs)
- [map.rs](https://github.com/verus-lang/verus/tree/main/source/vstd/map.rs)
- [map_lib.rs](https://github.com/verus-lang/verus/tree/main/source/vstd/map_lib.rs)
- [imap.rs](https://github.com/verus-lang/verus/tree/main/source/vstd/imap.rs)
- [imap_lib.rs](https://github.com/verus-lang/verus/tree/main/source/vstd/imap_lib.rs)

[API ドキュメント](https://verus-lang.github.io/verus/verusdoc/vstd/index.html)も参照してください。

## Seq、Set、ISet、Map、IMap の性質の証明 {#proving-properties-of-seq-set-iset-map-and-imap}

上の例で示したとおり、
SMT ソルバーは Seq、Set、ISet、Map、IMap についての性質の一部を自動的に証明します。
しかし、そのほかの性質では、ライブラリの補題を呼び出す必要があったり、
帰納法による証明が必要になったりすることがあります。

2 つのコレクション（`Seq`、`Set`、`ISet`、`Map`、`IMap`）が同じ要素を持つなら、
Verus はそれらを等しいとみなします。
これは[外延性](https://en.wikipedia.org/wiki/Extensionality)（extensionality）による等しさと呼ばれます。
しかし、コレクションが異なる方法で構築されていると、
SMT ソルバーは一般に、
2 つのコレクションが等しいことを自動的には認識しません。
たとえば、次の 3 つの列は等しいのですが、
`check_eq` の呼び出しは失敗します。

```rust
proof fn check_eq(x: Seq<int>, y: Seq<int>)
    requires
        x == y,
{
}

proof fn test_eq_fail() {
    let s1: Seq<int> = seq![0, 10, 20, 30, 40];
    let s2: Seq<int> = seq![0, 10] + seq![20] + seq![30, 40];
    let s3: Seq<int> = Seq::new(5, |i: int| 10 * i);
    check_eq(s1, s2); // FAILS, even though s1 equals s2
    check_eq(s1, s3); // FAILS, even though s1 equals s3
}
```

`s1`、`s2`、`s3` が等しいことを SMT ソルバーに納得させるには、
通常の等価演算子 `==` ではなく、
*外延的な* 等価演算子 `=~=` を使って、等しいことを明示的に表明しなければなりません。
`=~=` を使うと、SMT ソルバーは
コレクションのすべての要素が等しいかどうかを検査するようになります。
これは普段は行わない検査です。その結果、次のコードは成功します。

```rust
proof fn check_eq_extensionally(x: Seq<int>, y: Seq<int>)
    requires
        x =~= y,
{
}

proof fn test_eq() {
    let s1: Seq<int> = seq![0, 10, 20, 30, 40];
    let s2: Seq<int> = seq![0, 10] + seq![20] + seq![30, 40];
    let s3: Seq<int> = Seq::new(5, |i: int| 10 * i);
    check_eq_extensionally(s1, s2); // succeeds
    check_eq_extensionally(s1, s3); // succeeds
}
```

たとえば `assert(s1 =~= s2)` を使えば、元の `check_eq` を呼び出す前に、
`s1` が `s2` に等しいことを証明できます。

```rust
proof fn check_eq(x: Seq<int>, y: Seq<int>)
    requires
        x == y,
{
}

proof fn test_eq2() {
    let s1: Seq<int> = seq![0, 10, 20, 30, 40];
    let s2: Seq<int> = seq![0, 10] + seq![20] + seq![30, 40];
    let s3: Seq<int> = Seq::new(5, |i: int| 10 * i);
    assert(s1 =~= s2);
    assert(s1 =~= s3);
    check_eq(s1, s2); // succeeds
    check_eq(s1, s3); // succeeds
}
```

（なお、Verus は既定で、`assert`、`ensures`、`invariant` の中の
`==` を自動的に `=~=` に格上げします。
そのため、たとえば `assert(s1 == s2)` は実際には `assert(s1 =~= s2)` を意味します。
詳しくは[外延性による等しさ](extensional_equality.html)の節を参照してください。）

`ISet` と `Set` は型が一致しないので、決して等しくなりません。
両者を関係づける必要が出てきたら、述語 `Set::congruent` を検討するか、
等しいかどうかを検査する前に `to_iset()` で `Set` を `ISet` に変換してください。
（ライブラリ以外のコードでは、可能であれば、
有限の版か無限の版のどちらか一方だけを使うのが最善の習慣です。）

集合の濃度（`Set::len`）と集合の有限性（`ISet::finite`）についての証明では、
帰納法による証明が必要になることがよくあります。
たとえば、2 つの集合の共通部分の正確な濃度は、
2 つの集合がどの要素を共有しているかによって決まります。
2 つの集合が互いに素であれば、
共通部分の濃度は 0 になりますが、
そうでなければ、共通部分の濃度は 0 でない何らかの値になります。
共通部分の濃度が、2 つの集合のどちらの濃度よりも大きくないことを
証明してみましょう。
一般性を失うことなく、
共通部分の濃度が 1 つ目の集合の濃度より大きくないこと、
つまり `s1.intersect(s2).len() <= s1.len()` だけを証明すれば十分です。

この証明（[set_lib.rs](https://github.com/verus-lang/verus/tree/main/source/vstd/set_lib.rs) にあります）は、
集合 `s1` の大きさについての帰納法によります。
帰納ステップでは `s1` を小さくする必要があり、
そのためには `s1` から要素を 1 つ取り除く必要があります。
`.choose` と `.remove` という 2 つのメソッドを使うと、
`s1` から任意の要素を選び、それを取り除けます。

```rust
let a = s1.choose();
... s1.remove(a) ...
```

これをもとにすると、帰納法による証明は次のようなものになると予想されます。
帰納ステップでは `s1.choose()` を取り除きます。

```rust
pub proof fn lemma_len_intersect<A>(s1: Set<A>, s2: Set<A>)
    ensures
        s1.intersect(s2).len() <= s1.len(),
    decreases
        s1.len(),
{
    if s1.is_empty() {

    } else {
        let a = s1.choose();

        lemma_len_intersect(s1.remove(a), s2);
    }
}
```

残念ながら、Verus はこの証明の検証に失敗します。
そのため、基底ケースと帰納ケースに、もう少し詳しい内容を補う必要があります。
この詳細をコードに加える前に、
完全に明示的な証明を手で書き下したらどのようになるかを考えてみましょう。

```rust
pub proof fn lemma_len_intersect<A>(s1: Set<A>, s2: Set<A>)
    ensures
        s1.intersect(s2).len() <= s1.len(),
    decreases
        s1.len(),
{
    if s1.is_empty() {
        // s1 is the empty set.
        // Therefore, s1.intersect(s2) is also empty.
        // So both s1.len() and s1.intersect(s2).len() are 0,
        // and 0 <= 0.
    } else {
        // s1 is not empty, so it has at least one element.
        // Let a be an element from s1.
        // Let s1' be the set s1 with the element a removed (i.e. s1' == s1 - {a}).
        // Removing an element decreases the cardinality by 1, so s1'.len() == s1.len() - 1.
        // By induction, s1'.intersect(s2).len() <= s1'.len(), so:
        //   (s1 - {a}).intersect(s2).len() <= s1'.len()
        //   (s1.intersect(s2) - {a}).len() <= s1'.len()
        //   (s1.intersect(s2) - {a}).len() <= s1.len() - 1
        // case a in s1.intersect(s2):
        //   (s1.intersect(s2) - {a}).len() == s1.intersect(s2).len() - 1
        // case a not in s1.intersect(s2):
        //   (s1.intersect(s2) - {a}).len() == s1.intersect(s2).len()
        // In either case:
        //   s1.intersect(s2).len() <= (s1.intersect(s2) - {a}).len() + 1
        // Putting all the inequalities together:
        //   s1.intersect(s2).len() <= (s1.intersect(s2) - {a}).len() + 1 <= (s1.len() - 1) + 1
        // So:
        //   s1.intersect(s2).len() <= (s1.len() - 1) + 1
        //   s1.intersect(s2).len() <= s1.len()
    }
}
```

これほど単純な性質にしては、驚くほど長い証明です。
幸い、SMT ソルバーは上に書いた手順のほとんどを自動的に証明できます。
ただし、先に述べたとおり、
外延性による等しさを必要とする手順は自動的には証明しません。
外延性による等しさを必要とする重要な手順は、次の 2 つです。
- 「Therefore, s1.intersect(s2) is also empty.」（したがって、s1.intersect(s2) も空である。）
- `(s1 - {a}).intersect(s2)` を `s1.intersect(s2) - {a}` に置き換えること

これらについては、`=~=` を明示的に使う必要があります。

```rust
pub proof fn lemma_len_intersect<A>(s1: Set<A>, s2: Set<A>)
    ensures
        s1.intersect(s2).len() <= s1.len(),
    decreases
        s1.len(),
{
    if s1.is_empty() {
        assert(s1.intersect(s2) =~= s1);
    } else {
        let a = s1.choose();
        assert(s1.intersect(s2).remove(a) =~= s1.remove(a).intersect(s2));
        lemma_len_intersect(s1.remove(a), s2);
    }
}
```

これで、Verus と SMT ソルバーは証明を無事に完了します。
しかし、この証明の読み手は Verus と SMT ソルバーだけではありません。
このコードを保守する人は誰でも、なぜ `=~=` を使ったのかを知りたくなるかもしれません。
それを再発見するために、上の手書きの証明をまるごと自分で組み立て直させるのは、
おそらく避けるべきでしょう。
そこで、厳密には必要ありませんが、
表明を `assert...by` で包んで `=~=` の目的を示しておくのが、
おそらく親切です。

```rust
pub proof fn lemma_len_intersect<A>(s1: Set<A>, s2: Set<A>)
    ensures
        s1.intersect(s2).len() <= s1.len(),
    decreases s1.len(),
{
    if s1.is_empty() {
        assert(s1.intersect(s2).len() == 0) by {
            assert(s1.intersect(s2) =~= s1);
        }
    } else {
        let a = s1.choose();
        lemma_len_intersect(s1.remove(a), s2);
        // by induction: s1.remove(a).intersect(s2).len() <= s1.remove(a).len()
        assert(s1.intersect(s2).remove(a).len() <= s1.remove(a).len()) by {
            assert(s1.intersect(s2).remove(a) =~= s1.remove(a).intersect(s2));
        }
        // simplifying ".remove(a).len()" yields s1.intersect(s2).len() <= s1.len())

    }
}
```

---
{% endraw %}

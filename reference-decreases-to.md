{% raw %}
# `decreases_to!` {#decreases_to}

式 <code>decreases_to!(e<sub>1</sub>, e<sub>2</sub>, ..., e<sub>n</sub> => f<sub>1</sub>, f<sub>2</sub>, ..., f<sub>n</sub>)</code> は、
左辺の列
<code>e<sub>1</sub>, e<sub>2</sub>, ..., e<sub>n</sub></code>
が、右辺の列
<code>f<sub>1</sub>, f<sub>2</sub>, ..., f<sub>n</sub></code>
へ*辞書式に減少する*（lexicographically-decreases-to）かどうかを表す bool です。

辞書式の減少（lexicographic-decreases-to）は、
spec 関数の `decreases` の尺度を検査するために使われます。

辞書式の減少についての入門的な説明は、[チュートリアルのこの章](./lex_mutual.html)を参照してください。

### 構文 {#syntax}

```verus-grammar
V@[decreases_to_expr] ::= decreases_to!( V@[spec_expr] (, V@[spec_expr])* => V@[spec_expr] (, V@[spec_expr])* )
```

### 型付け {#typing}

`=>` の両側にある 2 つの式の並びは、要素の数が同じでなければなりません。
型付けについての要件は、ほかにありません。

### 意味論 {#semantics}

次の条件を満たす `k`（`1 <= k <= n`）が存在するとき、
<code>e<sub>1</sub>, e<sub>2</sub>, ..., e<sub>n</sub></code>
は
<code>f<sub>1</sub>, f<sub>2</sub>, ..., f<sub>n</sub></code>
へ*辞書式に減少する*と言います。

 * <code>e<sub>k</sub></code> が <code>f<sub>k</sub></code> へ減少する（decreases-to）。
 * `1 <= i < k` であるそれぞれの `i` について、
    <code>e<sub>i</sub> == f<sub>i</sub></code> である。

*減少*（decreases-to）の関係は、*すべての*値の上の半順序です。型の異なる値どうしも比較*できます*。この関係では、次のものが成り立ちます（ただし、これらに限られるとは限りません）。

 * `x` と `y` が整数で、`x > y >= 0` であれば、`x` は `y` へ*減少*します。
 * `a` がデータ型（構造体、タプル、列挙型）で、`f` がその「再帰の可能性がある」フィールドの 1 つであれば、`a` は `a.f` へ*減少*します。
   * データ型 `X` のフィールドは、`X` または `X` のジェネリック型パラメータに言及している場合に、「再帰の可能性がある」とみなされます。
 * `f` が `spec_fn` であれば、`f` は `f(i)` へ*減少*します。
 * `s` が `Seq` であれば、`s` は `s[i]` へ*減少*します。
 * `s` が `Seq` であれば、指定した範囲が `0 .. s.len()` より真に小さい場合、`s` は `s.subrange(i, j)` へ*減少*します。
   * [`axiom_seq_len_decreases`](https://verus-lang.github.io/verus/verusdoc/vstd/seq/fn.axiom_seq_len_decreases.html) は、より一般的な公理を提供します。これは明示的に呼び出さなければなりません。
 * `v` が `Vec` であれば、`v` は [`v@`](./reference-at-sign.html) へ*減少*します。

これらの公理は、該当する式（たとえば `x.f`、`x->f`、`s[i]`、`v@`）が `decreases-to` 式の一部として使われたときにトリガーされます。

### 注意 {#notes}

 1. タプルは辞書式には比較*されません*。タプルはデータ型であり、上で説明したとおりに比較されます。たとえば、`a` は `a.0` へ*減少*します。
    辞書式に比較されるのは、`decreases_to!` 式の「最上位」の列だけです。

 2. 列（`Seq`）は、`len()` だけでは比較されません。ただし、`s` の代わりに `s.len()` を減少の尺度としていつでも使えます。

### 例 {#examples}

```rust
proof fn example_decreases_to(s: Seq<int>)
    requires s.len() == 5
{
    assert(decreases_to!(8int => 4int));

    // fails: can't decrease to negative number
    // assert(decreases_to!(8 => -2));

    // Comma-separated elements are treated lexicographically:
    assert(decreases_to!(12int, 8int, 1int => 12int, 4int, 50000int));

    // Datatypes decrease-to their fields:
    let x = Some(8int);
    assert(decreases_to!(x => x->0));

    let y = (true, false);
    assert(decreases_to!(y => y.0));

    // fails: tuples are not treated lexicographically
    // assert(decreases_to!((20, 9) => (11, 15)));

    // sequence decreases-to an element of the sequence
    assert(decreases_to!(s => s[2]));

    // sequence decreases-to a subrange of the sequence
    assert(decreases_to!(s => s[1..3]));
}
```
{% endraw %}

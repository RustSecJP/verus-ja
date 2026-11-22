{% raw %}
# イテレータ {#iterators}

Verus は、Rust の
[`Iterator`](https://doc.rust-lang.org/std/iter/trait.Iterator.html) トレイトを実装した
任意の型に対する `for` ループの検証に対応しています。ただし、その型に適切な Verus の
仕様が付いていることが条件です。このページでは、Verus が `vstd` で提供している仕様を
使ってループ不変条件を書く方法を説明します。自分で作ったイテレータ型に仕様を
付ける方法については、[独自の型のためのイテレータ仕様](./iterator-specs.html)を参照してください。

## for ループの中のゴースト状態 {#ghost-state-in-for-loops}

Verus では、Rust が `for` ループの内部で使うイテレータに名前を付けられます。

```rust
for x in my_iter: some_expression
    invariant
        // can refer to my_iter.index() and my_iter.seq()
{
    // x is the current element; my_iter tracks ghost state
}
```
この名前（上の例では `my_iter`）を使うと、これまでの反復回数を
`my_iter.index()` として参照できます。また、`my_iter.seq()` を使うと、
イテレータが開始位置から生み出していく要素の完全な列を参照できます。
これは *予言的な*（prophetic）値です。イテレータがまだ走っていなくても、
Verus はこの値をあらかじめ知っています。`break` の無い `for` ループでは、
ループが完了した後に `my_iter.index() == my_iter.seq().len()` であることも分かります。

### 例: すべての要素が正であることの検査 {#example-checking-all-elements-are-positive}

```rust
fn all_positive(v: &Vec<u8>) -> (b: bool)
    ensures
        b <==> (forall|i: int| 0 <= i < v.len() ==> v[i] > 0),
{
    let mut b: bool = true;

    for x in iter: vec_iter(v)
        invariant
            b <==> (forall|i: int| 0 <= i < iter.index() ==> v[i] > 0),
    {
        b = b && *x > 0;
    }
    b
}
```

この不変条件は、`iter.index()` を通じて進み具合を追跡しています。ループが終わると、
`iter.seq().len()` までのすべての `i` について不変条件が成り立っていたことが分かり、
この値は `v.len()` に等しくなります。

### 範囲のイテレータ {#range-iterators}

Rust の標準的な整数の範囲も、同じように扱えます。

```rust
fn build_range(n: u32) -> (v: Vec<u32>)
    ensures
        v.len() == n,
        forall|i: int| 0 <= i < n ==> v[i] == i,
{
    let mut v: Vec<u32> = Vec::new();
    for i in r_iter: 0..n
        invariant
            v.len() == r_iter.index(),
            forall|j: int| 0 <= j < v.len() ==> v[j] == r_iter.seq()[j],
    {
        v.push(i);
    }
    v
}
```

範囲 `0..n` では `r_iter.seq()[k] == k` なので、`r_iter.seq()[r_iter.index()]` は
現在の要素 `i` に等しくなります。

### ラッパーの省略 {#omitting-the-wrapper}

不変条件でゴースト状態が必要ないなら、`my_iter:` の束縛を省略できます。

```rust
fn sum_multiples_of_3() -> u64 {
    let mut n: u64 = 0;
    for x in 0..10
        invariant n == x * 3,
    {
        n += 3;
    }
    assert(n == 30);
    n
}
```

### 逆順の反復 {#reversed-iteration}

同じ書き方の仕様が、イテレータのアダプタにも当てはまります。
たとえば、Rust の
[`DoubleEndedIterator`](https://doc.rust-lang.org/std/iter/trait.DoubleEndedIterator.html) を
実装したイテレータでは、`.rev()` を呼び出して逆順に反復できます。`seq()` と `index()` の
仕様はまったく同じように働き、逆順にした列を指します。

```rust
fn test_reversed(v: &Vec<u8>) -> (w: Vec<u8>)
    ensures
        w@ == v@.reverse(),
{
    let mut w: Vec<u8> = Vec::new();
    for x in iter: v.iter().rev()
        invariant
            w.len() == iter.index(),
            forall|i: int| 0 <= i < w.len() ==> w@[i] == *iter.seq()[i],
    {
        w.push(*x);
    }
    w
}
```
{% endraw %}

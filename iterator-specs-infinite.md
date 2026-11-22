{% raw %}
# 無限イテレータのイテレータ仕様を実装する {#implementing-iterator-specifications-for-infinite-iterators}

少し意外かもしれませんが、無限イテレータを実装する独自の型についても、同じ[イテレータの実装を検証するために必要な手順](iterator-specs.html)に従うことができます。

まずは、カウンタをもとにした無限イテレータから始めましょう。このカウンタは `next()` を呼び出すたびに 1 ずつ増え、オーバーフローしそうになったらラップアラウンドします。

### 1. イテレータの構造体
{: id="1-the-iterator-struct"}

`IterCtr` は、64 ビットのカウンタの値（`count`）と、[予言的（prophetic）](https://verus-lang.github.io/verus/verusdoc/vstd/proph)な長さのフィールドを持ちます。このフィールドは、戻り値の列がどれだけの長さになるかを定めます。この時点では、その長さがいくつなのかは分かりません。何らかの値を持つことだけが分かっています。

```rust
struct IterCtr {
    count: u64,
    len: Tracked<ProphecyGhost<nat>>,
}
```

### 2. `next` メソッド
{: id="2-the-next-method"}

これは、ごく普通の Rust の `Iterator` の実装です。ただし、`next()` がその義務を果たしていることの証明を含める必要があります。特に、`next()` が返す値が、予言された `remaining` の列の最初の要素と一致するという義務です。ここでの鍵となる考え方は、（`self.len` にある）既存の予言変数（prophecy variable）を、新しく作った予言変数と入れ替えることです。そのうえで、古い予言変数を、その最終的な値が、いま作った予言変数よりも 1 だけ大きくなるように解決します。これだけで、`next()` の仕様が求めるとおり、予言的な列の長さが 1 だけ減ったことを示せます。

```rust
impl Iterator for IterCtr {
    type Item = u64;

    fn next(&mut self) -> (ret: Option<Self::Item>) {
        proof {
            let tracked mut new = ProphecyGhost::new();
            tracked_swap(&mut new, &mut self.len);
            new.resolve_dependent(&self.len, |x:nat| (x + 1) as nat);
            // We learn: old(self).len().value() == final(self).len().value() + 1
        }
        let ret = self.count;
        if self.count == u64::MAX {
            self.count = 0;
        } else {
            self.count = self.count + 1;
        }
        Some(ret)
    }
}
```

### 3. 仕様の実装
{: id="3-the-spec-implementation"}

Verus は `vstd` の中で `IteratorSpec` を提供しています。これは Rust の [`Iterator`](https://doc.rust-lang.org/std/iter/trait.Iterator.html) トレイトを拡張したもので、さまざまな仕様関数と、`next()` 関数に対する Verus の仕様を定義しています。独自のイテレータについて推論できるようにするには、Verus が提供する `IteratorSpecImpl` トレイトを実装する必要があります（仕様を定義している `IteratorSpec` トレイトではありません。詳しくは[「外部トレイトの仕様」](external_trait_specifications.html)を参照してください）。それぞれの関数の目的は、[有限イテレータ](iterator-specs-finite.html#3-the-spec-implementation)の説明で述べています。ここでは、独自のイテレータについてそれぞれを次のように定義します。

- **`obeys_prophetic_iter_laws`** — この実装は、`IteratorSpec` が定める仕様に従うことが検証されているので、`true` を返します。
- **`remaining`** — 予言的な列は、`self.len` が予言する何らかの（未知の）長さを持ちます。中身は単純で、カウンタの現在の値から始まり、1 ステップごとに 1 ずつ増え、`u64::MAX` の次でラップアラウンドします。
- **`produces`** — next が呼ばれたときは必ず値を返すので、すべての添字を true にします。
- **`decrease`** — これは無限イテレータなので、有効な decreases の尺度がありません。そのため `None` を返します。
- **`peek`** — この値がどうなるかは簡単に予測でき、自動化の助けにもなるので、適切な値を入れた `Some` を返します。

```rust
impl IteratorSpecImpl for IterCtr {
    open spec fn obeys_prophetic_iter_laws(&self) -> bool { true }

    #[verifier::prophetic]
    closed spec fn remaining(&self) -> Seq<Self::Item> {
        Seq::new(self.len@.value(), |i:int| ((i + self.count) % (u64::MAX as int + 1)) as u64)
    }

    #[verifier::prophetic]
    closed spec fn produces(&self) -> Seq<bool> {
        Seq::new(self.len@.value(), |_i: int| true)
    }

    open spec fn decrease(&self) -> Option<nat> { None }

    open spec fn peek(&self, index: int) -> Option<Self::Item> {
        Some((index % (u64::MAX as int + 1)) as u64)
    }
}
```

### 4. コンストラクタ
{: id="4-the-constructor"}

コンストラクタの事後条件は、有限の場合よりも単純です。イテレータが停止することを約束する必要が無いからです。


### 使用例 {#example-usage}

次は、新しい無限イテレータを使った小さな例です。
```rust
#[verifier::exec_allows_no_decreases_clause]
fn infinite_ctr() {
    for x in iter: IterCtr::new()
    {
        assert(x == iter.index@ % (u64::MAX as int + 1));
    }
}
```
このループは決して停止しないので、当然ながら [`exec_allows_no_decreases_clause` 属性](reference-attributes.html#verifierexec_allows_no_decreases_clause)を使う必要があります。

### 健全性 {#soundness}

無限イテレータの出力を有限の列で指定できるというのは、最初は不安に思えるかもしれません。この例で起きていることを理解する 1 つの方法は、プログラムのそれぞれの地点で、「これまでに観測された接頭部分と矛盾しないすべての列」について量化している、と考えることです。その集合は、プログラムがどれだけ長く実行されても空になりません。


## `DoubleEndedIterator` を実装する {#implementing-doubleendediterator}

この考え方をさらに広げて、無限でありながら Rust の [`DoubleEndedIterator` トレイト](https://doc.rust-lang.org/std/iter/trait.DoubleEndedIterator.html)を実装するイテレータにも対応できます。次は、`next()` が常に 42 を返し、`next_back()` が常に 43 を返す例です。1 つの長さを予言する代わりに、これから返す 42 の個数と 43 の個数を予言します。
```rust
struct Iter42_43 {
    // Number of 42s that will (prophetically) still be handed out by `next`
    front: Tracked<ProphecyGhost<nat>>,
    // Number of 43s that will (prophetically) still be handed out by `next_back`
    back: Tracked<ProphecyGhost<nat>>,
}
```

そして、この 2 つの予言を使って、`remaining` を自然な形で定義します。
```rust
spec fn seq_42_43(front: nat, back: nat) -> Seq<u32> {
    Seq::new((front + back) as nat, |i: int| if i < front { 42u32 } else { 43u32 })
}

impl IteratorSpecImpl for Iter42_43 {
    open spec fn obeys_prophetic_iter_laws(&self) -> bool { true }

    #[verifier::prophetic]
    closed spec fn remaining(&self) -> Seq<Self::Item> {
        seq_42_43(self.front@.value(), self.back@.value())
    }

    #[verifier::prophetic]
    closed spec fn produces(&self) -> Seq<bool> {
        Seq::new((self.front@.value() + self.back@.value()) as nat, |_i: int| true)
    }

    open spec fn decrease(&self) -> Option<nat> { None }

    open spec fn peek(&self, index: int) -> Option<Self::Item> { Some(42) }
}

```
その結果、`next()` と `next_back()` の実装は素直なものになります。先ほどの例にならい、古い予言変数よりも値が 1 だけ小さい新しい予言変数に入れ替えるという、同じ「技」を使います。


詳しくは[ファイルの全体](https://github.com/verus-lang/verus/blob/main/examples/guide/iterators.rs)を参照してください。
{% endraw %}

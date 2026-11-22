{% raw %}
# 有限イテレータのイテレータ仕様を実装する {#implementing-iterator-specifications-for-finite-iterators}

まずは、最もよくある種類のイテレータから始めましょう。いずれ `None` を返し、その後の `next()` の呼び出しでもずっと `None` を返し続けるイテレータです。独自の型について、[イテレータの実装を検証するために必要な手順](iterator-specs.html)を説明するために、以下の例では `Vec` がイテレータを提供していないものと仮定し、自分たちで実装することにします。

### 1. イテレータの構造体
{: id="1-the-iterator-struct"}

`VecIterator` 構造体は、元になる `Vec` への参照と、現在の位置と終端の位置を表す添字 `i` と `j` を持ちます。型不変条件（type invariant）によって、常に `i <= j <= v.len()` が成り立つことを強制します。型不変条件を使うので、`VecIterator` のフィールドは非公開のままにしておく必要があります。一方で、いくつかの仕様の中で `v` の中身に触れたいので、closed な spec 関数（`elts()`）を用意して、中身について抽象的に推論できるようにします。

```rust
pub struct VecIterator<'a, T> {
    v: &'a Vec<T>,
    i: usize,
    j: usize,
}

impl <'a, T> VecIterator<'a, T> {
    pub closed spec fn elts(self) -> Seq<T> {
        self.v@
    }

    #[verifier::type_invariant]
    pub closed spec fn vec_iterator_type_inv(self) -> bool {
        self.i <= self.j <= self.v.len()
    }
}
```

### 2. `next` メソッド
{: id="2-the-next-method"}

これは、Verus 固有の注釈を持たない、ごく普通の Rust の `Iterator` の実装です。型不変条件を使って、`Iterator::next()` に対する Verus の汎用の仕様を満たしていることを証明します。

```rust
impl<'a, T> Iterator for VecIterator<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> (ret: Option<Self::Item>) 
    {
        proof { use_type_invariant(&*self); }
        if self.i < self.j {
            let i = self.i;
            self.i = self.i + 1;
            return Some(&self.v[i]);
        } else {
            return None;
        }
    }
}
```

### 3. 仕様の実装
{: id="3-the-spec-implementation"}

Verus は `vstd` の中で `IteratorSpec` を提供しています。これは Rust の [`Iterator`](https://doc.rust-lang.org/std/iter/trait.Iterator.html) トレイトを拡張したもので、さまざまな仕様関数と、`next()` 関数に対する Verus の仕様を定義しています。独自のイテレータについて推論できるようにするには、Verus が提供する `IteratorSpecImpl` トレイトを実装する必要があります（仕様を定義している `IteratorSpec` トレイトではありません。詳しくは[「外部トレイトの仕様」](external_trait_specifications.html)を参照してください）。以下に、仕様関数を簡単にまとめます。ここでは、独自のイテレータについてそれぞれをどう定義するかに重点を置きます。

- **`obeys_prophetic_iter_laws`** — `true` を返すと、このイテレータが `next` に対する Verus の仕様を満たしていると表明したことになります。この spec 関数を設けているのは、検証されていないイテレータの実装がすべて私たちの仕様を満たしていると（健全でない形で）仮定してしまうのを避けるためです（これは `vstd` のトレイト仕様で[よく使われるパターン](external_trait_specifications.html#the-obeys_-pattern-in-vstd)です）。

  検証済みのイテレータの実装は、ここで `true` を返すべきです。また、ほとんどのイテレータアダプタは、内側のイテレータの値をそのまま返すべきです。開発者は、検証されていないイテレータ（たとえば、検証されていないクレートに由来するもの）について、これが真であると仮定することもできます。その場合、そのイテレータが (a) `IteratorSpec` の仕様に従うこと、そして (b) 常に `Some` を返すか、いずれ `None` を返してそれ以降は `None` を返し続けること、を仮定することになります。
- **`remaining`** — [予言的（prophetic）](https://verus-lang.github.io/verus/verusdoc/vstd/proph)な spec 関数で、`next()` を呼び出すたびにイテレータがいずれ生成しうる要素の列を返します。`VecIterator` の場合、これは部分範囲 `v[i..j]` です。`remaining` は `Seq<Self::Item>` を返すことに注意してください。そのため、`VecIterator` の `Item` は `&T` なので、その `remaining` 関数は `Seq<&T>` を返します。`vstd` の列ライブラリには、`Seq<T>` を `Seq<&T>` に変換する便利な関数 `as_ref` と、逆向きの変換を行う `unref` が用意されています。
- **`produces`** — `remaining()` に含まれるそれぞれの要素の候補について、その要素が実際に返されるのか、それとも、たとえばクライアントが発散するクロージャを渡したなどの理由で、イテレータがその要素を生成しようとして「行き詰まる」のかを表します。この例では行き詰まることがないので、すべての要素が true である列を返します。
- **`decrease`** — Verus の [decreases チェッカー](recursion.html)のための停止性の尺度です。既定では、`for` ループはこれが `Some(n)` を返し、`n` が `next` の呼び出しのたびに減少することを期待します。ここでは `j - i` でうまくいきます。無限イテレータは `None` を返すべきです。
- **`peek`** — 指定した先読みの添字にある要素を、あれば返します。これを用意しておくと、Verus が反復中の現在の要素について推論しやすくなります。`peek` は予言的では**ない**ので、`remaining()` を使って定義することはできない点に注意してください。

まとめると、`VecIterator` に対するこれらの仕様の実装は次のようになります。
```rust
impl<'a, T> IteratorSpecImpl for VecIterator<'a, T> {

    open spec fn obeys_prophetic_iter_laws(&self) -> bool {
        true
    }

    closed spec fn remaining(&self) -> Seq<Self::Item> {
        self.v@[self.i..self.j].as_ref()
    }

    closed spec fn produces(&self) -> Seq<bool> {
        Seq::new(self.elts_remaining().len(), |_i: int| true)
    }

    closed spec fn decrease(&self) -> Option<nat> {
        Some((self.j - self.i) as nat)
    }
    
    open spec fn peek(&self, index: int) -> Option<Self::Item> {
        if 0 <= index < self.elts().len() {
            Some(&self.elts()[index])
        } else {
            None
        }
    }
}
```

### 4. コンストラクタ
{: id="4-the-constructor"}

ほとんどのイテレータ型は、何かほかの型から構築する必要があります。この例では、コンストラクタ `vec_iter` が `&'a Vec<T>` を受け取り、`VecIterator<'a, T>` を返します。通常は、以下に示すような事後条件を付けることになるでしょう。1 つ目は、イテレータの予言的な列を、構築の元になった値（この場合は `Vec<T>` の要素）に結び付けます。2 つ目は、予言的な列をイテレータの抽象的な `elts()` に結び付けます。peek は（`remaining()` ではなく）`elts()` を使って定義されているので、この結び付きが必要です。最後の事後条件によって、`for` ループが停止性を自動的に証明できるようになります。

```rust
pub fn vec_iter<'a, T>(v: &'a Vec<T>) -> (iter: VecIterator<'a, T>)
    ensures 
        IteratorSpec::remaining(&iter) == v@.as_ref(),
        IteratorSpec::remaining(&iter).unref() == iter.elts(),
        IteratorSpec::decrease(&iter) is Some,
{
    VecIterator { v: v, i: 0, j: v.len() }
}
```

### 5. `DoubleEndedIterator` を実装する
{: id="5-implementing-doubleendediterator"}

イテレータが逆向きの走査に対応している場合は、Rust の標準の [`DoubleEndedIterator` トレイト](https://doc.rust-lang.org/std/iter/trait.DoubleEndedIterator.html)を実装します。このトレイトは `next_back` メソッドを追加します。

```rust
impl<'a, T> DoubleEndedIterator for VecIterator<'a, T> {
    fn next_back(&mut self) -> (ret: Option<Self::Item>) {
        proof { use_type_invariant(&*self); }
        if self.i < self.j {
            self.j = self.j - 1;
            return Some(&self.v[self.j]);
        } else {
            return None;
        }
    }
}
```

`.rev()` について推論できるようにするには、（`IteratorSpecImpl` と同じ要領で）`DoubleEndedIteratorSpecImpl` も実装し、後ろから数えて指定した添字にある要素を返す `peek_back` 関数を用意する必要があります。これが無いと、Verus は、逆順にしたイテレータがどの要素を生成するのかを知ることができません。

```rust
impl<'a, T> DoubleEndedIteratorSpecImpl for VecIterator<'a, T> {
    open spec fn peek_back(&self, index: int) -> Option<Self::Item> {
        let len = self.elts().len();
        if 0 <= index < len {
            Some(&self.elts()[len - index - 1])
        } else {
            None
        }
    }    
}
```
{% endraw %}

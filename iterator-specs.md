{% raw %}
# 独自の型のためのイテレータの仕様 {#iterator-specifications-for-a-custom-type}

独自の型に対する反復について推論するには、次のことが必要です。
1. イテレータの構造体を定義します。これは、Rust のさまざまな型にそれぞれ対応するイテレータ型があるのと同様です。たとえば、Rust のスライスは [`std::slice::Iter`](https://doc.rust-lang.org/std/slice/struct.Iter.html) 構造体に頼っています。
2. 標準の [Rust の `Iterator` トレイト](https://doc.rust-lang.org/std/iter/trait.Iterator.html)（具体的には、その `next()` 関数）を実装します。
3. `IteratorSpecImpl` トレイトを実装して、イテレータに Verus の仕様を与えます。
4. 型のコンストラクタ関数を書き、構築されたイテレータについての有用な事後条件をいくつか与えます。

以降の節では、有限のイテレータ（いずれ `None` を返すもの）と無限のイテレータ（常に `Some` を返すもの）について、この手順の進め方を説明します。現在のところ、Verus は、`None` を返したあとで再び `Some` を返し始めるイテレータの検証には対応していません。
{% endraw %}

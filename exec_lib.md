{% raw %}
# 実行可能ライブラリ: Vec {#executable-libraries-vec}

前の節では、数学的なコレクション型である
`Seq`、`Set`、`Map` を説明しました。
この節では、`Seq` の実行可能な実装である `Vec` を説明します。
Verus は、Rust の `std::vec::Vec` 型の機能の一部に対応しています。
`Vec` を使うには、コードに `use std::vec::Vec;` を書きます。

`Vec::new` で `Vec` を確保し、そこに要素を push できます。

```rust
fn test_vec1() {
    let mut v: Vec<u32> = Vec::new();
    v.push(0);
    v.push(10);
    v.push(20);
    v.push(30);
    v.push(40);
    assert(v.len() == 5);
    assert(v[2] == 20);
    assert(v[3] == 30);
    v.set(2, 21);
    assert(v[2] == 21);
    assert(v[3] == 30);
}
```

上のコードは、`Vec` の値 `v` について直接、表明を行えています。
`Vec` の値について、もっと複雑な仕様や証明を書くこともできます。
しかし一般には、Verus はプログラマに対して、
仕様や証明を `Vec` のような特定の具体的なデータ型に固定してしまうのではなく、
`Seq`、`Set`、`Map` のような数学的な型について `spec` 関数と
`proof` 関数を書くことを勧めています。
こうすると、`spec` 関数と `proof` 関数は、
`Seq`、`Set`、`Map`、`int`、`nat` のような数学的な型で書かれた
本質的な考え方に集中できます。
`usize` のような有限幅の整数をいじり回したり、
算術オーバーフローを心配したりしなくて済みます。

もちろん、数学的な型と具体的な型の間にはつながりが必要です。
`exec` 関数の仕様は、数学的な抽象と具体的な現実の間を
行き来しなければならないことがよくあります。
これを楽にするため、Verus は、具体的な型から数学的な `view` を取り出す
糖衣構文 `@` に対応しています。
たとえば `v@` は、ベクタ `v` のすべての要素からなる `Seq` を返します。

```rust
spec fn has_five_sorted_numbers(s: Seq<u32>) -> bool {
    s.len() == 5 && s[0] <= s[1] <= s[2] <= s[3] <= s[4]
}

fn test_vec2() {
    let mut v: Vec<u32> = Vec::new();
    v.push(0);
    v.push(10);
    v.push(20);
    v.push(30);
    v.push(40);
    v.set(2, 21);
    assert(v@ =~= seq![0, 10, 21, 30, 40]);
    assert(v@ =~= seq![0, 10] + seq![21] + seq![30, 40]);
    assert(v@[2] == 21);
    assert(v@[3] == 30);
    assert(v@[2..4] =~= seq![21, 30]);
    assert(has_five_sorted_numbers(v@));
}
```

`Vec` の `Seq` としての view を使うと、
`Vec` の中身についての仕様を書くときに、
連結や部分列といった `Seq` のさまざまな機能を使えます。

Verus の `std::vec::Vec` への対応は、現在も拡張が続いています。最新の
ドキュメントは[このリンク](https://verus-lang.github.io/verus/verusdoc/vstd/std_specs/vec/index.html)を参照してください。
なお、vstd はこれらを、通常の `std::vec::Vec` の関数に対する[仕様](./reference-assume-specification.html)として
提供しています。そのため、Verus で書いたコードは、標準の
`Vec` のメソッド（たとえば `v.insert(i, x)`）をそのまま直接呼び出します。

`std::vec::Vec` の機能のドキュメントは[こちら](https://doc.rust-lang.org/std/vec/struct.Vec.html)にあります。
{% endraw %}

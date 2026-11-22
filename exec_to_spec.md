{% raw %}
# 実行可能コードに対する spec 関数を自動的に作る {#automatically-producing-a-spec-function-for-executable-code}

Verus でコードを検証していると、対応する実行可能な関数とまったく同じ実装を持つ spec 関数を書かなければならないことがあります。
実行可能な関数が小さく、純粋に計算だけを行うものである場合に、この状況はよく起こります。

`#[verus_verify(dual_spec)]` は、既存の実行可能な関数に対する spec 関数を書く作業を簡単にします。
実行可能な関数に適用すると、次のことを行って、対応する spec 関数を自動的に作ります。

1. 実行可能な関数から、ゴースト変数と証明ブロックを取り除きます。
2. 同一のロジックを持つ spec 関数を生成します。既定では、spec 関数には _VERUS_SPEC_xxx のような内部的な名前が付きます。`#[verus_verify(dual_spec($custom_name))]` を使って、独自の名前を指定することもできます。
3. 実行可能な関数に、属性 [`#[when_used_as_spec]`](reference-attributes.html) を適用します。`when_used_as_spec` 属性によって、証明の中では実行可能な関数の名前をそのまま spec 関数として使えるので、実際の spec 関数の名前は、たいていの場合は重要ではありません。

生成される spec 関数は、既定では `closed` です。
公開モードと不透明性は、`verus_verify` の追加の引数で制御できます。
`#[verus_verify(dual_spec, open)]`、`#[verus_verify(dual_spec, closed)]`、`#[verus_verify(dual_spec, opaque)]` は、生成される spec 関数をそれぞれ [`open`、`closed`](spec_functions.html)、[`opaque`](reference-attributes.html) として指定します。

例を示します。

```rust
#[verus_verify(dual_spec, closed)]
#[verus_spec(
    requires
        x < 100,
        y < 100,
    returns f(x, y)
)]
pub fn f(x: u32, y: u32) -> u32 {
    proof!{
        assert(true);
    }
    {
        proof!{assert(true);}
        x + y
    }
}

#[verus_verify(dual_spec, open)]
#[verus_spec(
    requires
        x < 100,
    returns
        f2(x),
)]
pub fn f2(x: u32) -> u32 {
    f(x, 1)
}
```

## 制限 {#limitations}

`#[verus_verify(dual_spec)]` を使うには、`#[verus_spec(...)]` を使う必要があります。
現時点では、`verus!` マクロで検証された実行可能な関数には対応していません。
`verus_verify` マクロは、`verus!` の構文ではなく、Rust 本来の構文で書かれた exec 関数を構文解析することを前提にしているからです。
したがって、`#[verus_verify(dual_spec)]` は、`verus!` マクロの外で `#[verus_spec(...)]` とともに使ってください。

`dual_spec` は exec 関数から spec 関数を生成しようとしますが、exec 関数が spec 関数の対応していない機能を使っている場合は、有効な spec 関数を生成できないことがあります。

たとえば、可変な入力には対応していません。

```
fn f(x: &mut u32, y: u32) -> u32 {
    *x = *x + y;
    *x
}
```

対応していない機能を使うと、次のエラーメッセージが表示されるはずです。

```
The verifier does not yet support the following Rust feature
```


## 関連 {#related}

`dual_spec` 属性は実行可能コードに対する spec 関数を作りますが、これではあなたの用途に合わないかもしれません。
たとえば、[spec から exec コードへ](exec_spec.html)変換したい場合や、[`when_used_as_spec` 属性](reference-attributes.html)を使いたい場合があります。
{% endraw %}

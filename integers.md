{% raw %}
# 整数型 {#integer-types}

Rust は、さまざまな[固定ビット幅の整数型](https://doc.rust-lang.org/book/ch03-02-data-types.html#integer-types)に対応しています。

- `i8`, `i16`, `i32`, `i64`, `i128`, `isize`
- `u8`, `u16`, `u32`, `u64`, `u128`, `usize`

Verus はこれらに加えて、仕様の中で任意の大きさの整数を表すための整数型を 2 つ追加しています。

- int
- nat

`int` 型は、Verus で整数演算について推論するための最も基本的な型です。
この型は、正と負の両方を含む[すべての数学的な整数](https://en.wikipedia.org/wiki/Integer)を表します。
SMT ソルバーには、`int` 型の値について推論するための直接の支援が備わっています。

内部的には、Verus はほかの整数型を `int` で表現し、整数の範囲を制限する数学的な制約を加えています。
たとえば、[自然数](https://en.wikipedia.org/wiki/Natural_number)の型 `nat` の値は、`0` 以上であるという制約の付いた数学的な整数です。
Rust の固定ビット幅の整数型には、下限と上限の両方があります。
`u8` の値は、`0` 以上かつ 256 未満であるという制約の付いた整数です。

```rust
fn test_u8(u: u8) {
    assert(0 <= u < 256);
}
```

`usize` と `isize` の範囲は、プラットフォームに依存します。
既定では、Verus はこれらの型の幅が 32 ビットか 64 ビットのどちらかであり得ると仮定しますが、[この仮定は設定で変えられます](./reference-global.html#with-usize-and-isize)。
Verus は、定数
[`usize::BITS`](https://doc.rust-lang.org/std/primitive.usize.html#associatedconstant.BITS)、
[`usize::MAX`](https://doc.rust-lang.org/std/primitive.usize.html#associatedconstant.MAX)、
[`isize::MAX`](https://doc.rust-lang.org/std/primitive.isize.html#associatedconstant.MAX)、
[`isize::MIN`](https://doc.rust-lang.org/std/primitive.isize.html#associatedconstant.MIN)
を認識します。
これらは、`usize` の整数の範囲について記号的に推論するのに役立ちます。

## 仕様で整数型を使う {#using-integer-types-in-specifications}

整数型は 14 種類もあるので（`int`、`nat`、`u8`...`usize`、`i8`...`isize` を数えた場合）、仕様を書くときにどの型を使うべきかは、いつも明らかとは限りません。
私たちの助言は、既定ではできるだけ一般的にしておくことです。
- 既定では `int` を使います。これが最も一般的な型であり、SMT ソルバーが最も効率よく扱える型だからです。
  - 例: Verus の[列（sequence）ライブラリ](https://github.com/verus-lang/verus/blob/main/source/vstd/seq.rs)は、列への添字アクセスなど、ほとんどの操作に `int` を使っています。
  - 注意: 後で説明するように、仕様の中のほとんどの算術演算は `int` 型の値を生み出します。
    そのため、仕様は `int` を使って書くのがたいてい最も便利です。
- 戻り値やデータ型のフィールドのうち、下限が 0 であることが有用な情報になりそうなものには、`nat` を使います。
  列の長さなどがそうです。
  - 例: Verus の [`Seq::len()` 関数](https://github.com/verus-lang/verus/blob/main/source/vstd/seq.rs)は、列の長さを表すために `nat` を返します。
  - `nat` 型は、再帰的な定義が停止することを証明するのにも便利です。
    たとえば、負の整数に対する `factorial` の定義を与えたくないなら、再帰的な `factorial` 関数を、`nat` 型の引数を取るように定義するとよいでしょう。
- バイトのような固定幅の値には、固定幅の整数型を使います。
  - 例: ネットワークパケットのバイト列は、`Seq<u8>` 型で表せます。これは、8 ビットの値からなる任意の長さの列です。

`int` と `nat` はゴーストコードの中でしか使えないことに注意してください。
これらは実行可能コードにコンパイルできません。
たとえば、次のコードはうまくいきません。

```rust
fn main() {
    let i: int = 5; // FAILS: executable variable `i` cannot have type `int`, which is ghost-only
}
```

## 整数定数 {#integer-constants}

通常の Rust と同じように、Verus の整数定数には、定数の型を正確に指定するために、型を接尾辞として付けられます
（たとえば `7u8`、`7u32`、`7int`）。

```rust
fn test_consts() {
    let u: u8 = 1u8;
    assert({
        let i: int = 2int;
        let n: nat = 3nat;
        0int <= u < i < n < 4int
    });
}
```

常にではありませんが、たいていの場合、Verus と Rust は整数定数の型を推論できます。
そのため、Rust の型検査器が型を推論できないと文句を言わないかぎり、接尾辞は省略できます。

```rust
fn test_consts_infer() {
    let u: u8 = 1;
    assert({
        let i: int = 2;
        let n: nat = 3;
        0 <= u < i < n < 4
    });
}
```

式 `0 <= u < i < n < 4` の中の値 `0`、`u`、`i`、`n`、`4` は、すべて異なる型を持っていてもよいことに注意してください。
ゴーストコードの中では、`<=`、`<`、`>=`、`>`、`==`、`!=` を使って、異なる整数型の値を比較できます
（たとえば `u < i` では、`u8` と `int` を比較しています）。

接尾辞 `int` や `nat` の付いた定数は、任意の大きさにできます。

```rust
fn test_consts_large() {
    assert({
        let i: int = 0x10000000000000000000000000000000000000000000000000000000000000000int;
        let j: int = i + i;
        j == 2 * i
    });
}
```

## 「as」を使った整数の型変換 {#integer-coercions-using-as}

通常の Rust と同じように、`as` 演算子は、ある整数型を別の整数型に変換（coerce）します。
ゴーストコードでは、`as int` や `as nat` を使って `int` や `nat` に変換できます。

```rust
fn test_coerce() {
    let u: u8 = 1;
    assert({
        let i: int = u as int;
        let n: nat = u as nat;
        u == i && u == n
    });
}
```

値 `v` が小さすぎたり大きすぎたりして型 `t` に収まらない場合でも、`as` を使って `v` を `t` に変換できます。
ただし、値 `v` が型 `t` の範囲の外にある場合、式 `v as t` は、型 `t` の何らかの任意の値を生み出します。

```rust
fn test_coerce_fail() {
    let v: u16 = 257;
    let u: u8 = v as u8;
    assert(u == v); // FAILS, because u has type u8 and therefore cannot be equal to 257
}
```

これは、アサーションについてのエラーを生じます。あわせて、`as` による変換の値が範囲外だった可能性があるという手がかりも表示されます。

```
error: assertion failed
   |
   |     assert(u == v); // FAILS, because u has type u8 and therefore cannot be equal to 257
   |            ^^^^^^ assertion failed

note: recommendation not met: value may be out of range of the target type (use `#[verifier::truncate]` on the cast to silence this warning)
   |
   |     let u: u8 = v as u8;
   |                 ^
```

Verus が as による切り捨てをどう定義しているか、そしてそれについてどう推論するかの詳細は、[リファレンス](./reference-as.html)を参照してください。

## 整数演算 {#integer-arithmetic}

整数演算は、ゴーストコードと実行可能コードとで、少し違った振る舞いをします。

**実行可能**コードでは、整数のオーバーフローについて推論しなければならないことがよくあります。
実際、Verus は、オーバーフローが起きないことの証明を要求します。
次の演算は失敗します。演算の結果が 255 より大きな値になるかもしれないからです。

```rust
fn test_sum(x: u8, y: u8) {
    let sum1: u8 = x + y; // FAILS: possible overflow
}
```

```
error: possible arithmetic underflow/overflow
   |
   |     let sum1: u8 = x + y; // FAILS: possible overflow
   |                    ^^^^^
```

一方、**ゴースト**コードでは、よく使う算術演算（`+`、`-`、`*`、`/`、`%`）は、決してオーバーフローも折り返し（wrap）もしません。
これを可能にするために、Verus は多くの演算の結果の型を広げます。
たとえば、2 つの `u8` の値の加算は、`int` 型に広げられます。

```rust
fn test_sum2(x: u8, y: u8) {
    assert({
        let sum2: int = x + y;  // in ghost code, + returns int and does not overflow
        0 <= sum2 < 511
    });
}
```

ゴーストコードでは `+` がオーバーフローしないので、オーバーフロー*についての*仕様を簡単に書けます。
たとえば、実行可能な `x + y` がオーバーフローしないようにするには、単に `requires x + y < 256` と書きます。
これは、`requires` 節の中で `x + y` が `int` 型に広げられることを利用しています。

```rust
fn test_sum3(x: u8, y: u8)
    requires
        x + y < 256,  // make sure "let sum1: u8 = x + y" can't overflow
{
    let sum1: u8 = x + y;  // succeeds
}
```

また、入力が同じ型である必要はないことにも注意してください。
ある整数型と別の整数型とを、加算、減算、乗算できます。

```rust
fn test_sum_mixed(x: u8, y: u16) {
    assert(x + y >= y);  // x + y has type int, so the assertion succeeds
    assert(x - y <= x);  // x - y has type int, so the assertion succeeds
}
```

一般に、ゴーストコードでは、Verus は、`+`、`-`、`*` のようにオーバーフローするかもしれない演算子について、Rust 本来の整数型を `int` に広げます。
型を広げる規則の詳細は、[リファレンスのページ](./spec-arithmetic.html)で説明しています。

覚えておくとよい点を、さらにいくつか挙げます。

 * ゴーストコードでは、`/` と `%` は、左辺または右辺が負の場合に、Rust の切り捨て除算と剰余ではなく、
    [ユークリッド除算と剰余](https://en.wikipedia.org/wiki/Euclidean_division)を計算します。
 * 0 による除算と 0 による剰余は、実行可能コードではエラーであり、ゴーストコードでは未規定（unspecified）です
   （詳しくは[ゴーストコードと exec コード](./ghost_vs_exec.html)を参照してください）。
 * 名前の付いた算術関数 `add(x, y)`、`sub(x, y)`、`mul(x, y)` は、型を広げません。
    そのため、ゴーストコードの中でも、切り捨ての振る舞いをします。
    Verus は、[`wrapping_add`](https://doc.rust-lang.org/std/primitive.u32.html#method.wrapping_add)
    や [`checked_add`](https://doc.rust-lang.org/std/primitive.u32.html#method.checked_add) のような Rust の関数もいくつか認識します。
    これらは、実行可能コードとゴーストコードのどちらでも使えます。
{% endraw %}

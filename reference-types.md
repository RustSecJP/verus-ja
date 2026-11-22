{% raw %}
# 型の数学的な解釈 {#mathematical-interpretations-of-types}

仕様コードにおける型（Rust の一般的な型でも Verus 固有の型でも）の意味を定義するために、それぞれの型に*数学的な領域*（mathematical domain）を割り当てます。
場合によっては、その領域をさらに制限する*妥当性述語*（validity predicate）も定義します。
そのうえで、仕様の式の意味論を、数学的な領域を使って定義します。

## 整数型 {#integer-types}

Verus の整数型には、Rust 標準のプリミティブ型（`u8`、`u16`、`u32`、`u64`、`u128`、`usize`、`i8`、`i16`、`i32`、`i64`、`i128`、`isize`）と、Verus の組み込み型である `int` と `nat` があります。

**数学的な領域。**
それぞれの整数型の数学的な領域は、整数の集合 ℤ です。
一部の型には、範囲を制限する妥当性述語が追加されています。

| 型      | 範囲  |
|---------|-------|
| `int`   | `(-∞, ∞)`（つまり、制限なし） |
| `nat`   | `[0, ∞)` |
| `uN`    | <code>&#x5b;0, 2<sup>N</sup>)</code> |
| `iN`    | <code>&#x5b;-2<sup>N-1</sup>, 2<sup>N-1</sup>)</code> |
| `usize` | <code>&#x5b;0, 2<sup>usize::BITS</sup>)</code> |
| `isize` | <code>&#x5b;-2<sup>isize::BITS-1</sup>, 2<sup>isize::BITS-1</sup>)</code>  |

`isize` と `usize` について、Verus がプラットフォームのワードサイズ（`usize::BITS`、あるいは同じことですが `isize::BITS`）に置く仮定は、[`global` ディレクティブ](./reference-global.html#with-usize-and-isize)で設定できます。

**spec 等価性。** 整数に対する spec 等価性は、整数としての等しさで定義されます。

**リテラル式**
spec コードでは、標準の型（`u8`、`i8` など）と Verus 固有の型（`nat` と `int`）の両方について、整数リテラルを書けます。
型注釈の無いリテラルでは型が推論されますが、型を明示することもできます（たとえば `1u8` や `1nat`）。

**spec 演算子**

* 整数型どうしは、[`as` キャスト](./reference-as.html)で相互に変換できます。
* Verus は[標準の算術演算子](./spec-arithmetic.html)に対応しています。
* Verus は[ビット演算子](./spec-bit-ops.html)に対応しています。

## Bool {#bool}

**数学的な領域。**
`bool` の数学的な領域は、離散的な真偽値の集合 𝔹 = `{true, false}` です。

**spec 等価性。** `bool` に対する spec 等価性は、標準的な真偽値の同値で定義されます。

**リテラル式。** `true` と `false` はどちらも有効なリテラルです。

**spec 演算子。**
標準の `&&` と `||` のほかに、Verus は論理学で標準的な真偽値の演算子にもいくつか対応しています。たとえば、[`含意と同値`](./reference-implication.html)（`==>`、`<==`、`<==>`）や、連言と選言の[優先順位の低い変種](./prefix-and-or.html)です。


## Char {#char}

**数学的な領域。**
`char` は、数学的な領域が ℤ で、専用の妥当な範囲を持つ[整数型](#integer-types)として解釈されます。

| 型      | 範囲  |
|---------|-------|
| `char`  | `[0, 0xD7FF] ∪ [0xE000, 0x10FFFF]` |

これは、「Unicode スカラー値」としての [Rust における `char` の定義](https://doc.rust-lang.org/std/primitive.char.html)と一致しています。

> char は「[Unicode スカラー値](https://www.unicode.org/glossary/#unicode_scalar_value)」であり、これは[サロゲートコードポイント](https://www.unicode.org/glossary/#surrogate_code_point)以外の任意の「[Unicode コードポイント](https://www.unicode.org/glossary/#code_point)」です。これには固定の数値的な定義があります。コードポイントは 0 から 0x10FFFF まで（両端を含む）の範囲にあります。UTF-16 で使われるサロゲートコードポイントは、0xD800 から 0xDFFF までの範囲にあります。

**spec 等価性。** `char` の spec 等価性は、整数としての等しさで定義されます。

**リテラル式。** spec コードでは、標準の [`char` リテラル](https://doc.rust-lang.org/reference/tokens.html#character-literals)を書けます。

**spec 演算子。** spec コードでは、`char` は [`as` キャスト](./reference-as.html)を使って整数型と相互に変換できます。
これは exec コードよりも寛容です。exec コードでは、こうした型強制（coercion）の多くが許されていません。
ほかの型強制と同じく、変換される整数が変換先の範囲に収まらない場合、結果は未定義になることがあります。

## Box（`Box<T>`） {#box-boxt}

**数学的な領域。**
`Box<T>` は、追加のメタデータを持たず、`T` とまったく同じに解釈されます。

**spec 等価性。**
`Box<T>` に対する spec 等価性は、`T` に対する spec 等価性を引き継ぎます。

**spec 演算子。**
ボックス化の操作 `Box::new` とボックスを外す操作（`*x`）は、意味論上は恒等関数です。

## 共有参照（`&T`） {#shared-reference-t}

**数学的な解釈。**
`&T` は、追加のメタデータを持たず、`T` とまったく同じに解釈されます。

**spec 等価性。**
`&T` に対する spec 等価性は、`T` に対する spec 等価性を引き継ぎます。

**spec 演算子。**
借用の操作 `&x` と参照外しの操作（`*x`）は、意味論上は恒等関数です。

## 可変参照（`&mut T`） {#mutable-reference-mut-t}

可変参照に対する私たちのエンコーディングは、[RustHorn のエンコーディング](https://dl.acm.org/doi/10.1145/3462205)に基づいています。
可変参照の使い方についての実用的な情報は、[ガイドのページ](./mutable-references.html)を参照してください。

> [!NOTE]
> 2026-05-03 のリリースより前の Verus は、可変参照をまったく違う方法で扱っていました。
> 移行についての詳しい情報は、[移行ガイド](https://github.com/verus-lang/verus/blob/main/source/docs/migration-mut-ref.md)を参照してください。


**数学的な解釈。**
`&mut T` は抽象型として解釈されます。
全射関数 `(&mut T) -> (T, *mut T, BorrowId)` と両立することが保証されています。
現時点では、ポインターや「BorrowId」について明示的に推論するための組み込みの機能はありません。

**spec 等価性。**
可変参照には外延的な等価性の概念が無く、現時点では、基本的な反射律による以外に等しさを証明する方法がありません。

つまり、2 つの可変参照の現在の値、最終的な値、アドレスが同じであっても、それらが等しいとは限りません。
とくに、再借用（reborrow）は、その借用元である可変借用と等しくありません。
2 つの参照が等しいと証明できるためには、それらが同じ借用命令に由来していなければなりません。

```rust
fn example() {
    let mut a = 0;
    let mut b = 0;

    let a_ref1 = &mut a;
    let a_ref2 = &mut a;

    assert(a_ref1 == a_ref1); // ok, basic reflexivity
    assert(a_ref1 == a_ref2); // not ok; these originate from different borrows
}
```

しかし、2 つの可変参照が同じ借用に由来しているだけでは、まだ十分ではありません。
異なる時点で取った 2 つのスナップショットは、そのあいだに可変参照が更新されていれば、等しくないと判定されます。

```rust
fn example2() {
    let mut a = 0;
    let a_ref = &mut a;

    let ghost snapshot1 = a_ref;
    let ghost snapshot2 = a_ref;

    *a_ref = 30;

    let ghost snapshot3 = a_ref;

    assert(snapshot1 == snapshot2); // ok, a_ref is unchanged between the two snapshots
    assert(snapshot1 == snapshot3); // not ok, a_ref was changed between the two snapshots
}
```

> [!DIFF]
> spec コードにおける可変参照の等価性は、exec コードにおける可変参照の等価性と同じではありません。exec コードでは、可変参照の等価性は、参照先の値の等価性で定義されます。

**spec 演算子。**
spec コードでは、可変参照 `x: &mut T` について、2 つの重要な値を通して推論します。

 * `*x`: 可変参照の先にある現在の値。
 * `*final(x)`: 可変参照が失効するときに指している最終的な値。
   この値は[予言的（prophetic）](./reference-attributes.html#verifierprophetic)であるとみなされます。

> [!NOTE]
> 事後条件では、`x: &mut T` が関数の入力である場合に、`*x` の代わりに `*old(x)` と書くよう求められることがあります。
> これは形式的なレベルでは意味を持ちません。
> `*x` がどの値を指すのかについて、ユーザーが混乱するのを防ぐためだけに使われます。

**補足。**

> [!NOTE]
> Verus は、spec モードでの可変参照の使用に対して「暗黙の再借用」を挿入しません。

> [!NOTE]
> spec のスナップショットを通して、`final` 演算子により、自己参照的な可変参照を作ることができます。
>
> ```rust
> struct Rec {
>     g: Option<Ghost<&'static mut Rec>>,
> }
>
> fn test() {
>     let mut r = Rec { g: None };
>
>     let mut r_ref = &mut r;
>     *r_ref = Rec { g: Some(Ghost(r_ref)) };
>
>     assert(r.g.is_some());
>     assert(r == *final(r.g.unwrap()@)); // cycle
> }
> ```
>
> そのため、Verus は [decreases-to](./reference-decreases-to.html) の目的では、<code>*final(x)</code> を `x` の部分フィールドとして扱いません。つまり、`final` の値を通した再帰は、整礎な再帰とはみなされません。

## 生ポインター（`*const T` と `*mut T`） {#raw-pointer-const-t-and-mut-t}

このページでは、生ポインターの型だけを扱います。
ポインターについて推論するための実用的な情報は、vstd ライブラリの [`raw_ptr`](https://verus-lang.github.io/verus/verusdoc/vstd/raw_ptr/index.html) を参照してください。

**数学的な解釈。**
spec コードでは、ポインター `ptr: *mut T` または `ptr: *const T` は、次の 3 つのデータの組として定義されます。

 * `usize` 型の `ptr.addr()`: ポインターのアドレス
 * [`<T as Pointee>::Metadata`](https://doc.rust-lang.org/std/ptr/trait.Pointee.html) 型の `ptr@.metadata`
 * 抽象型 [`Provenance`](https://verus-lang.github.io/verus/verusdoc/vstd/raw_ptr/struct.Provenance.html) の `ptr@.provenance`

> [!NOTE]
> Verus は `*const T` と `*mut T` を同一に扱います。
> 両者は[変性（variance）](https://doc.rust-lang.org/reference/subtyping.html?#variance)が異なりますが、そのことは spec レベルでの数学的な解釈に影響しません。
> `*const T` から `*mut T` へのキャストも、その逆も、意味論上は恒等関数です。

**spec 等価性。**
2 つのポインターは、アドレス、メタデータ、プロベナンス（provenance）が等しければ等しくなります。

> [!DIFF]
> spec コードにおけるポインターの等価性は、exec コードにおけるポインターの等価性と同じではありません。
> exec での比較はアドレスとメタデータだけを比べますが、spec での比較はプロベナンスも比べます。

**spec 演算子。**

 * `ptr.addr()`、`ptr@.metadata`、`ptr@.provenance` を使うと、組を構成する各部分を取り出せます。
 * [`ptr_from_data`](https://verus-lang.github.io/verus/verusdoc/vstd/raw_ptr/fn.ptr_from_data.html) を使うと、構成部分からポインターを組み立てられます。

## `String` と `str` {#string-and-str}

**数学的な解釈。**
`String` と `str` は、それぞれ抽象型として表現されます。
この型は、<code>Seq&lt;<a href="#char">char</a>&gt;</code> への全射である `view` 関数と整合することが保証されています。

> [!NOTE]
> Rust では、たとえば [`from_utf8_unchecked`](https://doc.rust-lang.org/std/string/struct.String.html#method.from_utf8_unchecked) や [`as_bytes_mut`](https://doc.rust-lang.org/std/string/struct.String.html#method.as_bytes_mut) を使って、バイト列が有効な UTF-8 エンコーディングになっていない `str` を得ることができます。ただし、そうした関数は unsafe であり、不正な形式の文字列を作ると簡単に未定義動作（UB）につながります。Verus は現時点では不正な形式の文字列に対応しておらず、`String`/`str` の値は有効な UTF-8 を含むと仮定されます。

**spec 等価性。**
`String` と `str` は外延的等価性を示しません。等価性について推論するときは、必ず `view` 関数を使ってください。

**リテラル式。**
Verus は、spec コードでの[文字列リテラル式](https://doc.rust-lang.org/reference/tokens.html#grammar-STRING_LITERAL)に対応しています。

> [!NOTE]
> 文字列リテラルの中身は、既定では証明器にとって不透明です。文字列リテラルの中身を見えるようにするには、[`reveal_strlit`](./reference-reveal-strlit.html) を使ってください。

**spec 演算子。**

文字列に対する主な spec 演算子は、「ビュー」演算子の `s@` または `s.view()` です。
{% endraw %}

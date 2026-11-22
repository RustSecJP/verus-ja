{% raw %}
# 外部トレイトの仕様 {#external-trait-specifications}

外部の Rust ライブラリとやり取りする検証済みのコードを書くとき、
それらのライブラリで定義されたトレイトに仕様を追加する必要が出てくることがあります。
Verus は、このための属性を 2 つ用意しています。

 * `#[verifier::external_trait_specification]` — トレイトのメソッドに requires / ensures を追加します
 * `#[verifier::external_trait_extension]` — それに加えて、トレイトに spec の補助関数を定義します

## 健全性についての警告 {#soundness-warning}

> [!CAUTION]
> 外部トレイトの仕様は検査されないため、Verus の保証を簡単にすり抜けられてしまいます。
>
> さらに、トレイトの仕様を正しく書くことは、通常の関数の仕様を書くことよりも難しい場合が少なくありません。
>
> そのトレイトのすべての実装は、検証されていないコードの中の実装も、まだ書かれていないコードの実装も含めて、
> 仕様を守っていると仮定されます。たとえば、`pub fn test<A: Formatter>(...)` を持つクレートを検証すると、
> Verus は、`A` に具体化されるどんな型も `Formatter` の仕様を満たすと仮定します。
> その型が検証されていないクレートに由来するものであっても、そう仮定します。
> これは、現在の検証されていないコードと、将来の検証されていないコードの両方に対する契約です。
>
> この健全性のリスクを和らげるのに役立つパターン（`vstd` が採用しているもの）については、[下の節](#the-obeys_-pattern-in-vstd)を参照してください。

## 基本的な外部トレイトの仕様 {#basic-external-trait-specification}

次のような外部トレイトがあるとします。

```rust
#[verifier::external]
trait Encoder {
    fn encode_value(&self, x: u64) -> u64;
}
```

これには、`#[verifier::external_trait_specification]` を使って仕様を追加できます。

```rust
#[verifier::external_trait_specification]
trait ExEncoder {
    // This associated type names the trait being specified:
    type ExternalTraitSpecificationFor: Encoder;

    fn encode_value(&self, x: u64) -> (result: u64)
        ensures
            result >= x;
}
```

この構文についての要点は、次のとおりです。
 * 仕様のトレイト（ここでは `ExEncoder`）には、`ExternalTraitSpecificationFor` という特別な名前の関連型が
   含まれていなければなりません。その関連型の境界が、仕様を与える対象の外部トレイトを指します。
 * トレイト名 `ExEncoder` は任意で、ほかの場所では使われません。
 * メソッドのシグネチャは外部トレイトと一致していなければなりませんが、`requires` 節と `ensures` 節を追加できます。
   また、`ensures` で使うために、戻り値に名前を付けられます（例: `(result: u64)`）。
 * 仕様のトレイトに、外部トレイトのメンバーをすべて含める必要はありません。
   含めなかったメンバーは、検証済みのコードからは使えません。

仕様を用意すれば、検証済みのコードでそのトレイトを使えます。

```rust
fn use_encoder<E: Encoder>(f: &E, val: u64) -> (result: u64)
    ensures
        result >= val,
{
    f.encode_value(val)
}
```

## 外部トレイトの拡張（spec の補助関数） {#external-trait-extension-spec-helper-functions}

トレイトの仕様に、元のトレイトには無い spec モードの関数が追加で必要になることがあります。
たとえば、あるメソッドの抽象的なふるまいを記述する spec 関数が欲しい場合です。
`#[verifier::external_trait_extension]` 属性は、これに対応しています。

この属性は、次の形をとります。
```
#[verifier::external_trait_extension(SpecTrait via SpecImplTrait)]
```

 * **SpecTrait** は、検証の中で使えるようになる spec モードのトレイトの名前です。
   これは、外部トレイトを実装するすべての型に対して自動的に実装されます。
 * **SpecImplTrait** は、spec の補助関数を定義するために具体的な型が実装するトレイトの名前です。

次に示すのは、`Summarizer` という架空の外部トレイトを使った例です。

```rust
#[verifier::external_trait_specification]
#[verifier::external_trait_extension(SummarizerSpec via SummarizerSpecImpl)]
trait ExSummarizer {
    type ExternalTraitSpecificationFor: Summarizer;

    // A spec helper function (not part of the original trait):
    spec fn spec_summary(&self) -> u64;

    fn summary(&self) -> (result: u64)
        ensures
            result == self.spec_summary();
}
```

具体的な型は、`SpecImplTrait`（ここでは `SummarizerSpecImpl`）を実装して spec の補助関数を定義します。
そうすると、検証済みのコードでその仕様を使えます。

```rust
struct MyCustomStruct { value: u64 }

// Implement the concrete external Summarizer trait
impl Summarizer for MyCustomStruct {
    fn summary(&self) -> u64 {
        // Prove that our overly complicated implementation satisfies the spec_summary
        let v = self.value;
        assert(v & 0xffff_ffff_ffff_ffff == v) by (bit_vector);
        v & 0xffff_ffff_ffff_ffff
    }
}

// Implement the additional spec functions
impl SummarizerSpecImpl for MyCustomStruct {
    spec fn spec_summary(&self) -> u64 {
        self.value
    }
}

fn test_hasher(h: &MyCustomStruct) {
    let v = h.summary();
    assert(v == h.value);
}
```

## `vstd` での `obeys_*` パターン {#the-obeys_-pattern-in-vstd}

`vstd` は、`PartialEq`、`Ord`、`Add`、`From` などの標準ライブラリのトレイトに対して、
`external_trait_extension` を広く使っています。これらの仕様は、`obeys_*` という spec 関数を使う共通のパターンに従っています。
この spec 関数は、ある型の実装が実際にその仕様に従っているかどうかを表します。

たとえば、`vstd` での `PartialEq` の仕様は、おおよそ次のようになっています。

```rust
#[verifier::external_trait_specification]
#[verifier::external_trait_extension(PartialEqSpec via PartialEqSpecImpl)]
pub trait ExPartialEq<Rhs = Self> {
    type ExternalTraitSpecificationFor: PartialEq<Rhs>;

    spec fn obeys_eq_spec() -> bool;
    spec fn eq_spec(&self, other: &Rhs) -> bool;

    fn eq(&self, other: &Rhs) -> (r: bool)
        ensures
            Self::obeys_eq_spec() ==> r == self.eq_spec(other);
}
```

この ensures 節は、その型が `PartialEq` についての Verus の仕様に従っている（`obeys_eq_spec()` が真である）**ならば**、
結果は `eq_spec` と一致する、と述べています。整数型と `bool` については、`vstd` は `obeys_eq_spec()` を真と定義し、
これらの型が `eq_spec` を満たすことを証明しています。そのほかの型については、
Verus は `obeys_eq_spec()` が真か偽かを知らないので、事後条件が成り立つとは仮定しません。
このパターンのおかげで、`vstd` は、すべての型について不健全な仮定を置くことなく、
行儀のよい型に対して役に立つ仕様を提供できます。外部の型に対して `Eq` のようなトレイトを使いたい場合は、
[`assume_specification`](./reference-assume-specification.html)
を使って、`obeys_eq_spec()` が真であると述べられます。

## 規則と制限 {#rules-and-restrictions}

 * 関連型 `ExternalTraitSpecificationFor` は必須で、外部トレイトを指していなければなりません。
 * 仕様のトレイトでは、どのメソッドにも本体を持たせないでください。
 * ジェネリックパラメータと関連型は、外部トレイトと厳密に一致していなければなりません。
 * `external_trait_extension` を使うと、2 つの名前（`SpecTrait` と `SpecImplTrait`）は実際のトレイト名になります。
   `SpecTrait` は境界で使え、`SpecImplTrait` は `impl` ブロックで使えます。

{% endraw %}

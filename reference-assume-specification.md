{% raw %}
# assume_specification {#assume_specification}

> [!CAUTION]
> `assume_specification` 文は検査されないため、Verus の保証を簡単にすり抜けられてしまいます。
>
> [検証されていないコードとやり取りするときの助言](./interacting-with-unverified-code.html)を必ず読んでください。

`assume_specification` ディレクティブは、与えられた関数に対して、与えられた仕様を使うよう Verus に伝えます。
Verus は、この仕様が成り立つことを**証明なしで**仮定します。
これは検証されていない仮定を持ち込むので、
[no-cheating モード](./tcb.html#no-cheating-mode)では、その関数が（直接または間接に）[信頼されたものとして注釈されて](./tcb.html)いない限り、拒否されます。

これは、そのままでは Verus が知りえない、任意の `exec` モードの関数に使えます。たとえば、
[`external`](./reference-attributes.html#verifierexternal) と印が付いた関数や、外部のクレートからインポートした関数です。

これは、`external_body` な関数を持つことに似ています。違いは、`assume_specification` を使う場合、仕様が関数の宣言や本体から切り離されていることです。

`assume_specification` の宣言は、対応する関数と同じモジュールやクレートにある必要は**ありません**。ただし、次の条件があります。
 * 関数が、その `assume_specification` の宣言から見えていなければなりません
 * `assume_specification` の宣言が、関数が見えるところならどこからでも見えていなければなりません。

このディレクティブの一般形は、次のとおりです。

```verus-grammar
V@[assume_specification_item] ::=
    visibility? assume_specification R@[generics]? [ R@[function_path] ] (R@[args...]) ( -> V@[exec_return_type] )?
        R@[where_clause]?
        V@[requires_clause]?
        V@[ensures_clause]?
        V@[returns_clause]?
        V@[invariants_clause]?
        V@[unwind_clause]?
        ;
```

これは、[Verus の仕様](./reference-exec-signature.html)が付いた通常の Rust の関数シグネチャのように見えることを意図しています。違いは、名前を持つ代わりに、別の関数をパスで参照することです。

関連関数とメソッドの場合、<code><span style="color: #800000; font-style: italic">function_path</span></code> は `Type::method_name` という形にします。
型には「ターボフィッシュ構文」を使います（例: `Vec::<T>`）。
トレイトのメソッドの場合、<code><span style="color: #800000; font-style: italic">function_path</span></code> には[「修飾された self」](https://doc.rust-lang.org/reference/paths.html#qualified-paths)の形、つまり `<Type as Trait>::method_name` を使います。

シグネチャは、引数、戻り値の型、ジェネリクス、トレイト境界を含めて、対象の関数と同じでなければなりません。
すべての引数に名前を付け、`self` は使わ*ない*でください。

### 例 {#examples}

通常の関数に適用する場合:

```rust
pub assume_specification<T> [core::mem::swap::<T>] (a: &mut T, b: &mut T)
    ensures
        *a == *old(b),
        *b == *old(a),
    opens_invariants none
    no_unwind;
```

`Vec` の関連関数に適用する場合:

```rust
pub assume_specification<T>[Vec::<T>::new]() -> (v: Vec<T>)
    ensures
        v@ == Seq::<T>::empty();
```

`Vec` のメソッドに適用する場合:

```rust
pub assume_specification<T, A: Allocator>[Vec::<T, A>::clear](vec: &mut Vec<T, A>)
    ensures
        vec@ == Seq::<T>::empty();
```

特定の型の `clone` に適用する場合:

```rust
pub assume_specification [<bool as Clone>::clone](b: &bool) -> (res: bool)
    ensures res == b;
```

### 型シグネチャの一致 {#type-signature-matching}

Verus は、型シグネチャとトレイト境界が厳密に一致することを求めます。
これらが揃っていないと、一致しないシグネチャ、または一致しないトレイト境界を示すエラーメッセージが出ます。
ソース上の形と内部の形に違いがあるため、これらを揃えるのが簡単でないこともあります。

コツ:

1. ふつうは、すべてのライフタイム変数（たとえば `'a`）を明示すると、うまくいきやすくなります。

2. `Sized` トレイトはやや複雑です。`Sized` 境界のソース上の形が、Rust の内部でのトレイト境界の表現と
   同じに見えるとは限らないからです。安定化されていない `Sized` 関連の機能によって、さらに複雑になっています。
   Rust 1.95.0 の時点では、両者は次のように対応しています。

| ソース上の境界    | 内部の境界     |
|-------------------|----------------|
| 境界なし          | `T: Sized`     |
| `T: Sized`        | `T: Sized`     |
| `T: ?Sized`       | `T: MetaSized` |
| `T: PointeeSized` | 境界なし       |

{% endraw %}

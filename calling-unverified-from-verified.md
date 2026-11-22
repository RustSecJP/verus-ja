{% raw %}
# 検証済みコードから未検証コードを呼び出す {#calling-unverified-code-from-verified-code}

システムの一部だけを検証することはよくあります。その場合、検証済みコードから未検証コードを呼び出す必要が出てきます。
そのためには、Verus に未検証コードの存在を認識させ、何を **証明なしに仮定する** べきかを Verus に伝える必要があります。

以下の仕組み（`external_body`、`assume_specification`）は、検証されていない仮定を持ち込みます。
そのため、それを含む関数が（直接または間接に）[信頼済みと注釈されて](./tcb.html)いない限り、[no-cheating モード](./tcb.html#no-cheating-mode)では拒否されます。

## 証明のない仕様 {#specifications-without-proof}

未検証の関数に仮定を当てはめる方法の 1 つは、`#[verifier::external_body]` 属性を使うことです。
この属性は、関数の本体を検証も処理もせずに、関数の _仕様_ だけを処理するよう Verus に伝えます。
つまり、Verus はその仕様を証明なしに仮定することになります。
言うまでもなく、これは慎重に使うべきです。誤った仕様は Verus の保証を台無しにしかねないからです。

```rust
#[verifier::external_body]
fn fib_impl(n: u64) -> (result: u64)
    requires
        fib(n as nat) <= u64::MAX
    ensures
        result == fib(n as nat),
{
    if n == 0 {
        return 0;
    }
    let mut prev: u64 = 0;
    let mut cur: u64 = 1;
    let mut i: u64 = 1;
    while i < n {
        i = i + 1;
        let new_cur = cur + prev;
        prev = cur;
        cur = new_cur;
    }
    cur
}
```

この実装は正しいのですが、証明されていません。
`external_body` 属性を取り除くと、Verus は本体を検証しようとし、ループ不変条件が無いために失敗します。
（この例について詳しくは[こちら](./invariants.html)を参照してください。）

## _既存の_ ライブラリ関数に仕様を当てはめる {#applying-specifications-to-existing-library-functions}

既存の関数、たとえば何らかのライブラリクレートや、Rust の標準ライブラリで定義された関数に仕様を当てはめたい、ということはよくあります。

その方法の 1 つは、ライブラリ関数を呼び出す「ラッパー関数」を `external_body` 付きで書くことです。
たとえば、[`std::mem::swap`](https://doc.rust-lang.org/stable/std/mem/fn.swap.html) を呼び出したいとしましょう。
次のようなラッパー関数を書けます。

```rust
#[verifier::external_body]
fn wrapper_swap<T>(a: &mut T, b: &mut T)
    ensures
        *a == *old(b),
        *b == *old(a),
{
    std::mem::swap(a, b);
}
```

しかし、これは不便かもしれません。なじみのある `std::mem::swap` の代わりに、`wrapper_swap` を呼び出さなければならなくなるからです。
`std::mem::swap` そのものに仕様を当てはめて、検証済みコードから呼び出せるようにしたい場合は、[`assume_specification` ディレクティブ](./reference-assume-specification.html)を使えます。
これは（関数と同じく）アイテムの階層に書きます。

```rust
pub assume_specification<T>[ std::mem::swap::<T> ](a: &mut T, b: &mut T)
    ensures
        *a == *old(b),
        *b == *old(a);
```

これで、検証済みコードから `std::mem::swap` を呼び出せます。

（ただし、vstd は `std::mem::swap` に対するこの仕様を _すでに_ 提供していることに注意してください。
Verus は仕様の重複を許さないため、2 つ目の仕様を宣言することはできません。
この例を自分で試したい場合は、`--no-vstd` フラグを付けて Verus を実行する必要がありますが、これは通常の使い方としてはおすすめしません。）

### 標準ライブラリの仕様 {#standard-library-specifications}

実のところ、vstd はこのディレクティブを使って、標準ライブラリに対する幅広い仕様を提供しています。
そのため、（通常の使い方どおり）vstd をインポートして Verus を実行していれば、これらの仕様は自動的にインポートされます。
その内容は[こちら](https://verus-lang.github.io/verus/verusdoc/vstd/std_specs/struct.VstdSpecsForRustStdLib.html)に記載されています。

もちろん、使いたい標準ライブラリの関数の仕様を vstd が提供していない場合は、自分のクレートに `assume_specification` を追加することもできます。

## Verus に型を認識させる {#making-verus-aware-of-types}

Verus が、ある型を認識できないと文句を言うことがあります。その場合は、その型を Verus に認識させるだけで済みます。
それには、`#[verifier::external_type_specification]` 属性を使えます。

次のように書くと、Verus は `SomeStruct` を認識するようになります。

```rust
#[verifier::external_type_specification]
struct ExSomeStruct(SomeStruct);
```

括弧とセミコロンを含め、厳密にこの形でなければなりません。
`ExSomeStruct` という名前は何でも構いません。これはディレクティブのために使う人工的な型にすぎず、ほかの場所から参照するべきではありません。

この宣言によって、Verus は型 `SomeStruct` とそのすべてのフィールド（列挙型の場合はすべてのバリアント）を認識するようになります。
フィールドやバリアントを Verus に認識させたくない場合は、`#[verifier::external_body]` も付けられます。

## 外部のトレイトに仕様を追加する {#adding-specifications-for-external-traits}

`#[verifier::external_trait_specification]` 属性を使うと、外部のトレイトにも仕様を追加できます。
これを使うと、トレイトのメソッドに `requires` 節と `ensures` 節を追加できます。
さらに、必要に応じて、`#[verifier::external_trait_extension]` で仕様用の補助関数を定義することもできます。

詳細と例については、[外部トレイトの仕様](./external_trait_specifications.html)を参照してください。
{% endraw %}

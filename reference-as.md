{% raw %}
# `as` による型強制 {#coercion-with-as}

## 構文 {#syntax}

```verus-grammar
V@[as_expr] ::= V@[spec_expr] as R@[type]
```

## 整数型と `char` の型強制 {#coercions-on-integer-types-and-char}

次の型はどれも、ほかのどの型へも型強制（coercion）できます。

 * `i8`、`i16`、`i32`、`i64`、`i128`、`isize`
 * `u8`、`u16`、`u32`、`u64`、`u128`、`usize`
 * `int`
 * `nat`
 * [`char`](./reference-types.html#char)

これは、Rust の exec コードにおける `as` よりも寛容であることに注意してください。
たとえば、Rust では `as` を使って `u16` から `char` へキャストすることは許されませんが、Verus の spec コードでは許されます。

### 定義 {#definition}

Verus は `as` キャストを次のように定義します。

 * `int` へのキャストはつねに定義されており、切り捨てを必要としません。
 * `nat` へのキャストは、入力値が負の場合は*未規定*（unspecified）です。
 * `char` へのキャストは、入力値が[`char` として許される値](./reference-types.html#char)の範囲外である場合は*未規定*です。
 * それ以外の有限サイズの整数型へのキャストは、*切り捨て*（truncation）として定義されます。つまり、適切な N について下位 N ビットを取り出し、その結果を符号付きまたは符号なしの整数として解釈します。

### 切り捨てについての推論 {#reasoning-about-truncation}

切り捨ての定義は、Verus の既定の証明器モードでは公開されて*いません*（つまり、未規定であるかのように振る舞います）。
切り捨てについて推論するには、[ビットベクトルソルバー](./reference-prover-mode-bit-vector.html)か [compute ソルバー](./reference-prover-mode-compute.html)を使ってください。

また、`usize` と `isize` に対する N の値は、[`global` ディレクティブで設定](./reference-global.html)できることにも注意してください。

## ポインターの型強制 {#coercions-on-pointers}

### ポインターから整数へ {#pointers-to-integers}

どのポインターも、どの整数型へも型強制できます。
ポインター `ptr: *mut T` または `ptr: *const T` に対して、式 `ptr as INT_TYPE` は `ptr.addr() as INT_TYPE` と等価です。

> [!NOTE]
> `as` キャストよりも、`ptr.addr()` を明示的に使うほうが分かりやすいことがよくあります。

> [!NOTE]
> Verus は、`as` を使って整数をポインターへキャストすることには対応していません。代わりに、[`with_addr`](https://doc.rust-lang.org/std/primitive.pointer.html#method.with_addr) や [`with_exposed_provenance`](https://verus-lang.github.io/verus/verusdoc/vstd/raw_ptr/fn.with_exposed_provenance.html) のような関数を使ってください。

### ポインターからポインターへ {#pointers-to-pointers}

Verus は、ポインターからポインターへの型強制のいくつかに対応しています。`*const` ポインターと `*mut` ポインターのあいだの変換もその 1 つです。

 * `*mut T` から `*const T` へ（恒等関数）
 * `*const T` から `*mut T` へ（恒等関数）

そのほかの種類の型強制もあります。

 * 任意の `S: Sized` について、`*mut T` から `*mut S` へ（[仕様](https://verus-lang.github.io/verus/verusdoc/vstd/raw_ptr/fn.spec_cast_ptr_to_thin_ptr.html)）
 * `*mut [T; N]` から `*mut [T]` へ（[仕様](https://verus-lang.github.io/verus/verusdoc/vstd/raw_ptr/fn.spec_cast_array_ptr_to_slice_ptr.html)）
 * `*mut [T]` から `*mut [U]` へ（[仕様](https://verus-lang.github.io/verus/verusdoc/vstd/raw_ptr/fn.spec_cast_slice_ptr_to_slice_ptr.html)）
 * `*mut [T]` から `*mut str` へ（[仕様](https://verus-lang.github.io/verus/verusdoc/vstd/raw_ptr/fn.spec_cast_slice_ptr_to_str_ptr.html)）
 * `*mut str` から `*mut [T]` へ（[仕様](https://verus-lang.github.io/verus/verusdoc/vstd/raw_ptr/fn.spec_cast_str_ptr_to_slice_ptr.html)）

`*const` ポインターについても、これらに相当するキャストがひととおりあります。
{% endraw %}

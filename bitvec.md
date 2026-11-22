{% raw %}
# ビットベクトルとビット演算 {#bit-vectors-and-bitwise-operations}

既定の証明器モードでは、Verus は `&`、`|`、`^`、`<<`、`>>` のようなビット演算を未解釈関数（uninterpreted function）として扱います。
`x & y == y & x` のような基本的な事実でさえ、Verus の既定のソルバーモードでは提供されません。

こうした状況に対処するために、Verus は専用のソルバーモード `bit_vector` を用意しています。
このソルバーはビット演算子についての性質にとても向いています。範囲の限られた整数の算術もいくらか扱えますが、こちらについては、効き目は場合によります。

## `bit_vector` 証明器モードを呼び出す {#invoking-the-bit_vector-prover-mode}

`bit_vector` 証明器モードは、[`nonlinear_arith` と同じように](./nonlinear.html#1-invoking-a-specialized-solver-nonlinear_arith)、`assert` または `proof fn` に `by(bit_vector)` を付けて呼び出せます。

`assert` に `by(bit_vector)` を使うと、次の例のように、短くて文脈に依存しないビット操作の性質を表明できます。
```rust
fn test_passes(b: u32) {
    assert(b & 7 == b % 8) by (bit_vector);
    assert(b & 0xff < 0x100) by (bit_vector);
}
```

`nonlinear_arith` と同じく、`assert(...) by(bit_vector)` で表した表明には、周囲のコンテキストからの事実（たとえば、外側の関数の `requires` 節や、それより前の変数への代入から得られるもの）は一切含まれません。
たとえば、次の例は失敗します。

```rust
fn test_fails(x: u32, y: u32)
  requires x == y
{
  assert(x & 3 == y & 3) by(bit_vector);  // Fails
}
```

しかし、`requires` 節を使えば、コンテキストを明示的に取り込めます。

```rust
fn test_success(x: u32, y: u32)
    requires
        x == y,
{
    assert(x & 3 == y & 3) by (bit_vector)
        requires
            x == y,
    ;  // now x == y is available for the bit_vector proof
}
```

証明関数 `f` に `by(bit_vector)` を付けると、Verus は `f` を検証するときに `bit_vector` ソルバーを使います。
ただし、別の関数が `f` を呼び出すときには、Verus は _通常の_ ソルバーを使って、`f` の事前条件（precondition）がすべて満たされていることを検証する点に注意してください。
証明関数に `by(bit_vector)` を使う例は次のとおりです。
```rust
proof fn de_morgan_auto()
    by (bit_vector)
    ensures
        forall|a: u32, b: u32| #[trigger] (!(a & b)) == !a | !b,
        forall|a: u32, b: u32| #[trigger] (!(a | b)) == !a & !b,
{
}
```


## `bit_vector` ソルバーの仕組みと得意なこと {#how-the-bit_vector-solver-works-and-what-its-good-at}

`bit_vector` ソルバーは、別の SMT 符号化を使います。ただし、すべての算術演算の意味は同じままの符号化です。
具体的には、すべての整数を [Z3 の `bv` 型](https://microsoft.github.io/z3guide/docs/theories/Bitvectors/)に符号化し、算術を組み込みのビットベクトル演算で符号化します。
SMT ソルバーは内部で、「ビットブラスティング（bit blasting）」と呼ばれる技法を使います。

この符号化を実現するために、Verus は、与えられた整数を表すのに適切なビット幅を選ぶ必要があります。
記号的な固定幅の整数値（たとえば `u64`）については、適切なビット幅（たとえば 64 ビット）を選ぶだけで済みます。
算術演算の結果については、Verus が適切なビット幅を自動的に選びます。
しかしこの理由のために、ビットベクトルのソルバーは、_記号的な_ 整数の値については推論できません。

`bit_vector` ソルバーは、[ビット演算](./spec-bit-ops.html)（`&`、`|`、`^`、`<<`、`>>`）についての証明に最適です。
ただし、範囲の限られた整数に対する算術（`+`、`-`、`*`、`/`、`%`）も、そこそこうまく扱えます。

## 例とこつ {#examples-and-tips}

### 関数 {#functions}

`bit_vector` ソルバーは、定数と `spec` 関数の使用に対応しています。
ただし、それらの関数が使えるのは、やはり上に挙げたのと同じビットベクトル演算と算術演算に限られます（内部的には、関数への対応は関数をインライン展開することで実現されています）。

関数に対応しているということは、内容が伝わる名前の補助関数を書いて使えるということです。
たとえば、次のように定義できます。

```rust
spec fn get_bit(val: u32, index: u32) -> bool {
    0x1u32 & (val >> index) == 1
}

fn test_get_bit() {
    assert(get_bit(128u32, 7)) by (bit_vector);
}
```

再帰関数に対応しているのは、停止することを静的に判定できる場合（たとえば、定数に対するフィボナッチの計算）だけである点に注意してください。

関数を不透明なまま（つまり、本体をインライン展開せずに）推論したい場合は、その結果を（ゴースト）変数に代入してから、ビットベクトルの表明の中で使えます。
たとえば次のとおりです。
```rust
proof fn test_overflow_check(a: u8, b: u8) {
    // Because we call `complex_f` here, it remains opaque in the assertion.
    let c = complex_f(a, b);
    assert(c >> 1 == c / 2) by(bit_vector);
}
```


### オーバーフローの検査 {#overflow-checking}

`bit_vector` ソルバーは記号的な `int` の値を扱いませんが、`int` の値を返す多くの算術演算には対応 _しています_。
これにより、オーバーフローについての条件を書けます。

```rust
proof fn test_overflow_check(a: u8, b: u8) {
    // `a` and `b` are both `u8` integers, but we can test if their addition
    // overflows a `u8` by simply writing `a + b < 256`.
    assert((a & b) == 0 ==> (a | b) == (a + b) && (a + b) < 256) by(bit_vector);
}
```

### 整数のラップアラウンドと切り捨て {#integer-wrapping-and-truncation}

`bit_vector` ソルバーは、切り捨てについて推論する最も簡単な方法の 1 つです。切り捨ては、ビット演算で自然に表せます。

```rust
proof fn test_truncation(a: u64) {
    assert(a as u32 == a & 0xffff_ffff) by(bit_vector);

    // You can write an identity with modulus as well:
    assert(a as u32 == a % 0x1_0000_0000) by(bit_vector);
}
```

`add`、`sub`、`mul` を使うのも便利かもしれません。これらは（`+`、`-`、`*` と違って）自動的に切り捨てます。

```rust
proof fn test_truncating_add(a: u64, b: u64) {
    assert(add(a, b) == (a + b) as u64) by(bit_vector);
}
```

### `usize` と `isize` を扱う {#working-with-usize-and-isize}

`usize` 型や `isize` 型の変数を使うと、`bit_vector` ソルバーは（既定では）それらが 32 ビットかもしれないし 64 ビットかもしれないと仮定します。これは符号化に影響します。
その場合、ソルバーは 2 つの異なる問い合わせを生成し、両方を検証します。

ただし、ソルバーは[特定のプラットフォームのサイズを仮定するように設定する](./reference-global.html#with-usize-and-isize)こともできます。

### ビット幅への依存と非依存 {#bitwidth-dependence-and-independence}

多くの演算では、その結果は入力のビット幅に依存しません。
`&`、`|`、`^`、`>>` がこれに当たります。
実のところ、これを証明するのに `bit_vector` ソルバーすら必要ありません。通常のソルバーモードも、この事実を「知っている」からです。

```rust
proof fn test_xor_u32_vs_u64(x: u32, y: u32) {
    assert((x as u64) ^ (y as u64) == (x ^ y) as u64) by(bit_vector);

    // XOR operation is independent of bitwidth so we don't even
    // need the `bit_vector` solver to do this:
    assert((x as u64) ^ (y as u64) == (x ^ y) as u64);
}
```

しかし、左シフト `<<` については、これは成り立ち _ません_。
左シフトでは、左オペランドのビット幅に常に注意する必要があります。

```rust
proof fn test_left_shift_u32_vs_u64(y: u32) {
    assert(1u32 << y == 1u64 << y); // FAILS (in either mode) because it's not true
}
```

### そのほかの例 {#more-examples}

目を通してみるとよい、もう少し大きな例をいくつか挙げます。

 * [ガベージコレクションの例](https://github.com/verus-lang/verus/blob/main/examples/bitvector_garbage_collection.rs)
 * [ビットベクトルの同値性の例](https://github.com/verus-lang/verus/blob/main/examples/bitvector_equivalence.rs)
 * [そのほか](https://github.com/verus-lang/verus/blob/main/examples/bitvector_basic.rs)
{% endraw %}

{% raw %}
# const 宣言 {#const-declarations}

Verus では、`const` 宣言は、内部的には引数が 0 個の関数呼び出しとして扱われます。
そのため、関数とちょうど同じように、`const` 宣言には `spec`、`proof`、`exec` の印を付けることも、明示的なモードを付けないままにすることもできます。
既定では、明示的なモードの無い `const` には、`spec/exec` という二重のモードが割り当てられます。
これらのモードがそれぞれ何を意味するのかを、順に見ていきます。

## `spec` の const {#spec-consts}

`spec const` は、引数の無い `spec` 関数のようなものです。
常にゴーストであり、`exec` の値としては使えません。

```rust
spec const SPEC_ONE: int = 1;

spec fn spec_add_one(x: int) -> int {
    x + SPEC_ONE
}

```

## `proof` と `exec` の const {#proof-and-exec-consts}

`proof` 関数と `exec` 関数が、事後条件を指定する `ensures` 節を持てるのとちょうど同じように、`proof` と `exec` の const も、宣言を `spec` の式に結び付ける `ensures` 節を持てます。
構文は、関数定義の構文にならっています。

```rust
exec const C: u64
    ensures
        C == 7,
{
    7
}

```

ここで、const を定義するときに `assert` も使えること、そして、別の `const` 関数の呼び出しを使って定義できることに注意してください。

```rust
spec fn f() -> int {
    1
}

const fn e() -> (u: u64)
    ensures
        u == f(),
{
    1
}

exec const E: u64
    ensures
        E == 2,
{
    assert(f() == 1);
    1 + e()
}

```

## `spec/exec` の const {#specexec-consts}

明示的なモードの無い `const` は、二通りに使えます。
`exec` の値としても、`spec` の値としても使えます。

```rust
const ONE: u8 = 1;

fn add_one(x: u8) -> (ret: u8)
    requires
        x < 0xff,
    ensures
        ret == x + ONE,  // use "ONE" in spec code
{
    x + ONE  // use "ONE" in exec code

}

```

そのため、この `const` の定義は、`exec` コードと `spec` コードの両方の規則に従うように制限されます。
たとえば、`exec` コードと同じように、その型はコンパイルできるものでなければなりません（たとえば `int` ではなく `u8`）。
また、`spec` コードと同じように、`exec` 関数や `proof` 関数を呼び出せません。

```rust
const fn foo() -> u64 { 1 }
const C: u64 = foo();  // FAILS with error "cannot call function `foo` with mode exec"
```

## `spec` や `proof` の文脈で `exec const` を使う {#using-an-exec-const-in-a-spec-or-proof-context}

関数の場合と同じように、`exec const` を `spec` や `proof` の文脈で使いたいときは、宣言に `#[verifier::when_used_as_spec(SPEC_DEF)]` という注釈を付けられます。
ここで `SPEC_DEF` は、`spec const` の名前、または引数の無い `spec` 関数の名前です。

```rust
use vstd::layout;

global layout usize is size == 8;

spec const SPEC_USIZE_BYTES: usize = layout::size_of_as_usize::<usize>();

#[verifier::when_used_as_spec(SPEC_USIZE_BYTES)]
exec const USIZE_BYTES: usize
    ensures
        USIZE_BYTES as nat == layout::size_of::<usize>(),
{
    8
}

```

この例では、注釈が無いと、Verus は「cannot read const with mode exec.」というエラーを出します。

さらに、`SPEC_USIZE_BYTES` を別に定義せずに、`#[verifier::when_used_as_spec(layout::size_of::<usize>)]` という注釈を使おうとしても、やはりエラーになります。
`when_used_as_spec` が扱えるのは、2 つの関数または const が同じシグネチャを持つ場合だけです。
`::<usize>` のようなものを使って、関数を別のシグネチャに変換することは扱えません。

## オーバーフローのエラーへの対処 {#trouble-shooting-overflow-errors}

Verus は、`const` 宣言がオーバーフローしないことを証明するのに苦労することがあります。
`#[verifier::when_used_as_spec(SPEC_DEF)]` や `#[verifier::nonlinear]` を使うと、助けになるかもしれません。

たとえば、ここでは、「possible arithmetic underflow/overflow.」というエラーを防ぐために `#[verifier::nonlinear]` を追加しています。
これにより、Verus は、（見かけ上は）非線形な式 `BAR_PLUS_ONE * BAR` について、すぐに諦めるのではなく、推論できるようになります。
詳しくは、[非線形の推論についての章](nonlinear.html)を参照してください。

```rust
pub const FOO: u8 = 4;

pub const BAR: u8 = FOO;

pub const BAR_PLUS_ONE: u8 = BAR + 1;

#[verifier::nonlinear]
pub const G: u8 = BAR_PLUS_ONE * BAR;

```
{% endraw %}

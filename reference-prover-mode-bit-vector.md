{% raw %}
# `bit_vector` 証明器モード {#the-bit_vector-prover-mode}

> [!TIP]
> `bit_vector` ソルバーを使うときの実践的なコツは、[ガイドのページ](./bitvec.html)を参照してください。

### 呼び出し方 {#methods-of-invocation}

**アサーションで呼び出す。**
`bit_vector` ソルバーは、[`assert-by` 文](./reference-assert-by-prover.html)で呼び出せます。

```
assert(Q) by(bit_vector);
```

`bit_vector` ソルバーで `Q` を証明します。

```
assert(Q) by(bit_vector) requires P;
```

`bit_vector` ソルバーで `P ==> Q` を証明します。

**proof 関数として呼び出す。**
`bit_vector` ソルバーは、[proof 関数](./reference-proof-signature.html)に `by(bit_vector)` を付けて呼び出せます。

```
proof fn example(...) by(bit_vector)
    requires P
    ensures Q
{ }
```

これは、`bit_vector` ソルバーで `P ==> Q` を証明します。

### 対応している述語 {#supported-predicates}

 * `bool` 型、または幅が有限の整数型（`u64`、`i64`、`usize` など）の変数
   * 自由変数はすべて記号的に扱われます。変数が、ビットベクトルのアサーションの外で宣言された `let` 文で
     定義されていても、その定義はソルバーからは見えません。
 * 整数リテラルと真偽値リテラル
 * 切り捨てを行わない算術演算（`+`、`-`、`*`、`/`、`%`）
 * 切り捨てを行う算術演算（`add`、`sub`、`mul` 関数）
 * ビット演算（`&`、`|`、`^`、`!`、`<<`、`>>`）
 * 等号と不等号（`==`、`!=`、`<`、`>`、`<=`、`>=`）
 * 真偽値の演算子（`&&`、`||`、`^`）と条件式
 * 定数 `usize::BITS`

## ソルバーの動作 {#solver-operation}

Verus のビットベクトルソルバーは、すべての整数を SMT の「ビットベクトル」型で表すことで、式を符号化します。
上に挙げた制約のほとんどは、Verus がどの式についても固定のビット幅を選ばなければならないことから生じています。

なお、ビットベクトルソルバーは `int` 型や `nat` 型の自由変数を扱えませんが、
その数を表すのに必要なビット幅の上限を Verus が求められるのであれば、
`int` 型や `nat` 型になる式の一部は扱*えます*。
たとえば、`x` と `y` の型が `u64` であれば、`x + y` の型は `int` ですが、
Verus は `x + y` が 65 ビットで表せることを知っています。

### `usize` と `isize` の扱い {#handling-usize-and-isize}

式の中に、幅がアーキテクチャに依存する記号的な値が使われていて、
アーキテクチャのビット幅が [`global` ディレクティブ](./reference-global.html)で指定されていない場合、
Verus は、ありうるビット幅（32 ビットまたは 64 ビット）ごとに 1 つずつ、複数のクエリを生成します。
{% endraw %}

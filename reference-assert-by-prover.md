{% raw %}
# assert ... by(...) {#assert--by}

`assert ... by` 文は、与えられたアサーションを証明するために、専用の*証明器モード*（prover mode）を呼び出すのに使います。

### 構文 {#syntax}

証明器モードは、「ソルバーモード」（ソルバーを呼び出すもの）と「インタプリタモード」（インタプリタを呼び出すもの）に分けられます。

```verus-grammar
V@[assert_by_prover_stmt] ::=
    assert ( V@[spec_expr] ) by ( V@[assert_by_solver_mode] ) V@[assert_by_prover_requires]? ;
  | assert ( V@[spec_expr] ) by ( V@[assert_by_interpreter_mode] );

V@[assert_by_solver_mode] ::= nonlinear_arith | bit_vector;
V@[assert_by_interpreter_mode] ::= compute | compute_only;

V@[assert_by_prover_requires] ::=
    requires (V@[spec_expr],)+
```

> [!NOTE]
> 現時点では、[`integer_ring`](./reference-prover-mode-integer-ring.html) 証明器モードは
> [proof 関数の宣言](./reference-proof-signature.html)でしか使えず、
> assert-by では使えません。

### 証明上の動作 {#proof-operation}

**ソルバーモード。**
`nonlinear_arith` モードと `bit_vector` モードでは、
Verus は与えられた述語を、指定されたソルバー
（[`nonlinear_arith` ソルバー](./reference-prover-mode-nonlinear.html)
または [`bit_vector` ソルバー](./reference-prover-mode-bit-vector.html)）
で証明しようとします。
述語は、周囲の文脈を一切使わずに、単独で証明されます。

具体的には、
<code>assert ( Q ) by ( V@[assert_by_solver_mode] )</code>
という文に対して、Verus はそのソルバーを使って `Q` を証明しようとし、それ以降のコードでは `Q` を仮定します。

さらに `requires P` 節が与えられている場合、Verus は次のことを行います。

 * `P` を既定のソルバーで証明する（文脈をすべて使える）
 * `P ==> Q` を指定されたソルバーで証明する（単独で）
 * 最後に、それ以降のコードでは `Q` を仮定する

**インタプリタモード。**
`compute` モードと `compute_only` モードでは、Verus は[仕様インタプリタ](./reference-prover-mode-compute.html)を使って式 `Q` をできるだけ簡約し、式 `Q'` を得ます。

 * `compute_only` の場合、Verus は `Q'` が真偽値 `true` であることを検査し、
   それ以降のすべてのコードでは、与えられた述語を仮定します。
 * `compute` の場合、Verus は assert-by 文を `assert(Q')` に置き換えます。これは、その後、
   通常の [`assert`](./reference-assert.html) と同じようにふるまいます。
{% endraw %}

{% raw %}
# 非線形ソルバー {#the-nonlinear-solver}

> [!TIP]
> `nonlinear_arith` ソルバーを使うときの実践的なコツは、[ガイドのページ](./nonlinear.html)を参照してください。

### 呼び出し方 {#methods-of-invocation}

**アサーションで呼び出す。**
`nonlinear_arith` ソルバーは、[`assert-by` 文](./reference-assert-by-prover.html)で呼び出せます。

```
assert(Q) by(nonlinear_arith);
```

`nonlinear_arith` ソルバーで `Q` を証明します。

```
assert(Q) by(nonlinear_arith) requires P;
```

`nonlinear_arith` ソルバーで `P ==> Q` を証明します。

**proof 関数として呼び出す。**
`nonlinear_arith` ソルバーは、[proof 関数](./reference-proof-signature.html)に `by(nonlinear_arith)` を付けて呼び出せます。

```
proof fn example(...) by(nonlinear_arith)
    requires P
    ensures Q
{ }
```

これは、`nonlinear_arith` ソルバーで `P ==> Q` を証明します。

## ソルバーの動作 {#solver-operation}

このソルバーは、Z3 の非線形算術の理論を使います。これにより、記号的な値の乗算や除算を含む問題を解けることがよくあります。
たとえば、`a * b == b * a` のような交換法則の公理が、このモードでは使えます。
{% endraw %}

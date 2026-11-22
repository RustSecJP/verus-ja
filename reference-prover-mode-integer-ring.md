{% raw %}
# `integer_ring` ソルバー {#the-integer_ring-solver}

> [!NOTE]
> `integer_ring` ソルバーを使うには、Singular というツールをインストールしておく必要があります。これは、Verus のバイナリリリースには既定では含まれていません。[インストールの詳細](./install-singular.html)を参照してください。

> [!TIP]
> `integer_ring` ソルバーを使うときの実践的なコツは、[ガイドのページ](./nonlinear.html#2-proving-ring-based-properties-integer_ring)を参照してください。

### 呼び出し方 {#methods-of-invocation}

**proof 関数として呼び出す。**
`integer_ring` ソルバーは、[proof 関数](./reference-proof-signature.html)に `by(integer_ring)` を付けて呼び出せます。

```
proof fn example(...) by(integer_ring)
    requires P
    ensures Q
{ }
```

これは、`integer_ring` ソルバーで `P ==> Q` を証明します。

### 対応している述語 {#supported-predicates}

`integer_ring` モードが対応しているのは、基本的な算術演算（`+`、`-`、`*`、`%`、`/`）と、
比較の `==` と `!=` だけです。

Verus は、すべての除数がゼロでないことを `requires` 節が述べているかどうかを検査します。

### ソルバーの動作 {#solver-operation}

与えられた述語は整数環の問題に帰着され、定理証明器 Singular によって証明されます。
{% endraw %}

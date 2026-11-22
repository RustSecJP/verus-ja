{% raw %}
# compute モード {#the-compute-mode}

> [!TIP]
> この機能を使うときの実践的なコツは、[ガイドのページ](./assert_by_compute.html)を参照してください。

### 呼び出し方 {#methods-of-invocation}

**アサーションで呼び出す。**
`compute` モードと `compute_only` モードは、[`assert-by` 文](./reference-assert-by-prover.html)で呼び出せます。

```
assert(Q) by(compute_only);
```

インタプリタで `Q` を簡約し、その結果が真偽値 `true` であるかどうかを検査することで、`Q` を証明します。

```
assert(Q) by(compute);
```

インタプリタで `Q` を式 `Q'` に簡約し、この文を `assert(Q');` に置き換えることで、`Q` を証明します。

### インタプリタの動作 {#interpreter-operation}

インタプリタは、関数の定義を展開し、算術式を評価します。
ある程度の記号的な操作はできますが、`a + b == b + a` のような代数法則は扱えません。
定数式を評価するときに、もっともうまく働きます。

なお、インタプリタはローカル変数を代入で置き換えることは**しません**。ローカル変数は記号的な値として扱われます。

次の文は、

```
assert(P) by(compute);
```

まず上のとおりにインタプリタを実行しますが、それで成功しなかった場合は、通常のソルバーを使って問題を片付けようとします。
そのため、たとえば、展開したあとの `P` が `a+b == b+a` のような自明な式になるのであれば、
それは `by(compute)` で解けるはずです。

### 評価戦略の設定 {#configuring-the-evaluation-strategy}

[`#[verifier::memoize]` 属性](./reference-attributes.html#verifiermemoize)を使うと、
特定の関数を[メモ化](https://en.wikipedia.org/wiki/Memoization)の対象として印を付けられます。
これにより、Verus の内部のインタプリタは、引数の組み合わせ 1 つにつき、その関数を 1 回だけ評価するようになります。
これは、次の例のように、素朴に評価するのが現実的でない関数に対して役に立ちます。

```rust
#[verifier::memoize]
spec fn fibonacci(n: nat) -> nat
    decreases n
{
    if n == 0 {
        0
    } else if n == 1 {
        1
    } else {
        fibonacci((n - 2) as nat) + fibonacci((n - 1) as nat)
    }
}

proof fn test_fibonacci() {
    assert(fibonacci(63) == 6557470319842) by(compute_only);
}
```
{% endraw %}

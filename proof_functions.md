{% raw %}
# proof 関数 {#proof-functions}

前の節の `pub closed spec` な `min` 関数を考えてみましょう。
これは、`min` の内部の定義をほかのモジュールに明かすことなく、抽象的な `min` 関数を定義していました。
しかし、抽象的な関数の定義は、その関数について何かを言えなければ役に立ちません。
そのために使えるのが `proof` 関数です。
一般に、`proof` 関数は、仕様についての性質を明かしたり証明したりします。
この例では、`lemma_min` という名前の `proof` 関数を定義します。
この関数は、`min` の正確な定義を明かすことなく、`min` についての性質を明かします。
具体的には、`lemma_min` は、`min(x, y)` が `x` か `y` のどちらかに等しく、`x` と `y` のどちらよりも大きくないことを明かします。

```rust
mod M1 {
    use verus_builtin::*;

    pub closed spec fn min(x: int, y: int) -> int {
        if x <= y {
            x
        } else {
            y
        }
    }

    pub proof fn lemma_min(x: int, y: int)
        ensures
            min(x, y) <= x,
            min(x, y) <= y,
            min(x, y) == x || min(x, y) == y,
    {
    }
}

mod M2 {
    use verus_builtin::*;
    use crate::M1::*;

    proof fn test() {
        lemma_min(10, 20);
        assert(min(10, 20) == 10); // succeeds
        assert(min(100, 200) == 100); // FAILS
    }
}
```

`exec` 関数と同じように、`proof` 関数は `requires` 節と `ensures` 節を持てます。
`exec` 関数とは違い、`proof` 関数はゴーストであり、実行可能コードにはコンパイルされません。
上の例では、`lemma_min(10, 20)` 関数を使って、モジュール `M2` の関数 `test` が `min(10, 20)` についてのアサーションを証明するのを助けています。
`min` が `closed` であるために、`M2` から `min` の内部の定義が見えなくても、証明できます。
一方、`min(100, 200)` についてのアサーションは、`test` が `lemma_min(100, 200)` も呼び出さないかぎり、やはり失敗します。

# proof ブロック {#proof-blocks}

結局のところ、`spec` 関数と `proof` 関数の目的は、`exec` 関数の中の実行可能コードについての性質を証明するのを助けることです。
実際、`exec` 関数は、*proof ブロック*の中に `proof` コードの断片を含められます。
proof ブロックは `proof { ... }` と書きます。
`proof` 関数が `proof` コードを含むのとちょうど同じように、`exec` 関数の中の `proof` ブロックは `proof` コードを含みます。
そして、`int` 型や `nat` 型のような、`proof` 関数で使えるゴーストコードの機能をすべて使えます。

アサーションの中で変数を導入していた、[以前の例](integers.html#integer-constants)を考えてみましょう。

```rust
fn test_consts_infer() {
    let u: u8 = 1;
    assert({
        let i: int = 2;
        let n: nat = 3;
        0 <= u < i < n < 4
    });
}
```

proof ブロックを使うと、これをもっと自然な書き方にできます。

```rust
fn test_consts_infer() {
    let u: u8 = 1;
    proof {
        let i: int = 2;
        let n: nat = 3;
        assert(0 <= u < i < n < 4);
    }
}

```

ここでは、`proof` ブロックの中の `proof` コードが、`int` 型と `nat` 型のローカル変数を作れます。
それらの変数は、その後のアサーションで使えます。
`proof` ブロック全体がゴーストコードなので、そのローカル変数を含めて、全体が実行可能コードへのコンパイルの前に消去されます。

proof ブロックは `proof` 関数を呼び出せます。
実際、`exec` 関数から `proof` 関数への呼び出しはすべて、`proof` ブロックのような `proof` コードの中に現れなければなりません。
`exec` 関数の `exec` コードから直接呼び出すことはできません。
これにより、どのコードが実行可能で、どのコードがゴーストなのかが、コンパイラにとっても、コードを読むプログラマーにとっても明確になります。
下に示す exec 関数 `test` では、`proof` ブロックを使って `lemma_min` を呼び出しています。
これにより、その後の `min` についてのアサーションが成功します。

```rust
mod M1 {
    use verus_builtin::*;

    pub closed spec fn min(x: int, y: int) -> int {
        if x <= y {
            x
        } else {
            y
        }
    }

    pub proof fn lemma_min(x: int, y: int)
        ensures
            min(x, y) <= x,
            min(x, y) <= y,
            min(x, y) == x || min(x, y) == y,
    {
    }

}

mod M2 {
    use verus_builtin::*;
    use crate::M1::*;

    fn test() {
        proof {
            lemma_min(10, 20);
            lemma_min(100, 200);
        }
        assert(min(10, 20) == 10);  // succeeds
        assert(min(100, 200) == 100);  // succeeds
    }

}

```

# assert-by {#assert-by}

前の例で、`test` が `min` について得た情報は、`proof` ブロックの中に閉じ込められていないことに注目してください。
その情報は `proof` ブロックの終わりを越えて伝わり、その後のアサーションの証明を助けます。
これは役に立つことがよくあります。
とくに、`proof` ブロックが、その後の `exec` 関数の呼び出しの事前条件を証明するのを助ける場合です。
`exec` 関数の呼び出しは、`proof` ブロックの外に現れなければならないからです。

しかし、ときには、特定の目的のためだけに情報を証明すれば足りることがあります。
その場合、得られた情報の有効範囲（スコープ）を限定すると、コードの構造が明確になります。
このため、Verus は `assert(...) by { ... }` 式に対応しています。
この式では、`by { ... }` ブロックの中に `proof` コードを書けます。
そのコードの唯一の目的は、`assert(...)` の中でアサートされた式を証明することです。
`proof` コードの中で得られたそれ以外の情報は、ブロックのスコープに限定され、`assert(...) by { ... }` 式の外には伝わりません。

下の例では、ブロックの中の `proof` コードが、`lemma_min(10, 20)` と `lemma_min(100, 200)` の両方を呼び出しています。
1 つ目の呼び出しは、`assert(...) by { ... }` 式の中の `min(10, 20) == 10` を証明するのに使われます。
これが証明されると、その後のアサーション `assert(min(10, 20) == 10);` は成功します。
しかし、アサーション `assert(min(100, 200) == 100);` は失敗します。
`lemma_min(100, 200)` の呼び出しで得られた情報は、その呼び出しを含むブロックの外には伝わらないからです。

```rust
mod M1 {
    use verus_builtin::*;

    pub closed spec fn min(x: int, y: int) -> int {
        if x <= y {
            x
        } else {
            y
        }
    }

    pub proof fn lemma_min(x: int, y: int)
        ensures
            min(x, y) <= x,
            min(x, y) <= y,
            min(x, y) == x || min(x, y) == y,
    {
    }
}

mod M2 {
    use verus_builtin::*;
    use crate::M1::*;

    fn test() {
        assert(min(10, 20) == 10) by {
            lemma_min(10, 20);
            lemma_min(100, 200);
        }
        assert(min(10, 20) == 10); // succeeds
        assert(min(100, 200) == 100); // FAILS
    }
}
```
{% endraw %}

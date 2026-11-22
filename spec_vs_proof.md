{% raw %}
# spec 関数と proof 関数の比較 {#spec-functions-vs-proof-functions}

`spec` 関数と `proof` 関数の両方を見てきたので、両者の違いをもう少しじっくり見てみましょう。
違いは、次の表にまとめられます
（参考のために、`exec` 関数も表に含めています）。

|                              | spec 関数         | proof 関数       | exec 関数        |
|------------------------------|-------------------|------------------|------------------|
| コンパイルされるか、ゴーストか | ゴースト          | ゴースト         | コンパイルされる |
| コードの書き方               | 純粋関数型        | 変更が許される   | 変更が許される   |
| `spec` 関数を呼び出せる      | はい              | はい             | はい             |
| `proof` 関数を呼び出せる     | いいえ            | はい             | はい             |
| `exec` 関数を呼び出せる      | いいえ            | いいえ           | はい             |
| 本体の可視性                 | 見える場合がある  | 決して見えない   | 決して見えない   |
| 本体                         | 本体は省略可能    | 本体は必須       | 本体は必須       |
| 決定性                       | 決定的            | 非決定的         | 非決定的         |
| 事前条件 / 事後条件          | recommends        | requires/ensures | requires/ensures |

[spec 関数](spec_functions.html)の節で説明したように、`spec` 関数は、その本体を同じモジュール内のほかの関数から見えるようにします。
さらに、必要であれば、ほかのモジュールからも本体が見えるようにできます。
`spec` 関数は、本体を完全に省略することもできます。

```
spec fn f(i: int) -> int;
```

このような[未解釈関数（uninterpreted function）](https://microsoft.github.io/z3guide/docs/logic/Uninterpreted-functions-and-constants)は、抽象的で未解釈の関数を、その関数についての信頼された公理とともに定義するライブラリで役に立つことがあります。

## 決定性 {#determinism}

`spec` 関数は決定的です。
つまり、同じ引数を与えると、常に同じ結果を返します。
コードは、関数の本体が見えない場合でも、この決定性を利用できます。
たとえば、下のコードでは、アサーション `x1 == x2` が成功します。
`x1` と `x2` はどちらも `s(10)` に等しく、`s` は `spec` 関数なので、`s(10)` は常に同じ結果を生み出すからです。

```rust
mod M1 {
    use verus_builtin::*;

    pub closed spec fn s(i: int) -> int {
        i + 1
    }

    pub proof fn p(i: int) -> int {
        i + 1
    }
}

mod M2 {
    use verus_builtin::*;
    use crate::M1::*;

    proof fn test_determinism() {
        let s1 = s(10);
        let s2 = s(10);
        assert(s1 == s2); // succeeds

        let p1 = p(10);
        let p2 = p(10);
        assert(p1 == p2); // FAILS
    }
}
```

これに対して、proof 関数 `p` は、原理的には、呼び出されるたびに異なる結果を返すことが許されています。
そのため、アサーション `p1 == p2` は失敗します。
（非決定性は、入出力の操作を行ったり乱数を扱ったりする `exec` 関数ではよくあることです。
実際には、`proof` 関数が非決定的に振る舞うことは珍しいでしょうが、許されてはいます。）

## recommends {#recommends}

`exec` 関数と `proof` 関数は、`requires` 節と `ensures` 節を持てます。
これに対して、`spec` 関数は、`requires` 節と `ensures` 節を持てません。
これは [Boogie](https://github.com/boogie-org/boogie) の仕組みに似ていますが、[Dafny](https://github.com/dafny-lang/dafny) や [F*](https://github.com/FStarLang/FStar) のようなほかのシステムとは異なります。
requires と ensures を許さない理由は、Verus の仕様言語を SMT ソルバーの数学的な言語に近いものに保ち、SMT ソルバーをできるだけ効率よく使うためです（[Verus の概要](overview.html)を参照してください）。

とはいえ、`spec` 関数に何らかの事前条件があると役に立つこともあります。
仕様の誤りを早い段階で見つけたり、`spec` 関数のうっかりした誤用を見つけたりするのに役立つからです。
そこで、`spec` 関数は `recommends` 節を含められます。
これは `requires` 節に似ていますが、厳格な要求ではなく、軽い推奨を表すにすぎません。
たとえば、次の関数では、呼び出し側には `i > 0` という推奨に従う義務は一切ありません。

```rust
spec fn f(i: nat) -> nat
    recommends
        i > 0,
{
    (i - 1) as nat
}

proof fn test1() {
    assert(f(0) == f(0));  // succeeds
}

```

`test1` が `f(0)` を呼び出すことは完全に正当で、`f` についてエラーも警告も生じません
（実際、Verus はこの推奨をまったく検査しません）。
ただし、関数の中に検証エラーが*あった場合には*、Verus は推奨の検査を有効にして、自動的に検証をやり直します。
推奨に反している箇所が、検証の失敗の原因を突き止める助けになることを期待してのことです。
たとえば、次のコードを見てください。

```rust
proof fn test2() {
    assert(f(0) <= f(1)); // FAILS
}
```

Verus は、失敗したアサーションをエラーとして表示し、続いて、満たされなかった推奨を注記（note）として表示します。

```
error: assertion failed
    |
    |     assert(f(0) <= f(1)); // FAILS
    |            ^^^^^^^^^^^^ assertion failed

note: recommendation not met
    |
    |     recommends i > 0
    |                ----- recommendation not met
...
    |     assert(f(0) <= f(1)); // FAILS
    |            ^^^^^^^^^^^^
```

この注記が役に立たなければ、プログラマーは無視してかまいません。

既定では、Verus は、`spec` 関数からの呼び出しについては `recommends` の検査を行いません。

```rust
spec fn caller1() -> nat {
    f(0)  // no note, warning, or error generated

}

```

ただし、`spec(checked)` と書けば、`recommends` の検査を要求できます。
こうすると、Verus は `recommends` への違反について警告を出します。

```rust
spec(checked) fn caller2() -> nat {
    f(0)  // generates a warning because of "(checked)"

}

```

これは、外部の検証されていない部品とのインターフェースを記述する、「信頼される計算基盤」（trusted computing base）の一部となる仕様で、とくに役に立ちます。
{% endraw %}

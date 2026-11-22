{% raw %}
# モジュール、隠蔽、opaque、reveal {#modules-hiding-opaque-reveal}

検証がタイムアウトする原因として考えられるものの 1 つは、とくに複雑な関数定義や、問題のある量化子を含む関数定義を展開してしまうことです。

自動的な展開を防ぐには、`opaque` を使って、関数の本体を検証器から隠すことができます。そのうえで `reveal` を使えば、検証器に定義を展開させたい箇所でだけ、関数の本体を選んで「明かす」ことができます。

`opaque` と `reveal` の使い方を示す小さな例を次に示します。

```rust
mod M1 {
    use verus_builtin::*;

    #[verifier::opaque]
    spec fn min(x: int, y: int) -> int {
        if x <= y {
            x
        } else {
            y
        }
    }

    fn test() {
        assert(min(10, 20) == min(10, 20)); // succeeds
        assert(min(10, 20) == 10); // FAILS
        reveal(min);
        assert(min(10, 20) <= 10); // succeeds
    }

}
```

これは、[前の節](spec_functions.html)で説明した `closed spec` 関数とよく似ています。主な違いは、`opaque` と `reveal` のほうが柔軟であることです。`opaque` は現在のモジュールの中でも関数の本体を隠すので、`reveal` を使って、特定の証明ブロックの中でだけ関数の本体を選んで公開できます。

一般に、現在のモジュールの中で関数の本体を使えるようにしておき、その仕様についての証明関数を同じモジュールの中で組み立てたいのであれば、`closed spec` を使うのがよいでしょう。そうすれば、モジュールの外で必要になるのは、この `spec` 関数に関する公開の証明関数だけになります。したがって、`open` や `closed` の spec 関数は抽象化に適しています。一方 `opaque` は、モジュール性のためというより、自動化と検証の性能を制御するための仕組みです。

関数の本体を隠す、より高度な使い方は、[リファレンス](reference-reveal-hide.html)で確認できます。
{% endraw %}

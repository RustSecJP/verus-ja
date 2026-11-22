{% raw %}
# クロージャ {#closures}

前の章では、関数を値として渡す方法を見ました。そこでは、関数アイテムを名前で参照していました。しかし Rust では、_クロージャ_ を使って関数を作るほうが一般的です。

## クロージャの事前条件と事後条件 {#preconditions-and-postconditions-on-a-closure}

Verus では、ほかの関数と同じように、クロージャにも `requires` と `ensures` を指定できます。次は、[前の章](./exec_funs_as_values.html)で定義した `vec_map` 関数を呼び出す例です。

```rust
fn test_vec_map_with_closure() {
    let double = |x: u8| -> (res: u8)
        requires 0 <= x < 128
        ensures res == 2 * x
    {
        2 * x
    };

    assert(forall |x| 0 <= x < 128 ==> call_requires(double, (x,)));
    assert(forall |x, y| call_ensures(double, (x,), y) ==> y == 2 * x);

    let mut v = Vec::new();
    v.push(0);
    v.push(10);
    v.push(20);
    let w = vec_map(&v, double);
    assert(w[2] == 40);
}
```

## クロージャのキャプチャ {#closure-capturing}

一般に、クロージャで最も難しい点の 1 つは、クロージャがまわりの文脈から変数をキャプチャできることです。Rust は、`FnOnce`、`FnMut`、`Fn` という関数トレイトの階層によって、この難しさを解決しています。クロージャの宣言と、文脈のキャプチャの詳細によって、そのクロージャがどのトレイトを持つかが決まります。そして、そのトレイトによって、呼び出し側に何ができるかが決まります。2 回以上呼び出せるのか、並列に呼び出せるのか、といったことです。

より詳しい入門は、[Rust のドキュメント](https://doc.rust-lang.org/book/ch13-01-closures.html#moving-captured-values-out-of-closures-and-the-fn-traits)を参照してください。

簡単に言うと、それぞれのトレイトは、呼び出し側に次の能力を与え、文脈のキャプチャに次の制限を課します。

|          | 呼び出し側の能力                             | キャプチャ                              |
|----------|----------------------------------------------|-----------------------------------------|
| `FnOnce` | 1 回だけ呼び出せる                           | 文脈から変数をムーブできる              |
| `FnMut`  | `&mut` 参照を通じて何回でも呼び出せる        | 文脈から _可変_ で借用できる            |
| `Fn`     | `&` 参照を通じて何回でも呼び出せる           | 文脈から _不変_ で借用できる            |

Verus は、文脈からの可変の借用にはまだ対応していません。一方、ムーブと不変の借用は簡単に扱えます。そのため、Verus は `Fn` と `FnOnce` のほうをよくサポートしています。Rust の `FnMut` が備えるキャプチャの能力は、まだ活用していません。

幸い、ムーブによるキャプチャと不変参照によるキャプチャは、どちらも簡単に扱えます。クロージャの内側でのそれらの値は、クロージャ式のあるプログラム地点での値そのものだと見なせばよいからです。

例:

```rust
fn example_closure_capture() {
    let x: u8 = 20;

    let f = || {
        // Inside the closure, we have seamless access to
        // variables defined outside the closure.
        assert(x == 20);
        x
    };
}
```
{% endraw %}

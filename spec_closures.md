{% raw %}
# spec クロージャ {#spec-closures}

Verus は、ゴーストコードの中での無名関数（Rust では「クロージャ」と呼ばれます）に対応しています。
たとえば、[この章](spec_lib.html)の前のほうに出てきた次のコードは、
無名関数 `|i: int| 10 * i` を使って、
列を 0、10、20、30、40 という値で初期化しています。

```rust
proof fn test_seq2() {
    let s: Seq<int> = Seq::new(5, |i: int| 10 * i);
    assert(s.len() == 5);
    assert(s[2] == 20);
    assert(s[3] == 30);
    assert(s[2..4] == seq![20, 30]);
    assert(s[2..] == seq![20, 30, 40]);
    assert(s[..4] == seq![0, 10, 20, 30]);
}
```

無名関数 `|i: int| 10 * i` の型は `spec_fn(int) -> int` で、
モードは `spec` です。
モードが `spec` なので、
この無名関数には、名前の付いた `spec` 関数と[同じ制限](modes.html)がかかります。
（たとえば、ほかの `spec` 関数は呼び出せますが、`proof` 関数や `exec` 関数は呼び出せません。）

標準的な実行可能な
[Rust のクロージャ](https://doc.rust-lang.org/book/ch13-01-closures.html)では
`Fn`、`FnOnce`、`FnMut` がトレイトであるのに対し、
`spec_fn(int) -> int` はトレイトではなく型であることに注意してください。
そのため、ゴーストコードは、
`spec_fn(t1, ..., tn) -> tret` 型の戻り値を使って、
spec クロージャを直接返せます。
標準的な実行可能な Rust のクロージャのように
[dyn や impl](https://doc.rust-lang.org/book/ch19-05-advanced-functions-and-closures.html#returning-closures) を
使う必要はありません。
たとえば、次に示す `spec` 関数 `adder` は、
`x` を `y` に足す無名関数を返せます。

```rust
spec fn adder(x: int) -> spec_fn(int) -> int {
    |y: int| x + y
}

proof fn test_adder() {
    let f = adder(10);
    assert(f(20) == 30);
    assert(f(60) == 70);
}
```
{% endraw %}

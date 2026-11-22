{% raw %}
# ゴーストコードと exec コード {#ghost-code-vs-exec-code}

`exec` コードの目的は、物理的に実在する値を操作することです。
つまり、プログラムが動くときに、物理的な電子回路の中に存在する値です。
一方、ゴーストコードの目的は、`exec` コードが操作する値*について語る*ことにすぎません。
ある意味で、これはゴーストコードに超自然的な能力を与えます。
ゴーストコードは、実行時に物理的には実装できないものについて語れるのです。
その一例は、すでに `int` 型と `nat` 型で見ました。
これらはゴーストコードの中でしか使えません。
別の例として、ゴーストコードは、ゼロによる除算の結果について語れます。

```rust
fn divide_by_zero() {
    let x: u8 = 1;
    assert(x / 0 == x / 0); // succeeds in ghost code
    let y = x / 0; // FAILS in exec code
}
```

これは単に、SMT ソルバーが、ゼロによる除算の結果を[未規定の整数値として](https://microsoft.github.io/z3guide/docs/theories/Arithmetic/#division)推論してくれることを反映しています。
これに対して、`exec` コードがゼロで割ろうとすると、Verus は検証の失敗を報告します。

```
error: possible division by zero
    |
    |     let y = x / 0; // FAILS
    |             ^^^^^
```

ゴーストコードの能力[^note_tracked]のうち、とくに次の 2 つは覚えておく価値があります。
- ゴーストコードは、どんな型の値でもコピーできます。
  その型が Rust の `Copy` トレイトを実装していなくてもかまいません。
- ゴーストコードは、どんな型の値でも作れます[^note_uninhabited]。
  その型に公開のコンストラクターが無くてもかまいません
  （たとえば、その型が、フィールドがすべて別のモジュールの非公開になっている構造体であってもです）。

たとえば、次の `spec` 関数は、`S` 型の値を作ったり複製したりします。
`S` は別のモジュールで定義されていて、フィールドは非公開で、`Copy` トレイトも持っていません。

```rust
mod MA {
    // does not implement Copy
    // does not allow construction by other modules
    pub struct S {
        private_field: u8,
    }

}

mod MB {
    use verus_builtin::*;
    use crate::MA::*;

    // construct a ghost S
    spec fn make_S() -> S;

    // duplicate an S
    spec fn duplicate_S(s: S) -> (S, S) {
        (s, s)
    }

}

```

これらの操作は、`exec` コードでは許されません。
さらに、ゴーストコードの値が `exec` コードに漏れ出すことも許されません。
ゴーストコードの中で起きたことは、ゴーストコードの中にとどまります。
ゴーストコードの値を `exec` コードで使おうとすると、必ずコンパイル時のエラーになります。

```rust
fn test(s: S) {
    let pair = duplicate_S(s); // FAILS
}
```

```
error: cannot call function with mode spec
    |
    |         let pair = duplicate_S(s); // FAILS
    |                    ^^^^^^^^^^^^^^
```

これらの能力を使うゴーストコードの例として、Verus の [`Seq::index(...)` 関数](https://github.com/verus-lang/verus/blob/main/source/vstd/seq.rs)の呼び出しがあります。
この呼び出しは、添字 `i` が範囲内であれば列の値を複製でき、`i` が範囲外であれば何も無いところから値を作り出せます。

```rust
impl<A> Seq<A> {
...
    /// Gets the value at the given index `i`.
    ///
    /// If `i` is not in the range `[0, self.len())`, then the resulting value
    /// is meaningless and arbitrary.

    pub spec fn index(self, i: int) -> A
        recommends 0 <= i < self.len();
...
}
```

## spec コードから exec コードを作る（およびその逆） {#producing-exec-code-from-spec-code-and-vice-versa}

Verus コミュニティからのさまざまな貢献のおかげで、Verus は、場合によっては、spec 関数を実装していることが証明できる [exec 関数を自動的に作り出せます](exec_spec.html)。
逆に、場合によっては、exec 関数から [spec 関数を自動的に作り出す](exec_to_spec.html)こともできます。


---

[^note_tracked]: `proof` コードの中の変数は、[`tracked` 注釈](reference-var-modes.html#using-tracked-and-ghost-variables-from-a-proof-function)を使って、これらの特別な能力を使わないことを選べます。
ただし、これは高度な機能なので、今のところは無視してかまいません。

[^note_uninhabited]: これは、Rust の `!`（「never」型）のように、その型が `exec` コードでは値を持たない場合でも成り立ちます
（[この技術的な議論](https://github.com/Chris-Hawblitzel/rust/wiki/Three-kinds-of-code-...-specification,-proof,-and-executable)に出てくる「bottom」の値を参照してください）。
{% endraw %}

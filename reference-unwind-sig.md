{% raw %}
# 巻き戻しのシグネチャ {#unwinding-signature}

`exec` モードのどの関数についても、その関数が[巻き戻し（unwind）](https://doc.rust-lang.org/nomicon/unwinding.html)を起こしうるかどうかを指定できます。シグネチャとして書ける形は、次のとおりです。

 * シグネチャなし（既定）- 関数は巻き戻しを起こすことがある、という意味です。
 * `no_unwind` - 関数は巻き戻しを起こさない、という意味です。
 * `no_unwind when {入力の引数についての真偽値の式}` - 与えられた条件が成り立つ*ならば*、その呼び出しは巻き戻しを起こさないことが保証されます。
    * `no_unwind when true` は `no_unwind` と等価です
    * `no_unwind when false` は既定の動作と等価です

既定では、関数は巻き戻しを起こしてよいことになっています。（ただし、関数のシグネチャが
技術的には巻き戻しを許している場合でも、Verus は、整数オーバーフローのような、
巻き戻しのよくある原因を*排除します*。）

## 例 {#example}

添字を受け取る関数を書きたいとします。そして、次のように仕様を定めたいとします。

 * 添字が範囲内であれば、関数は正常に実行される
 * そうでなければ、関数は巻き戻しを起こす

これは、次のように書けます。

```rust
fn get(&self, i: usize) -> (res: T)
    ensures i < self.len() && res == self[i]
    no_unwind when i < self.len()
```

これは、実質的に次のことを述べています。

 * `i < self.len()` であれば、関数は巻き戻しを起こさない。
 * 関数が正常に戻れば、`i < self.len()` である（言い換えると、`i >= self.len()` であれば、関数は巻き戻しを起こさなければならない）。

## 不変条件に関する制限 {#restrictions-with-invariants}

[不変条件](https://verus-lang.github.io/verus/verusdoc/vstd/macro.open_local_invariant.html)が開いている間は、巻き戻しを起こせません。
この制限が必要なのは、巻き戻しの操作がプログラムを異常終了（abort）させるとは限らないからです。
たとえば、Rust では、プログラムが[巻き戻しを「捕まえる」](https://doc.rust-lang.org/std/panic/fn.catch_unwind.html)ことができますし、実行を続けるほかのスレッドがあるかもしれません。
そのため、Verus は、不変条件を回復しないままプログラムが不変条件のブロックから途中で抜けることを、
たとえ巻き戻しのためであっても許せません。

この制限があるからこそ、Verus は[例外安全性の違反](https://doc.rust-lang.org/nomicon/exception-safety.html)を排除できます。

## Drop {#drops}

ある型に `Drop` を実装する場合は、それに `no_unwind` のシグネチャを与えることが求められます。
{% endraw %}

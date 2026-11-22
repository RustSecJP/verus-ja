{% raw %}
# verus! マクロ {#the-verus-macro}

[はじめに](./getting_started.html)の章に出てきたサンプルプログラムを思い出してください。

```rust
use vstd::prelude::*;

verus! {

spec fn min(x: int, y: int) -> int {
    if x <= y {
        x
    } else {
        y
    }
}

fn main() {
    assert(min(10, 20) == 10);
    assert(min(-10, -20) == -20);
    assert(forall|i: int, j: int| min(i, j) <= i && min(i, j) <= j);
    assert(forall|i: int, j: int| min(i, j) == i || min(i, j) == j);
    assert(forall|i: int, j: int| min(i, j) == min(j, i));
}

} // verus!
```

これはいったい何でしょうか。Verus でしょうか。それとも Rust でしょうか。

答えは、その両方です。これは、`verus!` マクロを使って Verus を埋め込んだ Rust のソースファイルです。
具体的には、`verus!` マクロは Rust の構文を拡張し、検証に関する機能を加えます。
事前条件、事後条件、アサーション、`forall`、`exists` などです。
これらについては、このチュートリアルで詳しく学んでいきます。

クレート内の各ファイルは、ふつう次のような形になります。

```rust
use vstd::prelude::*;

verus! {
    // ...
}
```

`vstd::prelude` は、`verus!` マクロと、そのほかの Verus の便利な機能をいくつか公開しています。

`verus!` マクロは、Rust の構文を拡張するだけでなく、*中に含まれる関数を検証するよう Verus に指示*します。
既定では、Verus は `verus!` マクロの内側にあるものをすべて検証し、`verus!` マクロの外側で定義されたものはすべて無視します。
この動作を変えるための属性や指示がいろいろあります（たとえば[この章](./interacting-with-unverified-code.html)を参照してください）。
ただし、このチュートリアルの大部分では、すべてのコードを、`verus!` マクロの中に置かなければならない Verus のコードとして扱います。

**チュートリアルについての注意。**
このガイドの以降の部分では、例が煩雑にならないよう、これらの宣言を省略します。
ただし、どの例のコードも `verus! { ... }` ブロックの中に置く必要があること、そしてファイルに `use vstd::prelude::*;` が含まれている必要があることを忘れないでください。

**別の構文。**
Verus は、これとは別の、[属性に基づく構文](exec_attr.html)にも対応しています。
この構文は、既存の Rust プロジェクトへの変更を最小限に抑えたいときに役立つことがあります。
しかし、`verus!` の構文のほうがすっきりしていて単純なので、このチュートリアルではそちらを使い続けます。
{% endraw %}

{% raw %}
# コマンドラインで始める {#getting-started-on-the-command-line}

## 1. Verus をインストールする
{: id="1-install-verus"}

Verus をインストールするには、[INSTALL.md](https://github.com/verus-lang/verus/blob/main/INSTALL.md) の手順に従ってください。

## 2. サンプルプログラムを検証する
{: id="2-verify-a-sample-program"}

`getting_started.rs` という名前のファイルを作り、次の内容を貼り付けます。

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

このファイルに対して Verus を実行するには、次のようにします。

macOS、Linux、またはそれに類するシステムでは、次を実行します。

```
/path/to/verus getting_started.rs
```

Windows では、次を実行します。

```
.\path\to\verus.exe getting_started.rs
```

次のような出力が表示されるはずです。

```
note: verifying root module

verification results:: 1 verified, 0 errors
```

これは、Verus が 1 個の関数（`main` 関数）の検証に成功したことを示しています。

### 検証に通らないコードで試す {#try-it-on-code-that-wont-verify}

よければ、`getting_started.rs` ファイルを編集して、検証の失敗を見てみてください。
たとえば、`main` に次の行を追加します。

```
    assert(forall|i: int, j: int| min(i, j) == min(i, i));
```

すると、次のエラーメッセージが表示されます。

```
note: verifying root module

error: assertion failed
  --> getting_started.rs:19:12
   |
19 |     assert(forall|i: int, j: int| min(i, j) == min(i, i));
   |            ^^^^^^ assertion failed

error: aborting due to previous error

verification results:: 0 verified, 1 errors
```

## 3. プログラムをコンパイルする
{: id="3-compile-the-program"}

上のコマンドはコードを検証するだけで、コンパイルはしません。
バイナリにコンパイルもしたい場合は、`--compile` フラグを付けて `verus` を実行します。

macOS、Linux、またはそれに類するシステムでは、次を実行します。

```
/path/to/verus getting_started.rs --compile
```

Windows では、次を実行します。

```
.\path\to\verus.exe getting_started.rs --compile
```

どちらの場合も、`getting_started` というバイナリが作られます。

ただし、この例では、バイナリは何も面白いことをしません。
`main` 関数に実行可能コードが含まれていないからです。
含まれているのは静的に検査されるアサーションだけで、それらはコンパイルの前に消去されます。

## 4. Verus についてさらに学ぶ
{: id="4-learn-more-about-verus"}

[チュートリアルに進みましょう](./verus_macro_intro.html)。まずは、上の例に出てきた `verus!` マクロの説明から始まります。
{% endraw %}

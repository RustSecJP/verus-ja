{% raw %}
# ゴースト消去 {#ghost-erasure}

Verus はゴースト消去（ghost erasure）を行います。
検証のために存在するゴーストコード（ghost code）は、実行可能な成果物をビルドするときに取り除かれ、成果物への影響が最小限になるようにされます。

その副産物の 1 つとして、検証時には存在する識別子の一部が、コンパイル時には存在しなくなります。
つまり、以下のコードは _コンパイル_ に失敗します。

```rust
use vstd::prelude::*;

verus! {

pub mod ghost_mod {
    pub closed spec fn ghost_fn() -> bool { true }
}

pub mod test_mod {
    use crate::ghost_mod::ghost_fn;
//  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//  FAILS: During compilation, ghost_fn is erased, causing rustc
//         to complain about a missing definition

    pub fn exec_fn() -> u64 {
        1
    }
}

} // verus!
```

これを解決するために、Verus は `verus_only` フラグを提供しています。
このフラグは、検証中は有効になり、それ以外のときは無効になります。
これを使うと、上のような `use` 文を保護できます。

```rust
pub mod test_mod {
    #[cfg(verus_only)]
    use crate::ghost_mod::ghost_fn;
//  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//  OK: During compilation, the `use` statement is removed
}
```

`verus_only` フラグのもう 1 つの使い道は、検証中にしか意味を持たない属性を設定することです。
たとえば、以下のコードは、`verus_only` が設定されている場合にだけ、そのファイルの `verifier::loop_isolation` オプションを false に設定します。
```rust
#![cfg_attr(verus_only, verifier::loop_isolation(false))]
```


> [!CAUTION]
> これは、`use` 文の保護と、設定用の属性の指定にだけ使うべきです。
> この `cfg` フラグをコードの条件付きコンパイルに使うと、**不健全性** を持ち込み、検証の保証を壊すおそれがあります。
>
> 例として、以下のコードは検証に成功しますが、実行ファイルを動かすと `f()` は 42 を返します。
>
> ```rust
> fn f() -> (u: u32)
>   ensures u != 0
> {
>   #[cfg(verus_only)]
>   { return 0; }
>   #[cfg(not(verus_only))]
>   { return 42; }
> }
> ```

> [!NOTE]
> Verus は _活発に開発されている_ ため、このオプションは変更されたり削除されたりする可能性があります
> （特に、この問題に対するより良い解決策が考え出された場合です）。
> 詳しくは、[GitHub の議論](https://github.com/verus-lang/verus/discussions/2101)を参照してください。

## `Cargo.toml` の設定 {#cargotoml-configuration}

[バージョン 1.80 以降](https://blog.rust-lang.org/2024/05/06/check-cfg/)、Rust は、すべての `cfgs` が想定された設定名と一致するかどうかを自動的に検査します。
`verus_only` という `cfg` フラグは、Verus が直接渡すもので、`Cargo.toml` で宣言されることはありません。
そのため、Rust はコンパイル時に警告を出します。
これを防ぐには、`Cargo.toml` に次の内容を追加します。

```toml
[lints.rust]
unexpected_cfgs = { level = "warn", check-cfg = [
  'cfg(verus_only)',
] }
```

> [!NOTE]
> `cargo verus new` でリポジトリを初期化した場合、これは自動的に行われます。
{% endraw %}

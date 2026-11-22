{% raw %}
# Cargo 経由で Verus を使う {#using-verus-via-cargo}

複数のクレートからなるプロジェクトのために、Verus は `cargo verus` サブコマンドを提供しています。
これは、Rust の通常の Cargo の作業の流れに検証を組み込みます。

## 前提条件 {#prerequisites}

`cargo-verus` バイナリが `PATH` に入っていなければなりません。
このバイナリは、公式の [Verus のリリースパッケージ](https://github.com/verus-lang/verus/releases)に、`verus` バイナリと並んで含まれています。

使える状態になっているかどうかは、次のコマンドを実行して確認します。

```
cargo verus --help
```

## 新しいプロジェクトを始める {#starting-a-new-project}

新しい Verus プロジェクトを作るいちばん速い方法は、次のとおりです。

```bash
# Binary project
cargo verus new --bin my_project

# Library project
cargo verus new --lib my_library
```

これは新しいディレクトリを作り、その中に、正しく設定された `Cargo.toml`、最初の `src/main.rs`（または `src/lib.rs`）、`.gitignore` を用意します。
生成されたプロジェクトには、依存としての `vstd` と、後述する `[package.metadata.verus]` セクションがすでに入っています。

## 既存の Rust プロジェクトを Verus に対応させる {#updating-an-existing-rust-project-to-support-verus}

プロジェクト内の 1 つ以上のクレートで検証を有効にするには、次のようにして `vstd` を依存に追加します。

```sh
cargo add vstd
```

次に、それぞれのルートモジュール（たとえば `lib.rs` や `main.rs`）で Verus のプレリュードをインポートします。

```rust
use vstd::prelude::*;
```

クレートを検証の対象にするには、その `Cargo.toml` に次の内容を入れます。

```toml
[package.metadata.verus]
verify = true
```

クレートがワークスペースに属していない場合は、`cfg(verus_only)` についての警告を抑えるために、次の内容も追加します。

```toml
[lints.rust]
unexpected_cfgs = { level = "warn", check-cfg = ['cfg(verus_only)'] }
```

詳細は後述します。

Verus の注釈を含むクレートやプロジェクトに対しても、通常の `cargo` コマンド（たとえば `cargo build`）を引き続き使えることに注意してください。

## Verus のプレリュード {#the-verus-prelude}

検証するクレートはそれぞれ、次のようにして Verus のプレリュードをインポートする必要があります。

```rust
use vstd::prelude::*;
```

## Cargo.toml の設定 {#cargotoml-configuration}

検証するべきクレートは、`Cargo.toml` に `[package.metadata.verus]` セクションを追加して、検証の対象になることを選ばなければなりません。

```toml
[package.metadata.verus]
verify = true
```

### `verus_only` リントを抑える {#suppressing-the-verus_only-lint}

Verus は、検証中に `verus_only` という cfg フラグを有効にします。
これにより、ゴーストのアイテムに対する `use` 文を `#[cfg(verus_only)]` で保護できます。
このフラグは `Cargo.toml` で宣言されていないため、抑えない限り、Rust 1.80 以降は警告を出します。

```toml
[lints.rust]
unexpected_cfgs = { level = "warn", check-cfg = ['cfg(verus_only)'] }
```

`cargo verus new` は、これを自動的に追加します。
`verus_only` と、それをいつ使うべきかについての詳しい説明は、[ゴースト消去](./erasure.html)を参照してください。

Cargo のワークスペースでは、代わりに、ルートの `Cargo.toml` に次の内容を入れるべきです。

```toml
[workspace.lints.rust]
unexpected_cfgs = { level = "warn", check-cfg = ['cfg(verus_only)'] }
```

そのうえで、それぞれのメンバークレートの `Cargo.toml` に次の内容があることを確かめます（これは、たとえば `cargo new` が自動的に追加します）。

```toml
[lints.rust]
workspace = true
```

## サブコマンド {#subcommands}

### `cargo verus verify` {#cargo-verus-verify}

`verify = true` を持つすべてのクレートを、依存も含めて検証します。
コンパイル済みのバイナリは **生成しません**。

```bash
cargo verus verify              # Verify all of the crates
cargo verus verify -p my_crate  # Verify only my_crate and its dependencies
```

日々の証明の開発には `verify` を使います。
インクリメンタルビルドにより、変更されたクレートだけが再検証されます。

### `cargo verus focus` {#cargo-verus-focus}

`verify` と似ていますが、依存の再検証を省きます。
ルートクレートだけを繰り返し手直ししていて、その依存が変わっていないときに便利です。

`verify` と違い、`focus` は成果物を `target/verus-partial` の下にある別のターゲットディレクトリに保存します。
これは、完全な `cargo verus verify` が、キャッシュされた部分的な結果に惑わされないようにするためです。

```bash
# Verify all of the root crates, but not their dependencies
cargo verus focus
# Verify my_crate, but not its dependencies
cargo verus focus -p my_crate
```

依存は引き続き *ビルド* されます（そのため、依存の型は利用できます）が、その証明は再検査されません。
プロジェクトの端から端までの証明が健全であることを確かめるために、コミットする前には `cargo verus verify` を使うべきです。

### `cargo verus build` {#cargo-verus-build}

検証の対象になることを選んだすべてのクレートを検証し、**さらに** ネイティブの成果物（ライブラリまたはバイナリ）にコンパイルします。
検証済みのバイナリを出荷したいときに使います。

```bash
cargo verus build
cargo verus build --release
```

Verus の注釈が付いたコードは、お好みであれば、通常の `cargo build` コマンドでもビルドできることに注意してください。


## 引数の渡し方 {#passing-arguments}

引数は `--` を境に分けられます。

- **`--` より前**: `cargo` に渡されます
- **`--` より後**: すべての `verus` の呼び出しに渡されます

```bash
cargo verus verify -p my_crate --release -- --rlimit 60 --expand-errors
#                  ^^^^^^^^^^^^^^^^^^^^^    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
#                  Cargo arguments            Verus arguments
```

Verus に関係する Cargo のオプションは、`--release` のような一般的な Cargo のフラグより前に置かなければなりません。
Verus に関係するオプションは次のとおりです。
- `-p`/`--package`、`--workspace`、`--all`、`--exclude`
- `--features`、`--all-features`、`--no-default-features`
- `--manifest-path`、`--target-dir`
- `--frozen`、`--locked`、`--offline`
- `--config`、`-Z`

`-Z` オプションは、`-Z unstable-options` のように、空白を入れて書かなければなりません。詰めて書く形式には対応していません。

この方法で渡せる、よく使う Verus の引数には、`--rlimit`、`--expand-errors`、`--log-all` などがあります。
クレートに固有の引数をこの方法で渡すことは、通常は避けるべきです。
たとえば、`--verify-module` や `--verify-function` を渡しても、おそらくうまくいきません。
うまくいくのは、プロジェクトとその依存に含まれる、検証するすべてのクレートに同じモジュールや関数がある場合だけです。
こうしたプロジェクトに固有のフラグを渡すには、どのクレートがどの Verus の引数を受け取るかを制御する方法を、以下で参照してください。

### どのクレートが Verus の引数を受け取るかを制御する {#controlling-which-crates-receive-verus-arguments}

既定では、`--` より後の Verus の引数は、検証する **すべての** クレートに渡されます。
対象を絞ったり広げたりするには、`--fwd-verus-args-to <target>` を使います。

| 値       | 効果                                                           |
|----------|----------------------------------------------------------------|
| `all`    | 依存を含め、検証するすべてのクレートに渡します（`verify`、`build` の既定） |
| `roots`  | ルートの（選択された）クレートにだけ渡します（`focus` の既定） |
| `deps`   | 依存のクレートにだけ渡し、ルートクレートには渡しません          |

```bash
# Only pass --rlimit to the root crate, not to its verified dependencies
cargo verus verify --fwd-verus-args-to roots -- --rlimit 60
```

標準の `cargo -p <crate_name>` を使って、操作を 1 つのクレート（と、場合によってはその依存）に限定することもできることを思い出してください。

## 複数クレートのワークスペース {#multi-crate-workspaces}

ワークスペースでは、検証するべきクレートがそれぞれ個別に、`Cargo.toml` ファイルに `verify = true` を追加して、検証の対象になることを選ばなければなりません。
この設定が無いクレートは通常どおりコンパイルされ、Verus の証明器には渡されません。

```bash
# Verify the entire workspace
cargo verus verify --workspace

# Verify a single crate and its verified dependencies
cargo verus verify -p my_crate
```

## インクリメンタル検証 {#incremental-verification}

cargo-verus は、`target` ディレクトリの中で、コンパイル済みの `.rlib` 成果物の隣に `.vir` ファイルを保存します。
Cargo のフィンガープリントの仕組みが、これらのファイルの変更を検出し、下流のクレートを自動的に再検証します。
`cargo clean` を実行すると、検証の成果物はすべて削除されます。
{% endraw %}

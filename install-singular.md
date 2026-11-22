{% raw %}
# Singular のインストールと設定 {#installing-and-configuring-singular}

[`integer_ring` ソルバーモード](./nonlinear.html#2-proving-ring-based-properties-integer_ring)を使うには、Singular がインストールされていなければなりません。

**手順:**

1. Singular をインストールする
   - Singular の標準ライブラリを使うには、Singular の実行可能バイナリだけでは足りません。
     そのため、可能であれば、システムのパッケージマネージャーを使うことを強くおすすめします。
     どの方法を選ぶ場合でも、Singular のバージョン 4.3.2 をインストールしてください。
     ほかのバージョンは試験されておらず、4.4.0 は Verus と互換性が無いことが分かっています。
     以下は、プラットフォームごとのおすすめの手順です。
     - Mac: `brew install Singular` を実行し、Verus を実行するときに環境変数 `VERUS_SINGULAR_PATH` を設定します（例: `VERUS_SINGULAR_PATH=/usr/local/bin/Singular`）。そのほかの選択肢については、Singular の [OS X 向けインストールガイド](https://www.singular.uni-kl.de/index.php/singular-download/install-os-x.html)を参照してください。

     - Debian 系の Linux: `apt-get install singular` を実行し、Verus を実行するときに環境変数 `VERUS_SINGULAR_PATH` を設定します（例: `VERUS_SINGULAR_PATH=/usr/bin/Singular`）。そのほかの選択肢については、Singular の [Linux 向けインストールガイド](https://www.singular.uni-kl.de/index.php/singular-download/install-linuxunix.html)を参照してください。

     - Windows: Singular の [Windows 向けインストールガイド](https://www.singular.uni-kl.de/index.php/singular-download/install-windows.html)を参照してください。

2. Singular 対応を有効にして Verus をコンパイルする
   - `integer_ring` の機能は、`singular` フィーチャーが設定されているときに条件付きでコンパイルされます。
     このフィーチャーを追加するには、Verus をコンパイルするために `cargo build` を実行するときに、`--features singular` フラグを付けます。
{% endraw %}

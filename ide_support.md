{% raw %}
# Verus の IDE 対応 {#ide-support-for-verus}

Verus は現在、VS Code と Emacs の IDE 対応を備えています。

VS Code では、`rust-analyzer` を Verus 向けにフォークした `verus-analyzer` が必要です。
VS Code で Verus を使うには、[verus-analyzer の README](https://github.com/verus-lang/verus-analyzer) の手順に従ってください。

Emacs では、Verus 単独での対応を用意しています。[Emacs](#quickstart-emacs) を使い始める手順は以下のとおりです。

## Emacs のクイックスタート {#quickstart-emacs}
Emacs での Verus プログラミングは、[verus-mode.el](https://github.com/verus-lang/verus-mode.el) を通じて対応しています。
これは、構文の強調表示、保存時の検証、定義へのジャンプなどに対応したメジャーモードです。

verus-mode を使うための設定は、`.emacs` で (i) `verus-home` に Verus へのパスを設定し、(ii) `verus-mode` を読み込む、というだけで済むこともあります。

たとえば、`use-package` を使っている場合は、Emacs が読み込める場所に [verus-mode.el](https://github.com/verus-lang/verus-mode.el) をクローンし、次のスニペットを追加します。
```
(use-package verus-mode
  :init
  (setq verus-home "PATH_TO_VERUS_DIR"))   ; Path to where you've cloned https://github.com/verus-lang/verus
```

Emacs の具体的な設定によって、verus-mode.el の正確なインストール手順は変わることがあります。
さまざまな Emacs の設定に応じた詳しいインストール手順は、[verus-mode.el の README の Install の節](https://github.com/verus-lang/verus-mode.el#install)に記載されています。

最新の機能、キーバインド、問題解決のヒントについて詳しくは、[verus-mode.el の README](https://github.com/verus-lang/verus-mode.el/blob/main/README.md) を確認してください。
{% endraw %}

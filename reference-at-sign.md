{% raw %}
# ビュー関数 `@` {#the-view-function-}

### 構文 {#syntax}

```verus-grammar
V@[view_expr] ::= V@[spec_expr] @
```

### 脱糖 {#desugaring}

式 `expr@` は式 `expr.view()` へと脱糖（desugar）されます。これは、Rust の[メソッド解決](https://doc.rust-lang.org/reference/expressions/method-call-expr.html)によって通常どおりに解決されます。
これは、つねにではありませんが、たいていの場合は [`View` トレイトで定義されている](https://verus-lang.github.io/verus/verusdoc/vstd/view/trait.View.html) `view` メソッドに解決されます。

### 例 {#examples}

慣習として、`view()` 関数は、exec モードのオブジェクトを抽象化したものを表す spec 関数です。
たとえば、次を参照してください。

 * [ビューを使って `Vec` について推論する](./exec_lib.html)
 * [ビュー関数を定義する](./container_bst_first_draft.html#the-abstract-view)
{% endraw %}

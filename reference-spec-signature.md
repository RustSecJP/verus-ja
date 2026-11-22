{% raw %}
# spec 関数のシグネチャ {#spec-fn-signature}

`spec` 関数のシグネチャの一般形は、次のとおりです。

```verus-grammar
V@[spec_fn_item] ::=
    R@[visibility]? uninterp? V@[openness]? spec fn R@[function_name] R@[generics]?(R@[args...]) -> R@[type]
        R@[where_clause]?
        V@[recommends_clause]?
        V@[spec_decreases_clause]?
    ( ; | { V@[spec_expr] } )

V@[openness] ::= closed
           | open
           | open ( R@[visibility] )
```

## `recommends` 節 {#the-recommends-clause}

`recommends` 節は「ゆるい事前条件」です。診断のために検査されることがありますが、
関数が well-defined であるための厳格な要件ではありません。

動機と概要については、[ガイドのこのページ](./spec_vs_proof.html#recommends)を参照してください。

## `decreases` 節 {#the-decreases-clause}

`decreases` 節は、再帰的な定義が整形式（well-formed）であることを保証するために使います。
なお、`decreases` 節に `when` の副節がある場合、関数の定義はその定義域に制限されます。

詳しくは [`decreases` のリファレンスページ](./reference-decreases.html)を参照してください。
動機と概要については、[再帰関数についてのガイドのページ](./recursion.html)を参照してください。

## 公開度の節 {#the-openness-clause}

公開度（openness）は、本体の可視性を定めます。本体の可視性は、関数名の可視性より狭くできます。具体的には、次のとおりです。

 * `open` は、本体がどこからでも、すべてのクレートから見えることを意味します。
 * <code>open(R@[visibility])</code> は、本体が、指定した可視性指定子の範囲で見えることを意味します。
   * 例: `open(crate)`、`open(self)`、`open(super)`、`open(in some::module::path)`
 * `closed` は、本体が、関数を定義したモジュールの中でだけ見えることを意味します。つまり、`open(self)` と等価です。

本体を与えるときは、公開度の指定子が必ず必要です。

## `uninterp` 指定子 {#the-uninterp-specifier}

`uninterp` 指定子は、関数を*未解釈*（uninterpreted）として宣言します。これは、その spec 関数の本体が
指定されていないことを意味します。

> [!NOTE]
> 未解釈関数は、ふつう、その関数の性質を定める公理と組み合わせて使わなければ役に立ちません。
> 未解釈関数のよくある使い道は、信頼された（つまり、検証されていない）ライブラリの型について、
> spec 上の解釈を定義することです。
>
> ただし、追加の公理なしに `uninterp` 関数を宣言することは、常に健全*です*。
{% endraw %}

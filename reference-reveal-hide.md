{% raw %}
# `reveal`、`reveal_with_fuel`、`hide` {#reveal-reveal_with_fuel-hide}

### 構文 {#syntax}

```verus-grammar
V@[reveal_stmt]           ::= reveal ( R@[path] ) ;
V@[reveal_with_fuel_stmt] ::= reveal_with_fuel ( R@[path] , R@[int_literal] ) ;
V@[hide_stmt]             ::= hide ( R@[path] ) ;
```

### 証明上の動作 {#proof-operation}

これらの属性は、Verus が求解の途中で spec 関数の定義を展開するかどうか、また、どのように展開するかを制御します。
spec 関数 `f` について、次のようになります。

 - `reveal(f)` は、`f` の使用箇所に出会ったときに `f` の定義を展開するよう Verus に指示します。
 - `hide(f)` は、`f` を未解釈関数（uninterpreted function）として扱い、
   その定義については推論しないよう Verus に指示します。

技術的に言えば、Verus は「関数の展開」を、
`forall |x| f(x) == (definition of f(x))` という形の公理を作ることで扱っています。
したがって、`reveal(f)` はこの公理をソルバーから使えるようにし、
`hide(f)` はこの公理をソルバーから使えないようにします。

既定では、関数はスコープに入っていれば常に reveal された状態になっています。
これは、関数に `#[verifier::opaque]` 属性を付けることで変えられます。

`reveal_with_fuel(f, n)` ディレクティブは、再帰関数に対して使います。
整数 `n` は、Verus が再帰関数を何回展開するかを表します。
燃料（fuel）を有限の量に制限することは、
[トリガーのループ](multitriggers.html#matching-loops-what-they-are-and-how-to-avoid-them)を避けるために必要です。
既定の燃料（`reveal_with_fuel` ディレクティブが無い場合）は 1 です。
{% endraw %}

{% raw %}
# FnOnce の仕様 {#specifications-on-fnonce}

どんな関数オブジェクト、つまり `FnOnce` を実装する任意の型の値（たとえば、名前付きの関数やクロージャ）についても、
Verus の組み込み関数 `call_requires` と `call_ensures` を使って、そのシグネチャを汎用的に推論できます。

 * `call_requires(f, args)` は、与えられた `args` で `f` を呼び出しても安全かどうかを表す述語です。静的でない呼び出しのすべてについて、Verus は、呼び出し箇所で `call_requires(f, args)` が満たされることを証明するよう開発者に求めます。
 * `call_ensures(f, args, output)` は、`args` で呼び出したときに `f` が与えられた `output` を返しうるかどうかを表す述語です。静的でない呼び出しのすべてについて、Verus は、呼び出し箇所のあとで `call_ensures(f, args, output)` が成り立つと仮定します。
 * 現時点では、シグネチャのうち `opens_invariants` の部分は汎用的には扱われません。Verus は保守的に、静的でない呼び出しはどれも、任意の不変条件を開くかもしれないものとして扱います。

`args` は、常にタプル（0 要素や 1 要素のタプルのこともあります）として与えます。

例やそのほかのコツについては、[チュートリアルの章](./exec_funs_as_values.html)を参照してください。

Verus のシグネチャを持つどの関数（名前付きの関数でもクロージャでも）についても、Verus は次のような公理を生成します。

```
(user-declared requires clause) ==> call_requires(f, args)
call_ensures(f, args, output) ==> (user-declared ensures clauses)
```

厳密な同値（`<==>`）ではなく含意（`==>`）を使っているのは、1 つには、
[トレイトでの柔軟なシグネチャ](./reference-signature-inheritance.html)を可能にするためです。
ただし、Verus の公理は、トレイトの関数に限らず、すべての関数についてこの形を使います。
この形は、[高階関数の仕様の正しい書き方](./exec_funs_as_values.html#an-important-note)を反映しています。
{% endraw %}

{% raw %}
# 〜を満たすもの（`choose`） {#such-that-choose}

直感に沿った入門は、[exists と choose](exists.html) を参照してください。

`choose` は、与えられた述語を満たす任意の値を返す、**spec モード専用**の式です。
これはヒルベルトの選択演算子（イプシロン演算子、または [such-that 演算子](https://en.wikipedia.org/wiki/Epsilon_calculus)とも呼ばれます）です。

### 構文 {#syntax}

```verus-grammar
V@[choose_expr] ::= choose |R@[binders...]| V@[spec_expr]
```

本体の V@[spec_expr] には、[トリガー注釈](./trigger-annotations.html)を含めることができます。

### 型付け {#typing}

本体の V@[spec_expr] の型は `bool` でなければなりません。
束縛変数は、本体の中で spec モードの変数として使えます。
戻り値の型は、束縛子の数によって決まります。

 * 束縛子が 1 つ（`x: T`）の場合、式の型は `T` です。
 * 束縛子が複数（`x: T, y: U, ...`）の場合、式の型はタプル型 `(T, U, ...)` です。

### 意味論 {#semantics}

`choose|x: T| P(x)` は、型 `T` の何らかの値に評価されます。

- `P(x)` が `true` になるような型 `T` の値 `x` が存在する場合、`choose` はそうした値の 1 つを返します。この選択は決定的です。同じ式は、つねに同じ値を返します。
- そうした値が存在しない場合、`choose` は型 `T` の任意の値を返します。その値について `P` が成り立つ保証はありません。

`P` が成り立つという保証付きで結果を使うには、`exists|x: T| P(x)` を別途示さなければなりません。
そうすれば、Verus は `P(choose|x: T| P(x))` を結論できるようにしてくれます。

`choose` と `exists` は相補的です。`exists|x: T| P(x)` は条件を満たす値が*存在する*と主張し、`choose|x: T| P(x)` はその 1 つを*取り出します*。
`choose` のトリガーは、`exists` のトリガーと同じように働きます。詳細は[トリガー注釈](./trigger-annotations.html)を参照してください。

**例:**

```rust
// Requires a proof that such an i exists; choose picks one.
proof fn get_zero_index(s: Seq<int>) -> (i: int)
    requires exists|i: int| 0 <= i < s.len() && s[i] == 0,
    ensures 0 <= i < s.len() && s[i] == 0,
{
    choose|i: int| 0 <= i < s.len() && s[i] == 0
}
```

```rust
// Multiple variables — result is a tuple.
let (i, j): (int, int) = choose|i: int, j: int| less_than(i, j);
```
{% endraw %}

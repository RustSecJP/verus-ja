{% raw %}
# 仕様のための式と演算子 {#expressions-and-operators-for-specifications}

Verus は Rust の構文を拡張し、仕様を書くのに役立つ演算子と式を加えています。
たとえば次のとおりです。

```rust
forall|i: int, j: int| 0 <= i <= j < len ==> f(i, j)
```

この断片には、次のものが現れています。

 * 量化子（quantifier）の `forall`。これについては[後で扱います](./forall.html)
 * 連鎖した演算子
 * 含意の演算子

ここでは、後の 2 つについて説明します。あわせて、論理積、論理和、フィールドへのアクセスのための Verus の記法も説明します。

## 連鎖した不等式 {#chained-inequalities}

仕様では、複数の `<=`、`<`、`>=`、`>` の演算をつなげて書けます。
たとえば、
`0 <= i <= j < len` は `0 <= i && i <= j && j < len` と同じ意味です。

## 論理的な含意 {#logical-implication}

仕様を読みやすくするために、Verus は*含意*（implication）の演算子 `==>` に対応しています。
式 `a ==> b`（「`a` ならば `b`」と読みます）は、論理的に `!a || b` と同値です。
例として、次の式を考えます。

```
forall|i: int, j: int| 0 <= i <= j < len ==> f(i, j)
```

この式は、`0 <= i <= j < len` を満たすすべての `i` と `j` の組について、`f(i, j)` が真であることを意味します。

`==>` の優先順位は、ほかのほとんどの真偽値の演算よりも低いことに注意してください。
たとえば、`a ==> b && c` は `a ==> (b && c)` を意味します。
Verus は、真偽値についての双方向の含意（`<==>`）にも対応しています。こちらの優先順位はさらに低いので、`a <==> b && c` は `a == (b && c)` と同値です。
Verus における[優先順位の完全な説明は、リファレンスを参照してください](./spec-operator-precedence.html)。

## 論理積と論理和 {#conjunction-and-disjunction}

`&&`、`||`、`==>` は Verus の仕様でとてもよく使われるので、`&&` と `||` の優先順位の低い版が欲しくなることがよくあります。
そこで Verus は、「三重の and」（`&&&`）と「三重の or」（`|||`）にも対応しています。
これらは、優先順位を除いて `&&` および `||` と同じです。
含意 `==>` と同値 `<==>` は、`&&&` と `|||` のどちらよりも強く結合します。
`&&&` と `|||` は、「箇条書き」の形で書くのにも便利です。

```
&&& a ==> b
&&& c
&&& d <==> e && f
```

これは `(a ==> b) && c && (d <==> (e && f))` と同じ意味です。

## `struct` や `enum` のフィールドへのアクセス {#accessing-fields-of-a-struct-or-enum}

Verus には、[`struct`](datatypes_struct.html) のフィールドにアクセスしたり、[`enum`](datatypes_enum.html) の列挙子（variant）に照合したりするための構文として、`->`、`is`、`matches` があります。

### `matches` {#matches}

`matches` キーワードを使うと、Rust の `match` 構文とよく似た形で、パターンマッチによってデータ型を分解できます。

```rust
fn option_min(x: Option<u8>, y: Option<u8>) -> (r : Option<u8>)
ensures
    x matches Some(x) ==> r matches Some(r) && r <= x,
    y matches Some(y) ==> r matches Some(r) && r <= y,
    r == x || r == y
{
    match (x, y) {
        (Some(x), Some(y)) => Some(if x <= y { x } else { y }),
        (x@Some(_), _) => x,
        (_, y@Some(_)) => y,
        (None, None) => None,
    }
}
```

詳しくは [`matches` 演算子](./reference-matches.html)を参照してください。
{% endraw %}

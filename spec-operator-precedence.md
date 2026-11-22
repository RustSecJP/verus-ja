{% raw %}
# 演算子の優先順位 {#operator-precedence}

次の表は、演算子の優先順位を、結合がもっとも強いもの（上）からもっとも弱いもの（下）の順に定義しています。

| 演算子                                                                               | 結合性               | 文法                                                    |
|--------------------------------------------------------------------------------------|----------------------|---------------------------------------------------------|
| **結合が強い**                                                                       |                      |                                                         |
| [`.` `->`](./datatypes_struct.html)                                                  | 左                   |                                                         |
| `has`, [`is`, `matches`](./datatypes_enum.html)                                      | 左                   | V@[has_expr] V@[is_expr] V@[matches_expr]               |
| [`*` `/` `%`](./spec-arithmetic.html)                                                | 左                   | V@[arith_expr]                                          |
| [`+` `-`](./spec-arithmetic.html)                                                    | 左                   | V@[arith_expr]                                          |
| [`<<` `>>`](./spec-bit-ops.html)                                                     | 左                   | V@[bit_expr]                                            |
| [`&`](./spec-bit-ops.html)                                                           | 左                   | V@[bit_expr]                                            |
| [`^`](./spec-bit-ops.html)                                                           | 左                   | V@[bit_expr]                                            |
| [<code>&#124;</code>](./spec-bit-ops.html)                                           | 左                   | V@[bit_expr]                                            |
| [`!==` `==` `!=`](./spec-equality.html) `<=` `<` `>=` `>`                            | 括弧が必要           | V@[equality_expr] V@[ineq_expr]                         |
| `&&`                                                                                 | 左                   | V@[and_expr]                                            |
| <code>&#124;&#124;</code>                                                            | 左                   | V@[or_expr]                                             |
| [`==>`](./reference-implication.html)                                                | 右                   | V@[implies_expr]                                        |
| [`<==`](./reference-implication.html)                                                | 左                   | V@[explies_expr]                                        |
| [`<==>`](./reference-implication.html)                                               | 括弧が必要           | V@[iff_expr]                                            |
| `..`                                                                                 | 左                   |                                                         |
| `=`                                                                                  | 右                   |                                                         |
| クロージャー; [`forall`, `exists`](./spec-quantifiers.html); [`choose`](./spec-choose.html) | 右            | V@[forall_expr] V@[exists_expr] V@[choose_expr]         |
| [`&&&`](./prefix-and-or.html)                                                        | 左                   | V@[prefix_and_expr]                                     |
| [<code>&#124;&#124;&#124;</code>](./prefix-and-or.html)                              | 左                   | V@[prefix_or_expr]                                      |
| **結合が弱い**                                                                       |                      |                                                         |

通常の Rust に由来する演算子はすべて、通常の Rust と同じ優先順位を持ちます。
[Rust の演算子の優先順位](https://doc.rust-lang.org/reference/expressions.html)も参照してください。
{% endraw %}

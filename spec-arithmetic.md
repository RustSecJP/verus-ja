{% raw %}
# spec コードにおける算術 {#arithmetic-in-spec-code}

**注意:** このリファレンスページは、*Verus の仕様コード*における算術を扱います。
このページの内容は、*実行可能な Rust コード*における算術には**当てはまりません**。

Verus の算術の入門は、[整数と算術](./integers.html)を参照してください。

### 構文 {#syntax}

```verus-grammar
V@[arith_expr] ::= V@[spec_expr] \+ V@[spec_expr]
             | V@[spec_expr] - V@[spec_expr]
             | - V@[spec_expr]
             | V@[spec_expr] \* V@[spec_expr]
             | V@[spec_expr] / V@[spec_expr]
             | V@[spec_expr] % V@[spec_expr]

V@[ineq_expr] ::= | V@[spec_expr] <= V@[spec_expr]
              | V@[spec_expr] <  V@[spec_expr]
              | V@[spec_expr] >= V@[spec_expr]
              | V@[spec_expr] >  V@[spec_expr]
```

### 型付け {#typing}

spec コードでは、オーバーフローやラップアラウンドを避けるために、算術の結果の型が自動的に広げられます。
各演算子の型を入力の型の関数として表したものを、次の表にまとめます。
ほとんどの場合、入力の型どうしが同じである必要は無いことに注意してください。

| 演算      | 左辺の型            | 右辺の型             | 結果の型    | 備考                 |
|-----------|---------------------|----------------------|-------------|----------------------|
| `<=` `<` `>=` `>`  | t<sub>1</sub>  | t<sub>2</sub>    | bool        |                      |
| `+`         | t<sub>1</sub>     | t<sub>2</sub>        | int         | nat + nat を除く     |
| `+`         | nat               | nat                  | nat         |                      |
| `-`         | t<sub>1</sub>     | t<sub>2</sub>        | int         |                      |
| `*`         | t<sub>1</sub>     | t<sub>2</sub>        | int         | nat * nat を除く     |
| `*`         | nat               | nat                  | nat         |                      |
| `/`         | t                 | t                    | int         | i8...isize, int の場合 |
| `/`         | t                 | t                    | t           | u8...usize, nat の場合 |
| `%`         | t                 | t                    | t           |                      |
| `add(_, _)` | t                 | t                    | t           |                      |
| `sub(_, _)` | t                 | t                    | t           |                      |
| `mul(_, _)` | t                 | t                    | t           |                      |

### 意味論 {#semantics}

型が広げられるので、どの算術演算子の結果も、オーバーフローや切り捨てを考慮しない、算術の正確な結果になります。
例外は、名前付きの演算子 `add`、`sub`、`mul` で、これらは切り捨てを行います。

**商と剰余。**
Verus の仕様では、`/` と `%` は[ユークリッド除法](https://en.wikipedia.org/wiki/Euclidean_division)で定義されます。

`b != 0` のとき、商 `a / b` と剰余 `a % b` は、次の条件を満たすただ 1 組の整数 `q` と `r` として定義されます。

 * `b * q + r == a`
 * `0 <= r < |b|`

次の点に注意してください。

 * 剰余 `a % b` はつねに非負です
 * b が正のとき、商は「床関数による除算（floor division）」になります
 * b が負のとき、商は「天井関数による除算（ceiling division）」になります

また、`b == 0` のとき、`a / b` と `a % b` は未規定（unspecified）であることにも注意してください。
ただし、すべての spec 関数は全域的なので、0 による除算や 0 による剰余は致命的なエラーにはなりません。

> [!DIFF]
> spec コードの `/` と `%` は、左オペランドが負のとき、実行可能コードにおける Rust の `/` 演算子や `%` 演算子と結果が異なることがあります。
>
> `/` の場合:
>
> |               | Verus の spec の `/`（ユークリッド） | Rust の `/` |
> |---------------|----------------------------|----------|
> | `7 / 3`       | 2                          | 2        |
> | `7 / (-3)`    | -2                         | -2       |
> | `(-7) / 3`    | -3                         | -2       |
> | `(-7) / (-3)` | 3                          | 2        |
>
> `%` の場合:
>
> |               | Verus の spec の `%`（ユークリッド） | Rust の `%` |
> |---------------|----------------------------|----------|
> | `7 % 3`       | 1                          | 1        |
> | `7 % (-3)`    | 1                          | 1        |
> | `(-7) % 3`    | 2                          | -1       |
> | `(-7) % (-3)` | 2                          | -1       |

## より高度な算術 {#more-advanced-arithmetic}

Verus の標準ライブラリには、spec 式で使える次の算術関数も含まれています。
* べき乗（[`vstd::arithmetic::power::pow`](https://verus-lang.github.io/verus/verusdoc/vstd/arithmetic/power/fn.pow.html)）
* 2 のべき乗（[`vstd::arithmetic::power2::pow2`](https://verus-lang.github.io/verus/verusdoc/vstd/arithmetic/power2/fn.pow2.html)）
* 整数の対数（[`vstd::arithmetic::logarithm::log`](https://verus-lang.github.io/verus/verusdoc/vstd/arithmetic/logarithm/fn.log.html)）
{% endraw %}

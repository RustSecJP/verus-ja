{% raw %}
# decreases ... when ... via ... {#decreases--when--via-}

`decreases` 節は、再帰関数と相互再帰関数が停止することを保証するために必要です。
入門的な説明は、[チュートリアルのこのページ](./recursion.html)を参照してください。

## 概要 {#overview}

関数の集まりは、その呼び出しグラフが強連結である（つまり、集まりの中のどの関数も、直接または間接に、集まりの中のすべての関数に依存している）とき、
*相互再帰的*（mutually recursive）であると言います。
（自分自身を呼び出す 1 つの関数は、大きさ 1 の相互再帰的な集まりを作ります。）
関数は、何らかの相互再帰的な集まりに含まれているとき、*再帰的*（recursive）であると言います。

再帰的な spec 関数には、`decreases` 節を付けることが求められます。その形は次のとおりです。

```verus-grammar
V@[spec_decreases_clause] ::=
    decreases V@[spec_expr],+ ( when V@[spec_expr] )?  ( via R@[function_ident] )?
```

decreases 節に並べた式の列が、*減少の尺度*（decreases-measure）です。
減少の尺度の式と、`when` 節の式では、関数の引数を参照できます。

Verus は、相互再帰的な 2 つの関数のどの組についても、
減少の尺度の要素の数が同じであることを求めます。

### 減少の尺度 {#the-decreases-measure}

Verus は、再帰関数が自分自身、または同じ相互再帰的な集まりの中のほかの関数を呼び出すとき、
呼び出し元の減少の尺度が、呼び出し先の減少の尺度へ*減少する*（decreases-to）ことを検査します。
[*減少*（decreases-to）の形式的な定義](./reference-decreases-to.html)を参照してください。

### `when` 節 {#the-when-clause}

`when` 節が与えられている場合、減少の性質を証明するときに、その条件を仮定できます。
ただし、関数の定義が具体的に定まるのは、`when` 節が真のときだけになります。
言い換えると、次のようなコードは、

```rust
fn f(...) -> _
    decreases ...
        when condition
{
    body
}
```

次のコードと等価になります。

```rust
fn f(args...) -> _
{
    if condition {
        body
    } else {
        some_unknown_function(args...)
    }
}
```

### Verus が停止性を証明するのを助ける {#helping-verus-prove-termination}

減少の尺度が実際には減少しているのに、Verus がそれを自動では証明できないことがあります。
そのような場合、利用者は、再帰関数の本体の中に証明を書くか、
別の補題を使って減少の性質を証明できます。
ほとんどの場合は、関数の本体の中に証明を書くほうが単純で、読みやすくなります。
別の補題を使う主な理由は、信頼する仕様をできるだけ小さく保ちたい場合です。

**例。**
Verus は、単独では、再帰呼び出しのたびに `n` が減少することを見抜けません。
この例を使って、Verus が停止性を証明するのを助ける 2 つの方法を説明します。
```rust
spec fn floor_log2(n: u64) -> int 
    decreases n
{
    if n <= 1 { 
        0   
    } else {
        floor_log2(n >> 1) + 1 
    }   
}
```
`floor_log2` の再帰が停止することを検査するために、Verus は
`n > 1 ==> decreases_to!(n => n >> 1)` という証明義務を生成します。（`n > 1` という仮定は、
再帰呼び出しが else ブロックの中にあることから来ています。）したがって、次を示す必要があります。

`n > 1 ==> (n >> 1) < n`

Verus は、これを自動では証明できません。

#### 再帰関数の中に停止性の証明を書く {#writing-a-proof-of-termination-inside-your-recursive-function}

停止性の検査のために、spec 関数の本体に `proof {}` ブロックを含められます。
その証明では、減少の尺度が、再帰呼び出しのそれぞれの箇所で本当に減少することを示す必要があります。

なお、現時点では、この proof ブロックは停止性の助けにしか使えません。
たとえば、ほかの場所で使うために、関数についての追加の性質を証明する用途には使えません。

**例**
ここでは、Verus の専用の[ビットベクトル推論モード](bitvec.html)を使って、
Verus に欠けていた事実を補います。

```rust
spec fn floor_log2(n: u64) -> int 
    decreases n
{
    if n <= 1 { 
        0   
    } else {
        proof {
          assert(n > 1 ==> (n >> 1) < n) by(bit_vector);
        }
        floor_log2(n >> 1) + 1 
    }   
}
```

#### `via` 節を使って停止性の証明を別に書く {#writing-a-separate-proof-of-termination-with-the-via-clause}

再帰的な spec 関数が証明のための記述で散らかるのを避けるために、
spec 関数に `via PROOF_FUNCTION_NAME` 節を追加できます。
`PROOF_FUNCTION_NAME` は、同じモジュールで定義された `proof` 関数の名前で、
その関数は再帰関数と同じ引数を取らなければなりません。
また、この proof 関数には `#[via_fn]` 属性を付けなければなりません。

この proof 関数の役目は、それぞれの呼び出し箇所について、該当する減少の性質を証明することです。
言い換えると、spec 関数の本体にあるそれぞれの再帰呼び出しで、
減少の尺度が実際に減少することを示す必要があります。

**例。**
次の定義では、`via` 節を使って、減少の尺度が減少することを証明しています。

```rust
spec fn floor_log2_via(n: u64) -> int 
    decreases n
    via floor_log2_decreases_proof
{
    if n <= 1 { 
        0   
    } else {
        floor_log2_via(n >> 1) + 1 
    }   
}

#[via_fn]
proof fn floor_log2_decreases_proof(n: u64) {
    assert(n > 1 ==> (n >> 1) < n) by(bit_vector);
}
```
proof 関数 `floor_log2_decreases_proof` は `via_fn` として定義されていて、`via` 節から参照されています。この proof 関数の本体には、`n > 1 ==> (n >> 1) < n` の証明（上でインラインに書いたのと同じ証明）が入っています。
{% endraw %}

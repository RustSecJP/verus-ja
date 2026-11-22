{% raw %}
# ループと不変条件 {#loops-and-invariants}

前の節では、`triangle` の末尾再帰による実装を作りました。

```rust
fn tail_triangle(n: u32, idx: u32, sum: &mut u32)
    requires
        idx <= n,
        *old(sum) == triangle(idx as nat),
        triangle(n as nat) <= u32::MAX,
    ensures
        *final(sum) == triangle(n as nat),
    decreases n - idx,
{
    if idx < n {
        let idx = idx + 1;
        assert(*sum + idx <= u32::MAX) by {
            triangle_is_monotonic(idx as nat, n as nat);
        }
        *sum = *sum + idx;
        tail_triangle(n, idx, sum);
    }
}
```

これは、次のように `while` ループとして書き直せます。

```rust
fn loop_triangle(n: u32) -> (sum: u32)
    requires
        triangle(n as nat) <= u32::MAX,
    ensures
        sum == triangle(n as nat),
{
    let mut sum: u32 = 0;
    let mut idx: u32 = 0;
    while idx < n
        invariant
            idx <= n,
            sum == triangle(idx as nat),
            triangle(n as nat) <= u32::MAX,
        decreases n - idx,
    {
        idx = idx + 1;
        assert(sum + idx <= u32::MAX) by {
            triangle_is_monotonic(idx as nat, n as nat);
        }
        sum = sum + idx;
    }
    sum
}
```

このループは、末尾再帰による実装とよく似ています。
（実際、Verus は内部で、このループを、それを囲む `loop_triangle` 関数とは別の
独立した関数であるかのように検証します。）
末尾再帰の関数に事前条件があったのに対し、
ループには *ループ不変条件*（loop invariant）があります。
ループ不変条件は、ループの各反復の前と後で何が真でなければならないかを記述します。
たとえば `n = 10` なら、
ループ不変条件は 11 回、真でなければなりません。
10 回の反復それぞれの前と、
最後の反復の後です。

不変条件 `idx <= n` は、`idx == n` の可能性を許していることに注意してください。
最後の反復の後にはそうなるからです。
もし不変条件を `idx < n` と書こうとすると、
Verus は、最後の反復の後で不変条件が維持されることを証明できません。

ループを抜けた後、
Verus は（ループ不変条件から）`idx <= n` であることを知っており、
ループ条件 `idx < n` が偽だったはずであることも知っています
（そうでなければ、ループは続いていたはずです）。
この 2 つを合わせると、Verus はループを抜けた後に `idx == n` であることを証明できます。
さらに不変条件 `sum == triangle(idx as nat)` もあるので、
Verus は `idx` に `n` を代入して `sum == triangle(n as nat)` と結論でき、
これで `loop_triangle` の事後条件が証明されます。

関数の検証で、適切な事前条件と事後条件を書くためにプログラマの労力が必要なのと同じように、
ループの検証では、ループ不変条件を書くためにプログラマの労力が必要です。
ループ不変条件は、弱すぎても（`invariant true` はたいてい弱すぎます）、
強すぎても（`invariant false` は強すぎます）いけません。
次のことが成り立つ必要があります。
- 最初にループに入る時点で不変条件が成り立つ
  （たとえば `0 <= n` なので、初期値 `idx = 0` に対して `idx <= n` が成り立つ）
- ループ本体の終わりでも不変条件が成り立ち、
  ループの反復をまたいで不変条件が維持される
- 不変条件が、ループを抜けた後に知りたい性質を証明できるだけの強さを持つ
  （たとえば `loop_triangle` の事後条件を証明できる）

上で述べたとおり、
Verus はループを、そのループを含む関数とは別に
（たとえば `loop_triangle` とは別に）検証します。
つまりループは、`triangle(n as nat) < 0x1_0000_0000` のような事前条件を
周囲の関数から自動的には引き継ぎません。
ループがこうした事前条件に依存するなら、
それらをループ不変条件に明示的に並べなければなりません。
（こうなっている理由は、大きな while ループを持つ大きな関数について、
SMT ソルバーによる求解の効率を上げるためです。
Verus が周囲の関数とループを別々の部品に分け、
モジュール的に検証するほうが、検証は速く走ります。）

Verus では、この振る舞いをやめることもできます。その場合、ループは
周囲の文脈から情報を引き継ぎます。これによってループ不変条件は簡単になりますが、
中規模から大規模の関数では検証時間が増えるかもしれません。
1 つの関数または 1 つの while ループについてやめるには、属性
`#[verifier::loop_isolation(false)]` を付けます。モジュールやクレートの
単位でやめることもでき、その場合は属性 `#![verifier::loop_isolation(false)]` を
モジュールまたはクレートのルートに付けます。そのうえで、個々の
関数やループに `#[verifier::loop_isolation(true)]` を付ければ、全体の
設定を局所的に上書きできます。
{% endraw %}

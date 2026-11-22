{% raw %}
# 証明を小さな部分に分ける {#breaking-proofs-into-smaller-pieces}

## 動機 {#motivation}

証明コードをたくさん含む長い関数を書くと、Verus はそれに応じて、長くて難しい問題を SMT ソルバーに解かせることになります。そこで、その関数を小さな部分に分けることで、ソルバーの性能を改善できます。この性能の改善は劇的なものになることがあります。ソルバーの応答時間は、ふつう、証明が大きくなるにつれて非線形に増えるからです。なにしろ、スコープ内の事実が 2 倍になれば、ソルバーが証明を探すためにたどりうる道筋は 2 倍よりはるかに多くなります。その結果、関数を分けるかどうかが、ソルバーがタイムアウトするか、すばやく成功するかの分かれ目になることさえあります。

## 部分的な証明を補題に移す {#moving-a-subproof-to-a-lemma}

長い関数があるなら、その中から、局所的に役立つ事実の集まり `S` の証明として働いている、ほどほどの大きさの部分 `P` を探してください。`P` を、事後条件が `S` である補題の呼び出しに置き換え、`P` をその補題の本体にします。`P` の元の文脈のうち、`S` を成り立たせるのに必要な部分はどれかを考え、それらを補題の `requires` 節として書きます。それらの `requires` 節にローカル変数が出てくる場合は、その変数を引数として補題に渡します。

たとえば、
```
fn my_long_function(x: u64, ...)
{
    let y: int = ...;
    ... // first part of proof, establishing fact f(x, y)
    P1; // modest-size proof...
    P2; //   establishing...
    P3; //   facts s1 and s2...
    P4; //   about x and y
    ... // second part of proof, using facts s1 and s2
}
```
は、次のようになるでしょう。
```
proof fn my_long_function_helper(x: u64, y: int)
    requires
        f(x, y)
    ensures
        s1(x),
        s2(x, y)
{
    P1; // modest-size proof...
    P2; //   establishing...
    P3; //   facts s1 and s2...
    P4; //   about x and y
}

fn my_long_function(x: u64, ...)
{
    let y: int = ...;
    ... // first part of proof, establishing fact f(x, y)
    my_long_function_helper(x, y);
    ... // second part of proof, using facts s1 and s2
}

```

`P` を補題の本体に移してみると、長い関数から `P` を取り除けるだけでなく、移した先の補題からも `P` のかなりの部分を取り除けることに気づくかもしれません。`S` を成り立たせることだけを目的とした補題では、ソルバーが推論すべき文脈が小さくなるからです。そのため、`S` をうまく、すばやく成り立たせるのに必要な証明注釈が少なくて済むことがあります。たとえば、次のようになります。

```
proof fn my_long_function_helper(x: u64, y: int)
    requires
        f(x, y)
    ensures
        s1(x),
        s2(x, y)
{
    P1; // It turns out that P2 and P3 aren't necessary when
    P4; //    the solver is focused on just f, s1, s2, x, and y.
}
```

## 証明をパート 1、2、……、n に分ける {#dividing-a-proof-into-parts-1-2--n}

もう 1 つの方法は、大きな関数の証明を連続した `n` 個の部分に分け、それぞれを専用の補題に入れることです。1 つ目の補題の `requires` 節は関数の `requires` 節にし、その `ensures` 節は、その証明が成り立たせることの要約にします。2 つ目の補題の `requires` 節は 1 つ目の補題の `ensures` 節と一致させ、その `ensures` 節は、それが成り立たせることの要約にします。これを `n` 番目の補題まで続け、その `ensures` 節は元の関数の `ensures` 節にします。最後に、元の関数の証明を、これら `n` 個の補題を順に呼び出す列に置き換えます。


たとえば、
```
proof fn my_long_function(x: u64)
    requires r(x)
    ensures  e(x)
{
    P1;
    P2;
    P3;
}
```
は、次のようになるでしょう。
```
proof fn my_long_function_part1(x: u64) -> (y: int)
    requires
        r(x)
    ensures
        mid1(x, y)
{
    P1;
}

proof fn my_long_function_part2(x: u64, y: int)
    requires
        mid1(x, y)
    ensures
        mid2(x, y)
{
    P2;
}

proof fn my_long_function_part3(x: u64, y: int)
    requires
        mid2(x, y)
    ensures
        e(x)
{
    P3;
}

proof fn my_long_function(x: u64)
    requires r(x)
    ensures  e(x)
{
    let y = my_long_function_part1(x);
	my_long_function_part2(x, y);
	my_long_function_part3(x, y);
}

```
`r(x)`、`mid1(x, y)`、`mid2(x, y)`、`e(x)` という式はそれぞれ 2 回ずつ繰り返されるので、それぞれを spec 関数として括り出し、繰り返しを避けるとよいかもしれません。
{% endraw %}

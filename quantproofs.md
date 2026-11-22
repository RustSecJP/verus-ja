{% raw %}
# forall と exists についての証明 {#proofs-about-forall-and-exists}

これまでの節では、`forall` 式と `exists` 式にとっての
トリガーの重要性を強調してきました。
具体的には、`forall|i| f(i)` を知っている場合、
SMT ソルバーはトリガーを見て `i` を具体化します。
また、`exists|i| f(i)` を証明したい場合、
SMT ソルバーはトリガーを見て、`f(i)` が真になる証拠 `i` を見つけます。
言い換えると、`forall` 式を *使う* ことはトリガーに頼り、
`exists` 式を *証明する* こともトリガーに頼ります。
これらの場合を、次の表に書けます。

|        | 証明する                                          | 使う                                           |
|--------|---------------------------------------------------|------------------------------------------------|
| forall | たいていはそのままうまくいく。だめなら assert-by | トリガー                                       |
| exists | トリガー                                          | たいていはそのままうまくいく。だめなら choose |

残りの 2 つの場合、
つまり `forall` 式を証明する場合と `exists` 式を使う場合はどうでしょうか。
これらの場合は、実は自動化がより簡単で、トリガーに頼りません。
実際、次の例のように、
多くの場合は自動的にそのままうまくいきます。

```rust
spec fn is_distinct(x: int, y: int) -> bool {
    x != y
}

spec fn dummy(i: int) -> bool;

proof fn prove_forall()
    ensures
        forall|i: int, j: int|
            #![trigger dummy(i), dummy(j)]
            is_distinct(i, j) ==> is_distinct(j, i),
{
    // proving the forall just works; the trigger is irrelevant
}

proof fn use_exists(x: int)
    requires
        exists|i: int| #![trigger dummy(i)] x == i + 1 && is_distinct(i, 5),
{
    // using the exists just works; the trigger is irrelevant
    assert(x != 6);
}
```

これらの例では、トリガーは何の役割も果たしていません。
このことを強調するために、例のほかのどこにも現れない
`dummy` 関数をトリガーに使いましたが、
それでも SMT ソルバーは関数を難なく検証します。
（ただし、上の関数のどれかを呼び出した場合、
呼び出す側は `exists` 式を証明するか、
`forall` 式を使わなければならず、
呼び出す側がトリガーを扱うことになる点に注意してください。）

SMT ソルバーが上のコードを検証するのに
なぜトリガーに頼らないのか、直感をつかみたいなら、
この検証を、次のコードの検証に似たものと考えるとよいでしょう。
このコードでは、量化子が取り除かれ、量化された変数が
関数の引数へと引き上げられています。

```rust
proof fn hoisted_forall(i: int, j: int)
    ensures
        is_distinct(i, j) ==> is_distinct(j, i),
{
}

proof fn hoisted_exists(x: int, i: int)
    requires
        x == i + 1 && is_distinct(i, 5),
{
    assert(x != 6);
}
```

## assert-by で forall を証明する {#proving-forall-with-assert-by}

上の単純な例のように証明が「そのままうまくいく」とは限りません。
たとえば、証明が、帰納法で証明される補題に頼っていることがあります。
SMT ソルバーは、そうした補題を完全に自動では証明できません。
任意の偶数 `i` について `f(i)` を証明する補題があるとします。

```rust
spec fn f(i: int) -> bool { ... }

proof fn lemma_even_f(i: int)
    requires
        is_even(i),
    ensures
        f(i),
{ ... }
```

ここで、すべての偶数 `i` について `f(i)` が真であることを証明したいとします。

```rust
proof fn test_even_f()
    ensures
        forall|i: int| is_even(i) ==> f(i), // FAILS because we don't call the lemma
{
}
```

上の証明は、`lemma_even_f` を呼び出していないので失敗します。
ところが、`lemma_even_f` を呼び出そうとすると、すぐに問題にぶつかります。
補題に引数として `i` を渡す必要がありますが、
`i` はスコープに入っていません。

```rust
proof fn test_even_f()
    ensures
        forall|i: int| is_even(i) ==> f(i),
{
    lemma_even_f(i); // ERROR: i is not in scope here
}
```

これに対処するため、Verus は、`forall` 式を証明するための
特別な形の `assert ... by` に対応しています。

```rust
proof fn test_even_f()
    ensures
        forall|i: int| is_even(i) ==> f(i),
{
    assert forall|i: int| is_even(i) implies f(i) by {
        // First, i is in scope here
        // Second, we assume is_even(i) here
        lemma_even_f(i);
        // Finally, we have to prove f(i) here
    }
}
```

`assert ... by` の本体の中では、
`forall` の変数がスコープに入っており、
`==>` の左辺が仮定されています。
これによって、本体は `lemma_even_f(i)` を呼び出せます。

## choose で exists を使う {#using-exists-with-choose}

上の例では、補題を呼び出すために、
`forall` の量化子の変数をスコープに入れる必要がありました。
同じような状況は、`exists` の量化子の変数でも起こりえます。
`f(i)` を証明する次の補題があるとします。

```rust
spec fn g(i: int, j: int) -> bool { ... }

proof fn lemma_g_proves_f(i: int, j: int)
    requires
        g(i, j),
    ensures
        f(i),
{ ... }
```

`g(i, j)` が真になる `j` が存在することを知っているなら、
`lemma_g_proves_f` を呼び出せるはずです。
ところが、`j` がスコープに入っていないという問題にぶつかります。

```rust
proof fn test_g_proves_f(i: int)
    requires
        exists|j: int| g(i, j),
    ensures
        f(i),
{
    lemma_g_proves_f(i, j); // ERROR: j is not in scope here
}
```

このような場合は、
`choose`（[前の節](./exists.html)で説明しました）を使って、
`exists` 式から値 `j` を取り出せます。

```rust
proof fn test_g_proves_f(i: int)
    requires
        exists|j: int| g(i, j),
    ensures
        f(i),
{
    lemma_g_proves_f(i, choose|j: int| g(i, j));
}
```
{% endraw %}

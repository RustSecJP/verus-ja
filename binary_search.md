{% raw %}
# 例: 二分探索 {#example-binary-search}

`forall` と `exists` がどのように働くのかを、もう少し大きな例で見てみましょう。
次のコードは、整列済みの列から値 `k` を探し、`k` がある位置の添字 `r` を返します。

```rust
fn binary_search(v: &Vec<u64>, k: u64) -> (r: usize)
    requires
        forall|i: int, j: int| 0 <= i <= j < v.len() ==> v[i] <= v[j],
        exists|i: int| 0 <= i < v.len() && k == v[i],
    ensures
        r < v.len(),
        k == v[r as int],
{
    let mut i1: usize = 0;
    let mut i2: usize = v.len() - 1;
    while i1 != i2
        invariant
            i2 < v.len(),
            exists|i: int| i1 <= i <= i2 && k == v[i],
            forall|i: int, j: int| 0 <= i <= j < v.len() ==> v[i] <= v[j],
        decreases i2 - i1,
    {
        let ix = i1 + (i2 - i1) / 2;
        if v[ix] < k {
            i1 = ix + 1;
        } else {
            i2 = ix;
        }
    }
    i1
}

fn main() {
    let mut v: Vec<u64> = Vec::new();
    v.push(0);
    v.push(10);
    v.push(20);
    v.push(30);
    v.push(40);
    assert(v[3] == 30);  // needed to trigger exists|i: int| ... k == v[i]
    let r = binary_search(&v, 30);
    assert(r == 3);
}
```

事前条件（precondition）`exists|i: int| 0 <= i < v.len() && k == v[i]` は、`k` が列のどこかにあることを指定しています。
これにより、探索は必ず `k` を見つけられます。
この `exists` 式に対して自動的に推論されるトリガーは `v[i]` です。
`main` 関数は、証拠（witness）`i = 3` によってこの条件を満たします。つまり `30 == v[3]` です。

```
assert(v[3] == 30); // needed to trigger exists|i: int| ... k == v[i]
let r = binary_search(&v, 30);
```

探索は、2 つの添字 `i1` と `i2` を保ちながら進みます。この 2 つは両側から `k` に向かって範囲を狭めていき、`k` を含む添字は探索のあいだずっと `i1` と `i2` の間にあります。

```
exists|i: int| i1 <= i <= i2 && k == v[i]
```

ループを抜けるには、ループ条件 `i1 != i2` が偽でなければなりません。
つまり、`i1` と `i2` は等しくなければなりません。
このとき、上の `exists` 式の `i` は `i1` とも `i2` とも等しいはずなので、`k == v[i1]` だと分かります。
したがって、結果として `i1` を返せます。

## ループ不変条件が保たれることを証明する {#proving-that-the-loop-invariant-is-maintained}

ループの各反復では、反復の前にループ不変条件が成り立つと仮定でき、反復の後にもループ不変条件が成り立つことを証明しなければなりません。
不変条件 `exists|i: int| i1 <= i <= i2 && k == v[i]` の証明を、SMT ソルバーが量化子 `forall` と `exists` をどう扱うかに注目して、もう少し詳しく見てみましょう。

鍵となる手順は次のとおりです。

- `exists|i: int| ... k == v[i]` が分かっていると、`k == v[i_witness]` となる証拠 `i_witness` が得られます。
- 現在の反復の `exists|i: int| ...` から得た証拠 `i_witness` が、次の反復の `exists|i: int| ...` の証拠になります。
- `if` の条件 `v[ix] < k` によって、証明は `v[ix] < k` と `v[ix] >= k` の 2 つの場合に分かれます。
- 式 `v[i_witness]` と `v[ix]` は、式 `forall|i: int, j: int| ... v[i] <= v[j]` のトリガー `v[i], v[j]` にマッチします。

これから、これらの手順をもっと詳しくたどっていきます。
（退屈すぎると感じたら、遠慮なく[先へ飛ばして](./binary_search.html#helping-the-automation-succeed)ください。
次の小節で説明するように、要点は、うまくお膳立てをしておけば、退屈な細部は SMT ソルバーが自動的に片付けてくれるということです。）

考えるべき場合は 2 つあります。`if` の条件 `v[ix] < k` が真の場合と、偽の場合です。
ここでは前者、つまり `v[ix] < k` の場合だけを見ます。

ループの反復の始めに、ループ不変条件を仮定します。

```
exists|i: int| i1 <= i <= i2 && k == v[i]
```

これにより、次を満たす証拠 `i_witness` が存在することが分かります。

```
i1 <= i_witness <= i2 && k == v[i_witness]
```

`v[ix] < k` の場合は、`i1 = ix + 1` を実行します。

```rust
let ix = i1 + (i2 - i1) / 2;
if v[ix] < k {
    i1 = ix + 1;
} else {
```

`i1` の新しい値は `ix + 1` なので、`i1` を `ix + 1` に置き換えたループ不変条件を証明する必要があります。

```
exists|i: int| ix + 1 <= i <= i2 && k == v[i]
```

`exists` 式を証明するには、SMT ソルバーがその式のトリガーにマッチするものを見つける必要があります。
この式に対して自動的に選ばれるトリガーは `v[i]` なので、SMT ソルバーは `v[...]` の形の式を探します。
すると、前のループ不変条件（上に示したもの）から `v[i_witness]` が見つかります。
また、`if` の条件 `v[ix] < k` から `v[ix]` も見つかります。
これらをもとに、SMT ソルバーは `i = i_witness` または `i = ix` として、`ix + 1 <= i <= i2 && k == v[i]` の証明を試みます。

```
ix + 1 <= i_witness <= i2 && k == v[i_witness]
ix + 1 <= ix <= i2 && k == v[ix]
```

`i = ix` の場合は行き止まりです。`ix + 1 <= ix` が真になることは決してないからです。
`i = i_witness` の場合のほうが見込みがあります。
ループの反復の始めに `i_witness` について仮定したことから、`i_witness <= i2` と `k == v[i_witness]` はすでに分かっています。
あとは `ix + 1 <= i_witness` を証明するだけです。
これは `ix < i_witness` に簡単化できます。

#### ix < i_witness を証明する {#proving-ix--i_witness}

`ix < i_witness` を証明するために、今度は `forall` のループ不変条件に目を向けます。

```
forall|i: int, j: int| 0 <= i <= j < v.len() ==> v[i] <= v[j],
```

これを具体化（instantiate）するために、SMT ソルバーはここでもトリガーに頼ります。
この `forall` 式ではトリガーは `v[i], v[j]` なので、SMT ソルバーは再び `v[...]` の形の項を探し、`v[i_witness]` と `v[ix]` を見つけます。
`i_witness` と `ix` を `i` と `j` に割り当てる方法は、4 通りあります。

```
0 <= i_witness <= i_witness < v.len() ==> v[i_witness] <= v[i_witness]
0 <= i_witness <= ix < v.len() ==> v[i_witness] <= v[ix]
0 <= ix <= i_witness < v.len() ==> v[ix] <= v[i_witness]
0 <= ix <= ix < v.len() ==> v[ix] <= v[ix]
```

このうち、最も役に立つのは 2 つ目です。

```
0 <= i_witness <= ix < v.len() ==> v[i_witness] <= v[ix]
```

`k == v[i_witness]` はすでに分かっているので、これは次のようになります。

```
0 <= i_witness <= ix < v.len() ==> k <= v[ix]
```

`==>` の右辺は `k <= v[ix]` と言っていますが、これは `v[ix] < k` という仮定と矛盾します。
つまり、`==>` の左辺は偽でなければなりません。

```
!(0 <= i_witness <= ix < v.len())
```

SMT ソルバーは `0 <= i_witness` と `ix < v.len()` を知っているので、これを次のように絞り込みます。

```
!(i_witness <= ix)
```

これは `ix < i_witness` を意味しており、まさに求めていたものです。

## 自動化がうまくいくように助ける {#helping-the-automation-succeed}

前の節で見たように、ループ不変条件の証明には長い推論の連鎖が必要です。
幸い、SMT ソルバーはこれらの手順をすべて自動的に行います。
実のところ、これはとりわけ運のよい例です。
Verus がトリガーも自動的に選び、しかもそのトリガーが、SMT ソルバーが証明を完成させるのにちょうど必要なものになっているからです。

しかし一般には、事前条件、事後条件（postcondition）、ループ不変条件をどう表現するかが、Verus と SMT ソルバーが自動的に成功するかどうかに大きく影響します。
たとえば、（事前条件とループ不変条件の中の）整列済みであるという条件を、次のように書いていたとしましょう。

```
forall|i: int| 0 <= i < v.len() - 1 ==> #[trigger] v[i] <= v[i + 1]
```

次のように書く代わりに、です。

```
forall|i: int, j: int| 0 <= i <= j < v.len() ==> v[i] <= v[j]
```

[前の節](./multitriggers.html)で説明したように、トリガー `v[i]` と `v[i] <= v[i + 1]` の組み合わせはマッチングループ（matching loop）を引き起こし、SMT ソルバーを無限ループに陥らせることがあります。
実際、まさにそのとおりのことが起こります。

```
error: while loop: Resource limit (rlimit) exceeded; consider rerunning with --profile for more details
    |
    | /     while i1 != i2
    | |         invariant
    | |             i2 < v.len(),
    | |             exists|i: int| i1 <= i <= i2 && k == v[i],
      |
    | |         }
    | |     }
    | |_____^
```

とはいえ、仮に SMT ソルバーが無限ループを避けられたとしても、どうすれば自動的に成功できたのかは想像しにくいところです。
上で説明したように、決定的な手順は、`i = i_witness` と `j = ix` で具体化して `v[i_witness] <= v[ix]` について何かを知ることでした。
`forall` 式に変数が `i` の 1 つしかないときは、このような具体化はそもそもできません。
`v[i_witness] <= v[ix]` について何かを知るには、`v[i] <= v[i + 1]` という手順をいくらでも長くつなぎ合わせて、`i_witness` から `i_witness + 1`、`i_witness + 2` と進み、`ix` までたどり着く必要があります。
これには、別途、帰納法による証明が必要になります。
直観的に言えば、二分探索のように、ある添字から別の添字へ大きく飛ぶアルゴリズムには、`v[i] <= v[i + 1]` よりも `v[i] <= v[j]` という式のほうが適しています。
`i` と `j` はいくらでも離れていてよいのに対し、`i` と `i + 1` は 1 要素しか離れていないからです。

SMT による自動化が失敗すると、証明が通るまで `assert` や補題、帰納法による証明などをすぐに足し始めたくなるものです。
手作業を十分に重ねれば、問題のある `v[i] <= v[i + 1]` という整列の定義のままでも、おそらく二分探索の証明を完成させられるでしょう。
しかし、それは間違ったやり方です。
それほどの手作業をしなくても自動化がうまくいくように、定義を組み立てるほうがよいのです。
手作業の長い証明を書いていることに気づいたら、一歩引いて、なぜ自動化が失敗しているのかを突き止める価値があります。
定義を変えれば、自動化の失敗を直せるかもしれません。

結局のところ、車が故障したら、押して歩くよりも修理するほうがたいていはよいのです。
{% endraw %}

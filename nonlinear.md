{% raw %}
# 整数: 非線形算術 {#integers-nonlinear-arithmetic}

一般的に言って、Verus の既定のソルバー（Z3）は、_線形_ の整数算術の扱いにとても優れています。
線形算術が扱うのは、等式、不等式、加算、減算、そして _定数_ による乗算と除算です。
つまり、`4 * x + 3 * y - z <= 20` のような式の扱いは得意です。
しかし、`x * y`（`x` も `y` も定数に置き換えられないとき）や `x / y`（`y` を定数に置き換えられないとき）のような _非線形_ の式が関わると、それほどうまく扱えません。

そのため、既定のモードでは、よく使われる公理の多くが利用できません。次のものが含まれますが、これに限りません。

 * `x * y == y * x`
 * `x * (y * z) == (x * y) * z`
 * `x * (a + b) == x * a + x * b`
 * `0 <= x <= y && 0 <= z <= w ==> x * z <= y * w`

この制約がある理由は、Verus が既定の証明器モードで、非線形算術の理論を _意図的に_ 無効にしているからです。

ただし、専用の証明器モードを呼び出すことで、非線形の推論を**明示的に有効にする**（オプトインする）ことができます。
非線形算術に関係する証明器モードは 2 つあります。

 * `nonlinear_arith` - Z3 の非線形算術の理論を有効にします。
 * `integer_ring` - 決定可能な、環の等式理論を有効にします。

1 つ目は汎用ですが、残念ながらいくらか予測しにくいところがあります。（既定で無効になっているのはこのためです。）
2 つ目が扱う問題の種類はもっと限られますが、決定可能で効率的です。
どちらの証明器モードを呼び出すにも、_証明器のコンテキストを最小限にする_ 方法を理解している必要があります。
以下で、それぞれのモードをもっと詳しく説明します。

なお、どちらのモードでも証明がうまくいかない場合は、Verus の[算術ライブラリ](https://verus-lang.github.io/verus/verusdoc/vstd/arithmetic/index.html)の補題を手で呼び出すこともできます。
このライブラリは、非線形の演算の振る舞いについての検証済みの事実を、数多く提供しています。
たとえば、上に挙げた利用できない性質は、それぞれ次のものを呼び出すことで証明できます。

* [`lemma_mul_is_commutative`](https://verus-lang.github.io/verus/verusdoc/vstd/arithmetic/mul/fn.lemma_mul_is_commutative.html)
* [`lemma_mul_is_associative`](https://verus-lang.github.io/verus/verusdoc/vstd/arithmetic/mul/fn.lemma_mul_is_associative.html)
* [`lemma_mul_is_distributive_add`](https://verus-lang.github.io/verus/verusdoc/vstd/arithmetic/mul/fn.lemma_mul_is_distributive_add.html)
* [`lemma_mul_upper_bound`](https://verus-lang.github.io/verus/verusdoc/vstd/arithmetic/mul/fn.lemma_mul_upper_bound.html)

証明でこうした補題をいくつも使う場合は、証明を読みやすく、保守しやすくするために、[構造化された証明](calc.html)を使うとよいでしょう。

## 1. 専用のソルバーを呼び出す: `nonlinear_arith`
{: id="1-invoking-a-specialized-solver-nonlinear_arith"}

専用のソルバーは `by` キーワードで呼び出します。`by` は `assert` 文にも `proof fn` にも付けられます。

ここでは、`nonlinear_arith` ソルバーを使って、その働きを見ていきます。
このソルバーは、[Z3 の整数に対する非線形算術の理論](https://microsoft.github.io/z3guide/docs/theories/Arithmetic/#non-linear-arithmetic)を有効にします。

### `assert(...) by(nonlinear_arith)` によるインラインの証明 {#inline-proofs-with-assert-bynonlinear_arith}
大きな関数の途中で非線形の性質を証明するには、`assert(...) by(nonlinear_arith)` と書けます。
これは、表明した性質を証明するためだけの、別の Z3 への問い合わせを作ります。この問い合わせでは、Z3 は非線形のヒューリスティクスを有効にして動きます。
この問い合わせには、周囲の事実（たとえば、外側の関数の `requires` 節や、それより前の変数への代入から得られる知識）は含まれ**ません**。ただし、次のものは例外です。

 * 変数の型から推論されるもの（たとえば、`u64` や `nat` が取り得る範囲）
 * 明示的に与えたもの

コンテキストを明示的に与えるには、次に示すように `requires` 節を使えます。

```rust
proof fn bound_check(x: u32, y: u32, z: u32)
    requires
        x <= 8,
        y <= 8,
{
    assert(x * y <= 100) by (nonlinear_arith)
        requires
            x <= 10,
            y <= 10;

    assert(x * y <= 1000);
}
```

この例を 1 段階ずつ見ていきましょう。

 * Verus は _通常のソルバー_ を使って、assert の `requires` 節、つまり `x <= 10 && y <= 10` を証明します。これは関数の事前条件（precondition）から導かれます。
 * Verus は Z3 の _非線形ソルバー_ を使って、`x <= 10 && y <= 10 ==> x * y <= 100` を証明します。これは通常のソルバーではできませんが、非線形ソルバーならできます。
 * これで、事実 `x * y <= 100` が、以降の assert のための証明コンテキストに加わります。
 * Verus は _通常のソルバー_ を使って、`x * y <= 1000` を証明します。これは `x * y <= 100` から導かれます。

さらに、`assert ... by(nonlinear_arith) by { ... }` のように `by` 節を使うと、`by` 節の中のすべてが非線形ソルバーを使うようになります。

### `proof fn ... by(nonlinear_arith)` による再利用できる証明 {#reusable-proofs-with-proof-fn--bynonlinear_arith}

`by(nonlinear_arith)` は、証明関数のシグネチャでも使えます。
`by(nonlinear_arith)` を付けると、その関数についての問い合わせは、非線形算術の推論を有効にして実行されます。
たとえば次のとおりです。

```rust
proof fn bound_check2(x: u32, y: u32, z: u32) by (nonlinear_arith)
    requires
        x <= 8,
        y <= 8,
    ensures
        x * y <= 64
{ }
```

このように `proof fn` に対して専用のソルバーを呼び出すと、そのソルバーは補題の証明に使われます。
その補題が別の場所から呼び出されるときには、Verus は（いつもどおり）事前条件が満たされていることを証明します。これには通常のソルバーを使います。

## 2. 環に基づく性質を証明する: `integer_ring`
{: id="2-proving-ring-based-properties-integer_ring"}

一般の非線形の論理式を安定して解くことはできませんが、非線形の論理式のうち特定の部分クラスであれば解けます。
たとえば、一連の合同関係（つまり、ある除数 `n` を法とする等式）からなる非線形の論理式です。
簡単な例として、`a % n == b % n ==> (a * c) % n == (b * c) % n` を示したいとしましょう。

Verus には、このような証明義務を果たすための決定的な証明戦略があります。
この戦略は `integer_ring` と呼ばれます。

[_注_: 現在のところ、`integer_ring` を呼び出せるのは `proof fn ... by(integer_ring)` の形だけです。インラインの assert には対応していません。]


Verus はそのとき、[Singular](https://www.singular.uni-kl.de/) という専用の代数ソルバーを使って証明義務を果たします。
注釈の名前が示唆するように、この証明技法が完全である（つまり、成功が保証されている）のは、すべての[環](https://en.wikipedia.org/wiki/Ring_(mathematics))で真である性質に対してだけです。
整数に特有の性質に依存する論理式は、うまく解けないことがあります。

この証明技法を使うには、Verus のインストールに少し追加の設定が必要です。
[Singular のインストールと設定](./install-singular.html)を参照してください。

### 詳細と制約 {#detailslimitations}
- **int** の引数でしか使えません。
- 不等式を含む論理式には対応していません。
- 除算には対応していません。
- 論理式の中の関数呼び出しは、未解釈関数（uninterpreted function）として扱われます。関数の定義が証明にとって重要な場合は、証明関数の `requires` 節の中で関数の定義を展開してください。
- `integer_ring` の補題を使うとき、剰余演算子（`%`）の除数は 0 であってはなりません。`integer_ring` の補題の ensures 節の中で除数が 0 になり得る場合、ensures 節の事実は呼び出し箇所で利用できません。

`integer_ring` に何ができて何ができないのかを理解するには、それが剰余演算子 `%` をどう扱うのかを理解することが大切です。
`integer_ring` は不等式を理解しないので、`0 <= (a % b) < b` であることを必要とする推論はできません。
そのため、`%` を主に、プログラミングでおなじみの演算子として考えていると、Singular の結果に戸惑うかもしれません。

たとえば、`a % b == x` を事前条件として使うとしましょう。
これを Singular 向けに符号化すると、`a % b == x % b` になります。もっと伝統的な「数学の」言い方をすれば、`a ≡ x (mod b)` です。
これは、`x` が範囲 `[0, b)` にあることを意味 _しません_。`a` と `x` が b を法として同じ同値類にあることを意味するだけです。
言い換えると、`a % b == x` ならば `a ≡ x (mod b)` ですが、その逆は成り立ちません。

同じ理由で、`x` が 0 でないかぎり、`a % b == x` の形の事後条件（postcondition）の証明を `integer_ring` ソルバーに頼むことはできません。
`integer_ring` ソルバーは `a ≡ x (mod b)`、同じことですが `(a - x) % b == 0` を証明できます。しかしこれは、`a % b == x` を意味 _しません_。

この制約を理解するために、具体的な例を見てみましょう。

```rust
proof fn foo(a: int, b: int, c: int, d: int, x: int, y: int) by(integer_ring)
    requires
        a == c,
        b == d,
        a % b == x,
        c % d == y
    ensures
        x == y,
{
}
```

この定理の主張は自明に見えますし、実際、Verus は既定の証明戦略を使えば簡単に解きます。
しかし、`integer_ring` はこれを解けません。
Singular への問い合わせを調べると、その理由が分かります。
（これを記録する方法は[こちら](#examining-the-encoding)を参照してください。）

```
ring ring_R=integer, (a, b, c, d, x, y, tmp_0, tmp_1, tmp_2), dp;
    ideal ideal_I =
      a - c,
      b - d,
      (a - (b * tmp_0)) - x,
      (c - (d * tmp_1)) - y;
    ideal ideal_G = groebner(ideal_I);
    reduce(x - y, ideal_G);
    quit;
```

ここから、`a % b` は `a - b * tmp_0` に変換され、`c % d` は `c - d * tmp_1` に変換されていることが分かります。
繰り返しになりますが、`a - b * tmp_0` や `c - d * tmp_1` の範囲が限られているという制約がないので、この簡単化が行われた後では、`a - b * tmp_0 == c - d * tmp_1` と結論することはできません。

## 3. `integer_ring` と `nonlinear_arith` を組み合わせる
{: id="3-combining-integer_ring-and-nonlinear_arith"}

上で説明したように、`integer_ring` の機能にはいくつかの制約があり、`integer_ring` の機能だけで任意の非線形の性質を得ることはできません。
その代わりに、目的の性質のための主な補題として `by(nonlinear_arith)` の関数を用意し、`integer_ring` の補題を補助の補題として使うのが、よくあるパターンです。

不等式と除算に対応していないことを回避するには、多くの場合、`integer_ring` で果たされる補助の証明を書き、それを使って、`integer_ring` が直接には対応していない性質を証明できます。
さらに、論理式に追加の変数を加えることもできます。
たとえば除算を回避するには、`b/a` の代わりに、`b = a * c` となる `c` を導入できます。

#### 例 1: 剰余算術についての事実を与える補助の補題としての `integer_ring` {#example-1-integer_ring-as-a-helper-lemma-to-provide-facts-on-modular-arithmetic}
次の `lemma_mod_difference_equal` 関数では、requires 節の中に 4 つの不等式があり、これらは `integer_ring` に符号化できません。
ensures 節では、`y % d - x % d == y - x` を証明したいと考えています。
補助の補題 `lemma_mod_difference_equal_helper` は、`y % d - x % d` が `d` を法として `(y - x)` に等しいことを与えるだけです。
証明の残りは `by(nonlinear_arith)` によって行われます。

```rust
pub proof fn lemma_mod_difference_equal_helper(x: int, y:int, d:int, small_x:int, small_y:int, tmp1:int, tmp2:int) by(integer_ring)
    requires
        small_x == x % d,
        small_y == y % d,
        tmp1 == (small_y - small_x) % d,
        tmp2 == (y - x) % d,
    ensures
        (tmp1 - tmp2) % d == 0
{}
pub proof fn lemma_mod_difference_equal(x: int, y: int, d: int) by(nonlinear_arith)
    requires
        d > 0,
        x <= y,
        x % d <= y % d,
        y - x < d
    ensures
        y % d - x % d == y - x
{
    let small_x = x % d;
    let small_y = y % d;
    let tmp1 = (small_y - small_x) % d;
    let tmp2 = (y - x) % d;
    lemma_mod_difference_equal_helper(x,y,d, small_x, small_y, tmp1, tmp2);
}
```

次の `lemma_mod_between` 関数では、`x % d <= z % d < y % d` を証明したいと考えています。
しかし、`integer_ring` は等式にしか対応していないので、`lemma_mod_between` を直接証明することはできません。
その代わりに、証明の助けになる事実を与えます。
補助の補題は、1) `x % d - z % d == x - z  (mod d)` と 2) `z % d - y % d == z - y  (mod d)` を与えます。
証明の残りは `by(nonlinear_arith)` によって行われます。

```rust
pub proof fn lemma_mod_between_helper(x: int, y: int, d: int, small_x:int, small_y:int, tmp1:int) by(integer_ring)
    requires
        small_x == x % d,
        small_y == y % d,
        tmp1 == (small_x - small_y) % d,
    ensures
        (tmp1 - (x-y)) % d == 0
{}

// note that below two facts are from the helper function, and the rest are done by `by(nonlinear_arith)`.
// x % d - z % d == x - z  (mod d)
// z % d - y % d == z - y  (mod d)
pub proof fn lemma_mod_between(d: int, x: int, y: int, z: int) by(nonlinear_arith)
    requires
        d > 0,
        x % d < y % d,
        y - x <= d,
        x <= z < y
    ensures
        x % d <= z % d < y % d
{
    let small_x = x % d;
    let small_y = y % d;
    let small_z = z % d;
    let tmp1 = (small_x - small_z) % d;
    lemma_mod_between_helper(x,z,d, small_x, small_z, tmp1);

    let tmp2 = (small_z - small_y) % d;
    lemma_mod_between_helper(z,y,d, small_z, small_y, tmp2);    
}
```


#### 例 2: `integer_ring` の助けを借りて、範囲の限られた整数についての性質を証明する {#example-2-proving-properties-on-bounded-integers-with-the-help-of-integer_ring}

`integer_ring` の証明は `int` にしか対応していないので、範囲の限られた整数についての性質を証明したいときは、明示的な範囲の制約を含める必要があります。
たとえば、次に示すように、証明 `lemma_mod_after_mul` を `u32` に対して使うには、`lemma_mod_after_mul_u32` が、すべての引数を `lemma_mod_after_mul` に渡す前に、それらが適切な範囲に収まっていることを保証しなければなりません。

必要な範囲の制約（たとえば `m > 0`）が含まれていないと、Verus は証明の検証に失敗します。

```rust
proof fn lemma_mod_after_mul(x: int, y: int, z: int, m: int) by (integer_ring)
    requires (x-y) % m == 0
    ensures (x*z - y*z) % m == 0
{}

proof fn lemma_mod_after_mul_u32(x: u32, y: u32 , z: u32, m: u32)   
    requires
        m > 0,
        (x-y) % (m as int) == 0,
        x >= y,
        x <= 0xffff,
        y <= 0xffff,
        z <= 0xffff,
        m <= 0xffff,
    ensures (x*z - y*z) % (m as int) == 0
{ 
  lemma_mod_after_mul(x as int, y as int, z as int, m as int);
  // rest of proof body omitted for space
}
```

`nat` についての目的の性質も、同じように証明できます。

次の例も似ていますが、integer_ring の証明を助けるために、追加の変数（`ac`、`bc`、`abc`）をいくつか導入していることに注意してください。

```rust
pub proof fn multiple_offsed_mod_gt_0_helper(a: int, b: int, c: int, ac: int, bc: int, abc: int) by (integer_ring)
    requires
        ac == a % c,
        bc == b % c,
        abc == (a - b) % c,
    ensures (ac - bc - abc) % c == 0
{}

pub proof fn multiple_offsed_mod_gt_0(a: nat, b: nat, c: nat) by (nonlinear_arith) 
    requires
        a > b,
        c > 0,
        b % c == 0,
        a % c > 0,
    ensures (a - b) % (c as int) > 0
{
    multiple_offsed_mod_gt_0_helper(
      a as int, 
      b as int, 
      c as int, 
      (a % c) as int, 
      (b % c) as int, 
      ((a - b) % (c as int)) as int
    );
}
```

`integer_ring` の例は、[このフォルダー](https://github.com/verus-lang/verus/tree/main/examples/integer_ring)と、この[テストケースのファイル](https://github.com/verus-lang/verus/blob/main/source/rust_verify_test/tests/integer_ring.rs)にもっとあります。

### 符号化を調べる {#examining-the-encoding}

Singular への問い合わせは、`--log-dir` で指定したディレクトリ（既定では `.verus-log`）の中の、そのファイルを含むモジュールの `.air` ファイルに記録されます。
{% endraw %}

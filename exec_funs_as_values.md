{% raw %}
# 関数を値として渡す {#passing-functions-as-values}

Rust では、`FnOnce`、`FnMut`、`Fn` の各トレイトを使って、関数を値として渡すことができます。Verus は、通常の関数と同じように、こうした関数の事前条件と事後条件についての推論に対応しています。

### 事前条件と事後条件についての推論 {#reasoning-about-preconditions-and-postconditions}

Verus では、`call_requires` と `call_ensures` という 2 つの組み込みの spec 関数を通じて、関数値の事前条件と事後条件について推論できます。

 * `call_requires(f, args)` は事前条件を表します。
    引数は 2 つで、関数オブジェクトと、タプルにまとめた引数を取ります。これが true を返すなら、与えた引数で `f` を呼び出すことができます。
 * `call_ensures(f, args, output)` は事後条件を表します。
    引数は _3 つ_ で、関数オブジェクト、引数、戻り値を取ります。
    これは、`f` に対する入力と出力の正しい組を表します。

`vstd` ライブラリは、`f.requires(args)` と `f.ensures(args, output)` という[別名も提供しています](https://verus-lang.github.io/verus/verusdoc/vstd/pervasive/trait.FnWithRequiresEnsures.html)。これらは `call_requires` と `call_ensures` と同じ意味です。

通常の呼び出しと同じように、Verus は、関数オブジェクトを呼び出すときに事前条件が満たされていることを要求します。次の例でそれを示します。

```rust
    fn double(x: u8) -> (res: u8)
        requires
            0 <= x < 128,
        ensures
            res == 2 * x,
    {
        2 * x
    }

    fn higher_order_fn(f: impl Fn(u8) -> u8) -> (res: u8) {
        f(50)
    }

    fn test() {
        higher_order_fn(double);
    }
```

見てのとおり、`test` は `double` を渡して `higher_order_fn` を呼び出します。そして `higher_order_fn` は、その引数を `50` で呼び出します。`double` の `requires` 節に照らせば、これは許されるはずです。しかし `higher_order_fn` は、これが正しいと分かるだけの情報を持っていません。Verus はエラーを出します。

```
error: Call to non-static function fails to satisfy `callee.requires(args)`
  --> vec_map.rs:25:5
   |
25 |     f(50)
   |     ^^^^^
```

これを直すには、`higher_order_fn` に、`f` の事前条件についての情報を与える事前条件を追加します。

```rust
    fn double(x: u8) -> (res: u8)
        requires
            0 <= x < 128,
        ensures
            res == 2 * x,
    {
        2 * x
    }

    fn higher_order_fn(f: impl Fn(u8) -> u8) -> (res: u8)
        requires
            call_requires(f, (50,)),
    {
        f(50)
    }

    fn test() {
        higher_order_fn(double);
    }
```

`(50,)` は少し変わった見た目をしています。これは 1 要素のタプルです。`call_requires` と `call_ensures` は、「args」として常にタプルの引数を取ります。`f` が引数を 0 個取るなら、`call_requires` はユニットのタプルを取ります。`f` が引数を 2 個取るなら、ペアを取ります。以下同様です。ここでは `f` が引数を 1 個取るので、1 要素のタプルを取ります。これは、`(50,)` のように末尾にカンマを付けることで作れます。

これで Verus はこのコードを受け入れます。`higher_order_fn` の事前条件が、`f` が入力 `50` を受け付けることを保証するようになったからです。

さらに進めて、`higher_order_fn` が `f` の _出力_ の値について推論できるようにすることもできます。

```rust
    fn double(x: u8) -> (res: u8)
        requires
            0 <= x < 128,
        ensures
            res == 2 * x,
    {
        2 * x
    }

    fn higher_order_fn(f: impl Fn(u8) -> u8) -> (res: u8)
        requires
            call_requires(f, (50,)),
            forall|x, y| call_ensures(f, x, y) ==> y % 2 == 0,
        ensures
            res % 2 == 0,
    {
        let ret = f(50);
        return ret;
    }

    fn test() {
        higher_order_fn(double);
    }
```

`higher_order_fn` の事前条件が、`f` の事後条件に制約を課していることに注目してください。その結果、`higher_order_fn` は `f(50)` の戻り値についての情報を得ます。具体的には、`call_ensures(f, (50,), ret)` が成り立つことを知り、これは `higher_order_fn` の事前条件によって `ret % 2 == 0` を含意します。

### 重要な注意 {#an-important-note}

上の例は、関数の引数の事前条件と事後条件に制約を課すときの、慣用的な書き方を示しています。`call_requires` が _正_ の位置で使われていることに注目してください。つまり「この値について `call_requires` が成り立つ」という形です。一方で `call_ensures` は _負_ の位置、つまり含意の左辺で使われています。「ある値について `call_ensures` が成り立つなら、その値はこの特定の制約を満たす」という形です。

`f(args)` がある特定の値、たとえば `expected_return_value` を返すという保証が必要になることは、とてもよくあります。このような場面では、次のような制約を書きたくなるかもしれません。

```rust
requires call_ensures(f, args, expected_return_value),
```

しかし、**これが本当に望んでいるものであることは、まずありません**。それどころか、Verus がこれを証明させてくれないこともあります。`call_ensures(f, args, expected_return_value)` という命題は、`expected_return_value` が `f(args)` の戻り値として _ありうる_ と述べています。しかし、_ほかの_ ありうる戻り値については何も述べていません。一般に、`f` は非決定的かもしれないのです。`expected_return_value` がありうる戻り値の 1 つだからといって、それが唯一のものだということにはなりません。

このような場面で、**本当に書きたいのは次のものです**。

```rust
requires forall |ret| call_ensures(f, args, ret) ==> ret == expected_return_value
```

これこそが本当に望んでいる命題です。つまり、「_もし_ `f(args)` が値 `ret` を返すなら、その値は `expected_return_value` に等しい」という命題です。

もちろん、事後条件を書くときには、これが逆向きになります。次の例で見ていきます。

### 例: `vec_map` {#example-vec_map}

学んだことを使って、`vec_map` という単純な関数を書いてみましょう。これは、与えられた関数をベクタのそれぞれの要素に適用し、新しいベクタを返します。

鍵となる課題は、使うべき正しい仕様を見定めることです。

欲しいシグネチャは次のとおりです。

```rust
fn vec_map<T, U>(v: &Vec<T>, f: impl Fn(T) -> U) -> (result: Vec<U>) where
    T: Copy,
```

まず、何を **require**（要求）する必要があるでしょうか。ベクタのどの要素を入力としても `f` を呼び出してよい、ということを要求する必要があります。

```rust
    requires
        forall|i|
            0 <= i < v.len() ==> call_requires(
                f,
                (v[i],),
            ),
```

次に、何を **ensure**（保証）すべきでしょうか。当然、返されるベクタは入力と同じ長さであってほしいです。さらに、出力のベクタのどの要素も、与えられた関数 `f` を入力のベクタの対応する要素で呼び出したときにありうる出力であることを、保証したいです。

```rust
    ensures
        result.len() == v.len(),
        forall|i|
            0 <= i < v.len() ==> call_ensures(
                f,
                (v[i],),
                #[trigger] result[i],
            )
        ,
```

仕様ができたので、実装とループ不変条件は自然に決まるはずです。

```rust
fn vec_map<T, U>(v: &Vec<T>, f: impl Fn(T) -> U) -> (result: Vec<U>) where
    T: Copy,

    requires
        forall|i|
            0 <= i < v.len() ==> call_requires(
                f,
                (v[i],),
            ),
    ensures
        result.len() == v.len(),
        forall|i|
            0 <= i < v.len() ==> call_ensures(
                f,
                (v[i],),
                #[trigger] result[i],
            )
        ,
{
    let mut result = Vec::new();
    let mut j = 0;
    while j < v.len()
        invariant
            forall|i| 0 <= i < v.len() ==> call_requires(f, (v[i],)),
            0 <= j <= v.len(),
            j == result.len(),
            forall|i| 0 <= i < j ==> call_ensures(f, (v[i],), #[trigger] result[i]),
    {
        result.push(f(v[j]));
        j += 1;
    }
    result
}
```

最後に、例で試してみましょう。

```rust
fn double(x: u8) -> (res: u8)
    requires
        0 <= x < 128,
    ensures
        res == 2 * x,
{
    2 * x
}

fn test_vec_map() {
    let mut v = Vec::new();
    v.push(0);
    v.push(10);
    v.push(20);
    let w = vec_map(&v, double);
    assert(w[2] == 40);
}
```

### まとめ {#conclusion}

この章では、高階の仕様を持つ高階関数の書き方を学びました。高階の仕様とは、値として受け渡される関数の仕様に制約を課す仕様のことです。

この章の例はどれも、関数を名前で直接指して渡していました。たとえば、`double` と書くことで関数 `double` を渡していました。Rust で高階関数を扱うときには、_クロージャ_ を渡すほうが一般的です。次の章では、クロージャの使い方を学びます。
{% endraw %}

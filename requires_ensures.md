{% raw %}
# 事前条件（requires 節） {#preconditions-requires-clauses}

小さな例から始めましょう。
数を 8 倍する関数 `octuple` を検証したいとします。

```rust
fn octuple(x1: i8) -> i8 {
    let x2 = x1 + x1;
    let x4 = x2 + x2;
    x4 + x4
}
```

このコードの検証を Verus に頼むと、Verus はすぐに `octuple` についてのエラーを報告します。

```
error: possible arithmetic underflow/overflow
   |
   |     let x2 = x1 + x1;
   |              ^^^^^^^
```

ここで Verus は、`x1 + x1` の結果が 8 ビットの `i8` の値に収まることを証明できません。
`i8` で表せる値の範囲は `-128`...`127` です。
たとえば `x1` が `100` なら、`x1 + x1` は `200` になり、`127` より大きくなります。
そこで、`octuple` が呼び出されるときに、引数 `x1` が大きすぎないようにする必要があります。
そのためには、`octuple` に*事前条件*（precondition。「`requires` 節」とも呼ばれます）を追加して、`x1` に許される値を指定します。
Verus では、事前条件は `requires` に続けて、カンマで区切った 0 個以上の真偽値の式を並べて書きます。

```rust
fn octuple(x1: i8) -> i8
    requires
        -64 <= x1,
        x1 < 64,
{
    let x2 = x1 + x1;
    let x4 = x2 + x2;
    x4 + x4
}
```

上の 2 つの事前条件は、x1 が `-64` 以上で `64` 未満でなければならないと述べています。
こうすれば、`x1 + x1` は `-128`...`127` の範囲に収まります。
これで `x1 + x1` についてのエラーは直りますが、まだ `x2 + x2` についてのエラーが出ます。

```
error: possible arithmetic underflow/overflow
   |
   |     let x4 = x2 + x2;
   |              ^^^^^^^
```

`x1 + x1`、`x2 + x2`、`x4 + x4` のすべてを成功させたいなら、`x1` にもっと厳しい範囲が必要です。

```rust
fn octuple(x1: i8) -> i8
    requires
        -16 <= x1,
        x1 < 16,
{
    let x2 = x1 + x1;
    let x4 = x2 + x2;
    x4 + x4
}
```

今度は、検証に成功します。

では、`octuple` の事前条件を満たさない値で `octuple` を呼び出そうとしたとしましょう。

```rust
fn main() {
    let n = octuple(20);
}
```

この呼び出しに対して、Verus はエラーを報告します。`20` は `16` 未満ではないからです。

```
error: precondition not satisfied
   |
   |         x1 < 16,
   |         ------- failed precondition
...
   |     let n = octuple(20);
   |             ^^^^^^^^^^^
```

`20` の代わりに `10` を渡すと、検証に成功します。

```rust
fn main() {
    let n = octuple(10);
}
```

# 事後条件（ensures 節） {#postconditions-ensures-clauses}

`octuple` が返す値についての性質を検証したいとします。
たとえば、`octuple` が返す値が、`octuple` に渡した引数の 8 倍であることをアサートしたいかもしれません。
`main` の中に、`octuple(10)` を呼び出した結果が `80` であるというアサーションを置いてみましょう。

```rust
fn main() {
    let n = octuple(10);
    assert(n == 80);
}
```

`octuple(10)` は実際に `80` を返すのですが、それでも Verus はエラーを報告します。

```
error: assertion failed
   |
   |     assert(n == 80);
   |            ^^^^^^^ assertion failed
```

このエラーが起きるのは、`octuple` が引数を `8` 倍しているにもかかわらず、その事実をプログラム内のほかの関数に公開していないからです。
公開するには、`octuple` に事後条件（postcondition。`ensures` 節）を追加して、`octuple` の戻り値の性質を指定します。

```rust
fn octuple(x1: i8) -> (x8: i8)
    requires
        -16 <= x1,
        x1 < 16,
    ensures
        x8 == 8 * x1,
{
    let x2 = x1 + x1;
    let x4 = x2 + x2;
    x4 + x4
}
```

戻り値についての性質を書くには、戻り値に名前を付ける必要があります。
そのための Verus の構文は `-> (name: return_type)` です。
上の例では、`-> (x8: i8)` と書くことで、事後条件 `x8 == 8 * x1` の中で、`octuple` の戻り値を `x8` という名前で使えるようになります。

事前条件と事後条件は、関数どうしの間に、モジュール化された検証の取り決めを作ります。
`main` が `octuple` を呼び出すとき、Verus は、その呼び出しの引数が `octuple` の事前条件を満たすことを検査します。
Verus が `octuple` 関数の本体を検証するときは、事前条件が満たされていると仮定できます。
`main` が実際にどんな引数を渡すかについては、何も知る必要がありません。
同じように、Verus が `main` 関数の本体を検証するときは、`octuple` がその事後条件を満たすと仮定できます。
`octuple` の本体については、何も知る必要がありません。
このようにして、Verus は各関数を独立に検証できます。
この*モジュール化された検証*（modular verification）の方法は、検証を小さく扱いやすい部分に分割します。
そのため、プログラムのすべての関数をまとめて同時に検証しようとする場合よりも、検証が効率的になります。
とはいえ、事前条件と事後条件を書くには、プログラマーの相当な労力が必要です。
多くの関数を持つ大きなプログラムを検証したいなら、おそらく、関数の事前条件と事後条件を書くのにかなりの時間を費やすことになるでしょう。

# assert と assume {#assert-and-assume}

`requires` と `ensures` が関数どうしをつなぐのに対して、`assert` は、ある事実を証明するよう SMT ソルバーにその場限りの内輪の依頼をします。
（注意: `assert(...)` を、Rust の `assert!(...)` マクロと混同しないでください。
前者は SMT ソルバーを使って静的に検査され、後者は実行時に検査されます。）

`assert` には、`assume` という名前の邪悪な双子がいます。
`assume` は、ある真偽値の式を、証明なしに事実としてそのまま受け入れるよう SMT ソルバーに求めます。
`assert` は無害で、証明に不健全性を持ち込むことはありません。
一方、assume を使うと、真ではない事実の「証明」が簡単にできてしまいます。
実際、`assume(false)` と書けば、好きなことを何でも証明できます。

```rust
assume(false);
assert(2 + 2 == 5); // succeeds
```

Verus のプログラマーは、証明の開発とデバッグを助けるために、`assert` と `assume` をよく使います。
一時的な `assert` を追加して、SMT ソルバーがどの事実を証明でき、どの事実を証明できないのかを調べることがあります。
また、一時的な `assume` を追加して、SMT ソルバーが証明を完了するのにどんな追加の仮定が必要なのかを調べたり、まだ書いていない証明の部分の仮置きにしたりすることもあります。
証明ができあがっていくにつれて、プログラマーは `assume` を `assert` に置き換えていき、最終的にはその `assert` も取り除くかもしれません。
完成した証明は、`assert` を含んでいてもかまいませんが、`assume` を含んでいてはいけません。
（これを Verus に強制させるには、[no-cheating モード](./tcb.html#no-cheating-mode)を参照してください。
このモードは、明示的に信頼されたコードの外に残っている `assume` を拒否します。）

（場合によっては、`assert` は SMT ソルバーが証明を完了するのを助けることがあります。
[`forall` や `exists` の式をどう扱うか](forall.html)についての手がかりを SMT ソルバーに与えるからです。
また、`assert(...) by(bit_vector)` のような特別な形の `assert` もあります。
これらは、[ビットベクトル](bitvec.html)、[非線形の整数演算](nonlinear.html)、[`forall` 式](quantproofs.html)などについての性質を証明するのに役立ちます。）

# 実行可能コードとゴーストコード {#executable-code-and-ghost-code}

この節で扱ったことをすべてまとめて、例のプログラムの最終版にしましょう。

```rust
use vstd::prelude::*;

verus! {

#[verifier::external_body]
fn print_two_digit_number(i: i8)
    requires
        -99 <= i < 100,
{
    println!("The answer is {}", i);
}

fn octuple(x1: i8) -> (x8: i8)
    requires
        -16 <= x1 < 16,
    ensures
        x8 == 8 * x1,
{
    let x2 = x1 + x1;
    let x4 = x2 + x2;
    x4 + x4
}

fn main() {
    let n = octuple(10);
    assert(n == 80);
    print_two_digit_number(n);
}

} // verus!
```

ここでは、最後の調整をいくつか行っています。

1 つ目に、2 つの事前条件 `-16 <= x1` と `x1 < 16` を、1 つの事前条件 `-16 <= x1 < 16` にまとめました。
Verus では、複数の不等式を 1 つの式の中でつなげて書けるからです
（同じ意味で、`-16 <= x1 && x1 < 16` と書くこともできました）。

2 つ目に、`octuple` の結果を表示する関数 `print_two_digit_number` を追加しました。
`main` や `octuple` とは違い、`print_two_digit_number` は検証しないよう Verus に頼んでいます。
そのために、この関数に `#[verifier::external_body]` という印を付けています。
こうすると、Verus は関数の事前条件と事後条件には注意を払いますが、関数の本体は無視します。
これは Verus を使うプロジェクトではよくあることです。
一部（おそらくプログラムの中核となるアルゴリズム）は検証したいけれど、入出力の操作のようなほかの部分は検証しないままにしておきたい、ということがあります。
より一般的に言えば、世界中のソフトウェアをすべて検証することはまだ現実的ではないので、検証されたコードと検証されていないコードの間には何らかの境界ができます。
`#[verifier::external_body]` は、この境界に印を付けるのに使えます。

これで、Verus の `--compile` オプションを使って、上のプログラムをコンパイルできます。

```
./target-verus/release/verus --compile ../examples/guide/requires_ensures.rs
```

これにより、実行するとメッセージを表示する実行ファイルが作られます。

```
The answer is 80
```

生成された実行ファイルには、`requires`、`ensures`、`assert` のコードが含まれていないことに注意してください。
これらは静的な検証のときにだけ必要で、実行時には必要ないからです。
`requires`、`ensures`、`assert`、`assume` を*ゴーストコード*（ghost code）と呼びます。
これは、実際にコンパイルされる*実行可能コード*（executable code）と対比した呼び方です。
Verus はコンパイルの前にすべてのゴーストコードを消去するので、ゴーストコードが実行時の負担になることはありません。
{% endraw %}

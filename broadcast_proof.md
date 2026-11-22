{% raw %}
# `broadcast` で証明環境に常時有効な事実を加える {#adding-ambient-facts-to-the-proof-environment-with-broadcast}

典型的な Verus のプロジェクトでは、開発者はある事実（たとえば、列に要素を追加しても、すでに列にある要素は変わらないこと）を、次のように証明関数の中で証明するでしょう。
```rust
pub proof fn seq_contains_orig_elems_after_push<A>(s:Seq<A>, v:A, x:A)
    requires s.contains(x)
    ensures s.push(v).contains(x)
{
  ...
}
```
この事実を使うには、開発者は証明関数を明示的に呼び出さなければなりません。
たとえば次のとおりです。
```rust
proof fn example<A>(s: Seq<A>, v: A, x: A) {
    assume(s.contains(x));
    let t = s.push(v);
    // assert(t.contains(x)); // FAILS
    seq_contains_orig_elems_after_push(s, v, x); // Adds the proof's fact to the proof environment
    assert(t.contains(x)); // SUCCEEDS
  }
```
しかし場合によっては、ある証明済みの事実がとても役に立つので、対応する証明を手で呼び出さなくても、いつでもスコープに入っていてほしいと開発者が考えることがあります。
たとえば、空の列の長さが 0 であるという事実はあまりにも「自明」なので、ほとんどのプログラマーは、Verus がそれを常に知っているものと期待するでしょう。
ただし、この機能は慎重に使うべきです。常時有効な（ambient）事実が 1 つ増えるたびに、証明器の全体的な性能が落ちるからです。

ソルバーの性能への影響を検討したうえで、プログラマーが `push` についての上の事実を常時有効にすると決めたとしましょう。
そのためには、`seq_contains_orig_elems_after_push` の定義に `broadcast` 修飾子を付けて、`pub broadcast proof fn seq_contains_orig_elems_after_push<A>(s: Seq<A>, v:A, x:A)` とします。
その効果として、量化子の付いた次の事実が証明環境に導入されます。
```rust
forall |s: Seq<A>, v: A, x: A| s.contains(x) ==> s.push(v).contains(x)
```
これは量化子を導入するので、Verus はたいてい、たとえば `#[trigger]` 注釈を付けるなどして、トリガーを明示的に選ぶよう求めます。
したがって、この例の最終版は次のようになるでしょう。
```rust
pub broadcast proof fn seq_contains_orig_elems_after_push<A>(s:Seq<A>, v:A, x:A)
    requires s.contains(x)
    ensures #[trigger] s.push(v).contains(x)
{
    ...
}
```

この常時有効な補題を、特定の証明や、モジュール全体のスコープに入れるには、`broadcast use seq_contains_orig_elems_after_push;` を使います。

こうした broadcast された補題の一部は、Verus の標準ライブラリ `vstd` で提供されています。
その中には、broadcast の「グループ」の一部になっているものもあります。グループは、いくつもの性質を 1 つのグループ名にまとめたもので、`broadcast use broadcast_group_name;` でスコープに入れられます。
たとえば、上の例は `broadcast use vstd::seq_lib::group_seq_properties;` だけですべて自動的に証明できます。
標準ライブラリのドキュメントでこれらのグループを見つけやすくする作業を進めているところです。現在のところ、グループは通常の関数として表示されます。

## 実験的な機能: broadcast 補題の使用状況の情報 {#experimental-broadcast-lemma-usage-information}

実験的なフラグ `-V axiom-usage-info` を使うと、各関数の検証で使われた broadcast 済みの公理と補題を、多めに見積もった形（過大近似）で得られます。
大きなプロジェクトでは、`--verify-only-module` や、場合によっては `--verify-function` を使って、出力の量を絞ってください。

例として、[examples/broadcast_proof.rs](https://github.com/verus-lang/verus/blob/main/examples/broadcast_proof.rs) に対して `-V axiom-usage-info` を使うと、`increase_twice` 関数について次の情報が出力されます。

```
note: checking this function used these broadcasted lemmas and broadcast groups:
        - (group) broadcast_proof::multiple_broadcast_proof::Multiple::group_properties,
        - broadcast_proof::multiple_broadcast_proof::Multiple::lemma_add_aligned
   --> ../examples/broadcast_proof.rs:161:11
    |
161 |       proof fn increase_twice(
    |  ___________^
162 | |         p1: Multiple, v: Multiple, p2: Multiple)
    | |________________________________________________^
```

これは、`group_properties` グループによって `lemma_add_aligned` が使えるようになり、それがおそらく証明で使われたことを示しています。
{% endraw %}

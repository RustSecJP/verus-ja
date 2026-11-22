{% raw %}
# トリガー注釈 {#trigger-annotations}

Verus は、プログラム中のすべての量化子の式（`forall`、`exists`、`choose`）に、トリガー（trigger）の集合を割り当てます。`broadcast` 補題にあるような「暗黙の」量化子（quantifier）も対象です。

トリガーは証明の自動化にさまざまな影響を与えるので、Verus の開発者はそれを知っておく必要があります。
[ガイドの該当する章](./quants.html)、とくに[複数のトリガーとマッチングループ](./multitriggers.html)の節を参照してください。

このページでは、Verus がソースコードからこれらのトリガーを決める手順を説明します。

### 構文 {#syntax}

```verus-grammar
V@[root_trigger_attr_expr]  ::= #![trigger V@[spec_expr] (, V@[spec_expr])*]
                          | #![auto]
                          | #![all_triggers]
V@[inner_trigger_attr_expr] ::= #[trigger] V@[spec_expr]
                          | #[trigger(R@[n])] V@[spec_expr]
```

V@[root_trigger_attr_expr] の注釈は、量化子の本体の先頭に現れ、量化子全体のトリガー選択を設定します。
V@[inner_trigger_attr_expr] の注釈は、本体の中の特定の部分式に付き、それらを個別にトリガーとして指定します。

### 意味論 {#semantics}

トリガー注釈は、spec 式の意味論にまったく影響しません。
トリガーが影響するのは、自動定理証明器が探索する証明の空間だけです。

## 用語: トリガーグループとトリガー式 {#terminology-trigger-groups-and-trigger-expressions}

すべての量化子は、いくつかの*量化変数*（quantifier variable）を持ちます。
ソルバーがこれらの変数をどのように具体化（instantiate）するかを制御するために、トリガーグループが使われます。

 * Verus は、すべての量化子に対して、*トリガーグループ*（trigger group）の集まりを決めます。
 * それぞれの*トリガーグループ*は、*トリガー式*（trigger expression）の集まりです。

必然的に、どのトリガーグループも、すべての量化変数がグループ内の少なくとも 1 つのトリガー式で使われている場合にだけ、整形式（well-formed）になります。

次の点に注意してください。

 * SMT ソルバーは、トリガーグループの*どれか 1 つ*でも発火すれば、量化子を具体化します。
 * ただし、トリガーグループが発火するのは、グループ内の*すべての*式が一致したときだけです。

したがって、

 * トリガーグループが多いほど、量化子は具体化される頻度が*高く*なります。
 * トリガー式が多いトリガーグループほど、発火する頻度は*低く*なります。

## トリガーグループの選択 {#selecting-trigger-groups}

Verus は、トリガーグループの集まりを次のように決めます。

 * Verus は、量化子の本体の中から、適用対象となるすべての `#[trigger]` 注釈と `#[trigger(n)]` 注釈を探します。
      * 量化子が入れ子になっている場合、それぞれの `#[trigger]` 注釈や `#[trigger(n)]` 注釈は、ちょうど 1 つの量化子の式に適用されます。それは、トリガー式が使っている変数を束縛する*もっとも内側*の量化子です。
 * `#[trigger]` が付いた適用対象の式すべてが、1 つのトリガーグループになります。
 * 同じ `n` の `#[trigger(n)]` が付いた適用対象の式すべてが、1 つのトリガーグループになります。
 * 量化子の式の先頭にある注釈 <code>#![trigger EXPR<sub>1</sub>, ..., EXPR<sub>k</sub>]</code> のそれぞれが、1 つのトリガーグループになります。
 * 以上をすべて行ってもトリガーグループが 1 つも見つからなかった場合、Verus は、量化子の式の本体に基づいて、ヒューリスティックでトリガーグループを決める*ことがあります*。
     * `#![all_triggers]` が指定されている場合、Verus は「積極的な」戦略を使い、本体から適用可能だと合理的に推測できるトリガーグループをすべて選びます。
     * `#![auto]` が指定されている場合、Verus は「保守的な」戦略を使い、さまざまなヒューリスティックで最適だと判断されたトリガーグループを 1 つだけ選びます。
     * `#![all_triggers]` も `#![auto]` も指定されていない場合、Verus は `#![auto]` の場合と同じ「保守的な」戦略を使います。
 * 以上をすべて行っても Verus がトリガーグループを 1 つも見つけられなかった場合は、エラーになります。

## トリガーのログ出力 {#trigger-logging}

既定では、Verus のヒューリスティックが選んだトリガーグループに「自信が無い」場合、Verus はしばしば、選んだトリガーについての詳しい情報を表示します。
`#![auto]` 属性を使うと、この情報を個別に抑止できます。
量化子に `#![auto]` を付けると、Verus に自信が無い場合でも自動的に選ばれたトリガーを使いたい、と Verus に伝えることになります。その場合、このログ出力は抑止されます。

この動作は、コマンドラインで設定できます。

<div class="table-wrapper">
<table>
<colgroup>
   <col span="1" style="width: 30%;">
   <col span="1" style="width: 70%;">
</colgroup>
<thead><tr><th>オプション</th><th>動作</th></tr></thead>
<tbody>
<tr><td><code>--triggers-mode silent</code></td><td>自動的に選ばれたトリガーを表示しません</td></tr>
<tr><td><code>--triggers-mode selective</code></td><td><strong>既定。</strong> ヒューリスティックに自信が無く、かつ <code>#![auto]</code> が指定されていない場合にだけ、自動的に選ばれたトリガーを表示します</td></tr>
<tr><td><code>--triggers</code></td><td>検証対象のモジュールについて、自動的に選ばれたトリガーをすべて表示します</td></tr>
<tr><td><code>--triggers-mode verbose</code></td><td>検証対象のモジュールと、ほかのモジュールからインポートした定義について、自動的に選ばれたトリガーをすべて表示します</td></tr>
</tbody></table>
</div>

トリガーのログ出力に関するそのほかのオプションは、`verus --help` を参照してください。
{% endraw %}

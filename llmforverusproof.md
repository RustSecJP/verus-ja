{% raw %}
# LLM を使って Verus の証明を書く作業を助ける {#using-llms-to-help-write-verus-proofs}

## クイックスタート: お勧めの構成 {#quick-start-recommended-setup}

最良の結果を得るために、次の構成をお勧めします。

1. モデルの API を直接呼び出すのではなく、**コーディングエージェントを使う**
2. モデルが参照できるように、**Verus の資料（vstd など）を与える**
3. モデルが実行してエラーメッセージを見られるように、**Verus のバイナリを与える**
4. 不正な証明の近道を見つけるために、**不正チェッカー（cheat checker）を使う**

これらが整っていれば、複雑なプロンプトは必要ありません。次のような簡単なプロンプトで、うまくいくことがよくあります。

```
The file X.rs cannot be verified by Verus yet.
Please add proof annotations so that it can be successfully verified by Verus.
The vstd folder contains Verus standard library definitions and helper lemmas.
Keep editing your proof until Verus shows no errors.
Do NOT change existing specifications (requires/ensures) or executable code.
Do NOT use assume(...) or admit(...).
Before you finish, run the cheat checker to make sure you haven't cheated.
```

## コーディングエージェントを使う {#use-a-coding-agent}

**なぜコーディングエージェントなのか。** コードを書くことと、コードについての証明を書くことには、似た点がたくさんあります。どちらも既存のコードベースをよく理解している必要があり、どちらも道具の支援から恩恵を受けられます。たとえば、コードの合成にはテストツールが、証明の合成には検証器が役立ちます。これまでの実践から、最先端の LLM は、コードベースを探索し、ツールを実行し、自分のペースで繰り返し作業できるときに最もよく働くことが分かっています。[GitHub Copilot CLI](https://githubnext.com/projects/copilot-cli) のような既存のコーディングエージェントは、これをしっかり支えてくれます。

**実践的なヒント**
- エージェントは、安全なコンテナやサンドボックスの中で動かすことをお勧めします。そうすれば、たとえば `--allow-all-tools`（Copilot CLI）や同様のオプションを使って、人が何度も介入しなくても、Verus やそのほかの利用可能なツールを実行する自由を安全に与えられます。

## Verus の資料を与える {#provide-verus-resources}

Verus の標準ライブラリ（[`vstd`](https://github.com/verus-lang/verus/tree/main/source/vstd)）、Verus のテストスイート（[`rust_verify_test`](https://github.com/verus-lang/verus/tree/main/source/rust_verify_test)）、Verus の[ガイド](https://github.com/verus-lang/verus/tree/main/source/docs/guide/src/)といった既存の Verus の資料に LLM がアクセスできるようにすると、次のような利点があります。

1. **Verus の構文を学べる**: これらの資料を調べることで、LLM は正しい Verus の書き方を学べます。これは、ときどき構文エラーを出すモデルにとくに役立ちます。

2. **補助的な補題を見つけられる**: `vstd` ライブラリには、証明を簡単にしたり、証明を可能にしたりする便利な補題がたくさん含まれています。LLM に `vstd` へのアクセスを与えないと、どんな補題が使えるのかについて、存在しないものをでっち上げる（ハルシネーションを起こす）ことがあります。

3. **最新の機能を学べる**: `rust_verify_test` は、Verus に追加されたばかりの機能を LLM が学ぶのに最適な資料です。

次に示すのは、[`lemma_len_intersect`](./assert_assume.html) という補題を証明するよう与えたときの、GPT-5-mini の推論ログの一部です。

```
● Web Search (MCP: github-mcp-server) · Verus set len subset injective_map "Verus" "Set" len subset…
  └ ...

● Edit lemma_len_intersect.rs +1 -1

● Run verus on modified lemma_len_intersect.rs to see if pervasive::set::len_mono exists (shell)
  └ ...

● Edit lemma_len_intersect.rs +1 -1

● Run verus on modified lemma_len_intersect.rs to see if vstd::set::len_mono exists (shell)
  └ ...

● Edit lemma_len_intersect.rs +1 -1

● Run verus to check if subset_len exists (shell)
  └ ...

● Edit lemma_len_intersect.rs +1 -1

● Run verus to check if len_le_of_subset exists (shell)
  └ ...
```

ご覧のとおり、`vstd` が与えられていないと、モデルは苦戦し、ハルシネーションを起こすことがあります。

## Verus のバイナリを与える {#provide-a-verus-binary}

LLM は、Verus を呼び出して、そのエラー出力を見られるようになっているべきです。Verus のエラーメッセージは情報が豊富で、証明の開発を進める手がかりになります。複雑な証明の課題に取り組むとき、現在の最も有能なモデルであっても、正しくて十分な証明を 1 回の試みで生成できることはめったにありません。Verus のエラー報告を見られなければ、モデルは、不完全な証明や間違った証明を直すのに運に頼ることになります。

次に示すのは、`test.rs` にある非常に難しい補題の関数を検証するよう依頼したときの、Claude Opus 4.5（非常に強力な推論モデル）の推論ログの一部です。
```
Let me run Verus to see what the errors are:

● Run Verus to see initial errors (shell)
  │ cd /usr/src/work && verus test.rs --rlimit 100 2>&1 | head -200
  └ 46 lines...

Now I need to add proof annotations to help Verus verify this lemma. Let me look at the vstd folder for any useful lemmas:
...
... 

● Run Verus to check proof (shell)
  │ cd /usr/src/tmp && verus test.rs --rlimit 150 2>&1 | head -400
  └ 94 lines...

There are several issues. Let me take a more careful approach by understanding what needs to be proven. Let me simplify the proof and add the necessary assertions more carefully:

... 
...

● Run Verus to check proof (shell)
  │ cd /usr/src/tmp && verus test.rs --rlimit 150 2>&1 | head -400
  └ 16 lines...

I see there are issues with imports. Let me look at the file structure more carefully:

...
```

Verus を繰り返し実行し、得られた Verus のエラー報告を踏まえて考え、それに応じて証明注釈を編集する、というこの作業の流れは、人間の開発者にとっても LLM にとってもよい習慣です。

**要点**
- Verus を実行するための簡単なコマンド（たとえば `verus file.rs`）を与えます
- 成功か失敗かだけでなく、エラー出力の全体をモデルが見られるようにします（コーディングエージェントを使う場合は、既定でそうなります）
- `--expand-errors` フラグを使うと、さらに詳しいフィードバックが得られます

## 不正チェッカーを使う {#use-a-cheat-checker}

明示的な歯止めが無いと、LLM は証明を通すために「不正」をすることがあります。これは、証明の課題が難しいときによく起こります。

LLM が不正をする方法はたくさんあります。ここでは、典型的な不正の手口をいくつか挙げるにとどめます。

| 不正の種類 | どのように見えるか |
|------------|-------------------|
| `assume()` または `admit()` を使う | 証明なしに性質を仮定する |
| `external_body` タグを追加する | 関数の本体の検証を飛ばす |
| `axiom` タグを追加する | 証明なしに補題を仮定する |
| 仕様を変更する | 事前条件を強める、または事後条件を弱める |
| 実行可能なコードを変更する | 検証を楽にするために Rust のコードを変更する。これにより、プログラムの意味が変わったり、性能が下がったり、コードの読みやすさが損なわれたりすることがある |

モデルの出力を元のファイルと比べる簡単なツールがあれば、これらのほとんどを見つけられます。最初の 3 つの不正の手口は、LLM が追加した内容の中から ```assume```、```admit```、```external_body```、```axiom``` といった文字列を検索すれば、簡単に特定できます。仕様や実行可能なコードの変更を正確に検査するには、パーサーの支援が必要になるでしょう。

Verus は、最初の 3 つを自動的に強制することもできます。[不正禁止モード（no-cheating mode）](./tcb.html#no-cheating-mode)で実行すると、`assume`、`admit`、`external_body`、`assume_specification` などの構文は、[信頼済みと注釈された](./tcb.html)コードの中に明示的に現れる場合を除いて拒否されます。


## モデルの能力とコストを理解する {#understand-model-capabilities-and-costs}

モデルによって、強み、コスト、速さは異なります。見込んでおくべきことは次のとおりです。

### 成功率 {#success-rates}

証明を合成する能力は、モデルによって大きく異なります。GPT-4o や GPT o4-mini のような古いモデルは、Verus の構文も、検証の一般的な技能も不得意です。これらのモデルを自明でない証明の課題で使えるようにするには、汎用のコーディングエージェント以上の、手の込んだプロンプトの工夫が必要になるでしょう。新しいモデルは、ずっとよい結果を出しています。これまでの実践から、Claude Sonnet 4.5 や Claude Opus 4.5 のようなモデルは、Verus で検証された既存のシステムのプロジェクトにあるほとんどの証明補題について、証明注釈をうまく埋められることが分かっています。


### 時間とお金 {#time-and-money}

LLM を使う前に、そのコストを把握しておくべきです。

コストに影響しうる要因はいくつかあります。
- 一般に、コーディングエージェントを使うほうが、モデルの API を直接呼び出すよりもコストがかかります。コーディングエージェントは、利用者に結果を返すまでに、モデルの API を何度も呼び出すことがあるからです。
- 大きなプロジェクトの中にある証明の課題に LLM を使うと、同じ課題をプロジェクトから取り出して使う場合より、はるかにコストがかかることがあります。LLM は、解くべき問題に落ち着くまでに、どうしてもコードベースを探索するからです。
- 当然ながら、高度なモデルほどコストがかかり、ときにははるかに多くかかります。私たちの経験では、たった 1 つの関数の証明を生成するのに数百ドルかかることがあり、しかもその証明が不完全なこともあります。LLM を使う前に、大きな証明の課題を小さな課題に分解し、必要に応じて実行ごとにトークンの上限を設定するとよいでしょう。
- 推論にかかる時間は、モデルによって大きく異なります。Claude Opus 4.5 は、Claude Sonnet 4.5 と比べて、証明注釈の生成に目に見えて長い時間がかかります。GPT-5 のような一部の古いモデルは、自分の限界を分かっているようで、ごくあっさりと諦めます。これは、使う場面によっては魅力的な特徴になりえます。一部のモデル（たとえば Opus 系列のモデル）は、複雑な証明の課題に対して、コマンドラインオプション --rlimit を通じて Verus に巨大なリソース上限（たとえば 2000）を与えがちで、その結果、Verus が終わるのを何時間も待つことになります。利用者はこの傾向を知っておき、そうするのを避けて、代わりに証明を小さなステップに分けることに集中するよう、モデルに積極的に指示するべきです。
- 高度なモデルは、多くの時間とお金を費やしたあとでも証明全体を完成できないことがありますが、証明の対象を小さくする助けにはなります。これは、すべてのモデルに当てはまるわけではありません。不完全な証明を人間の専門家が引き継げるように、途中の結果を出力し、推論を説明するようモデルに促すのがよい習慣です。

**注意**: コストが大きな懸念である場合は、次のことを検討してください。
- 安価なモデルから始め、必要であれば、より有能なモデルに切り替える
- まず、複雑な証明を小さな部分に分ける
- 汎用のコーディングエージェントを介さず、モデルに直接プロンプトを与える（ただし、多くの手作業が伴います）

### モデルが苦戦する場面 {#when-models-struggle}

LLM は、次のようなものでつまずきがちです。

1. **帰納的な不変条件**: より強い帰納的な性質を見つけることで、状態機械が不変条件を保つことを証明するのは、今でも難しい課題です
2. **手続きマクロの展開**: 手続きマクロが生成したコードは、モデルを混乱させることがあります
3. **重い抽象化**: closed / opaque な spec 関数を理解し、その性質を明らかにするために適切な補題を選ぶ必要があるとき、証明を書くのは難しくなります
4. **非常に大きな課題**: 成功率は、課題の大きさ（コードの行数）と負の相関があります

## LLM の証明は冗長になることがある {#llm-proofs-may-be-verbose}

LLM が生成した証明は、人間が書いた証明より長くなるものと考えておいてください。平均して 2〜3 倍の長さになることがよくあります。こうなる理由は次のとおりです。

1. 定理証明器が自動的に処理できることを、LLM が常に知っているわけではない
2. 結果的には不要だと分かる「念のため」のアサーションを追加しがちである
3. 推論のステップをより明示的に書くことがあり、証明の保守をあまり気にかけない

**例**:
次の証明関数を Sonnet 4.5 に与えたところ、
```rust
pub proof fn myseq_lemma<A>()
    ensures
        forall|s: Seq<A>, v: A, x: A| !s.contains(x) && v != x ==> !s.push(v).contains(x),
{
}
```

次のような証明の本体を出してきました。

```rust
    // Prove the postcondition: if s doesn't contain x and v != x, then s.push(v) doesn't contain x
    assert forall|s: Seq<A>, v: A, x: A| !s.contains(x) && v != x implies !s.push(v).contains(x) by {
        if !s.contains(x) && v != x {
            // Proof by contradiction: assume s.push(v) contains x
            if s.push(v).contains(x) {
                // Then there exists an index j where s.push(v)[j] == x
                let j = choose|j: int| 0 <= j < s.push(v).len() && s.push(v)[j] == x;

                // Case analysis on j
                if j < s.len() {
                    // By axiom_seq_push_index_different: s.push(v)[j] == s[j]
                    assert(s.push(v)[j] == s[j]);
                    // So s[j] == x, which means s contains x - contradiction!
                    assert(s.contains(x));
                    assert(false);
                } else if j == s.len() {
                    // By axiom_seq_push_index_same: s.push(v)[j] == v
                    assert(s.push(v)[j] == v);
                    // So v == x - contradiction with v != x!
                    assert(v == x);
                    assert(false);
                } else {
                    // j >= s.push(v).len(), which contradicts 0 <= j < s.push(v).len()
                    assert(false);
                }
                assert(false);
            }
        }
    };
```

これには何も間違ったところはありません。ただし、空の証明でも同じようにうまくいったはずです。

**この冗長さは**、正しさの面では**たいてい害がありません**が、次のものを増やします。
- トークンのコスト（出力が多いほど、お金がかかります）
- 証明の保守の負担
- 検証時間（わずかに）

検証が成功したあとで、証明注釈を整理するよう LLM に頼んでみることもできます。


## 協力者としての LLM {#llms-as-collaborators}

もちろん、すべての証明を LLM だけに書かせる必要はありません。LLM は、人間の証明開発者の協力者としてよく働きます。

- **部分的な証明から始める**: 人間の証明開発者が証明の骨組みを書き、細かな証明注釈をすべて LLM に仕上げさせることができます。LLM は、人間の証明開発者にとっては退屈かもしれない、決まりきった証明義務を埋めるのが得意なので、これはとくに役立ちます
- **補題の仕様を調整する**: 人間の証明開発者が、大きな証明の課題を小さな補題に分解するのに苦労しているとき、LLM は補題の仕様の調整に役立ちます
- **ライブラリの使い方を提案する**: 尋ねられれば、LLM は、証明を簡単にするのに使える `vstd` やプロジェクト固有の補題のうち、関連するものを見つけるのが得意です

検証済みのシステムの開発において、LLM は強力な助手ですが、設計、仕様、証明の構成についての人間の判断に取って代わるものではありません。

## まとめ {#summary}

| ベストプラクティス | 理由 |
|--------------|-----|
| コーディングエージェントを使う | 証明の開発に欠かせない、観察して調整するという繰り返しが可能になる |
| vstd にアクセスできるようにする | 補題を見つけ、構文を学ぶ助けになる |
| モデルに Verus を実行させる | 繰り返し改良していくには、エラーのフィードバックが欠かせない |
| 不正チェッカーを使う | 不正な証明の近道を防ぐ |
| 予算が許すなら Sonnet 4.5 から始める | 成功率が最も高い |
| 証明が冗長になることを見込んでおく | LLM は説明しすぎる。あとで切り詰められる |
| 人間と LLM の協力 | 人間の洞察と LLM の徹底ぶりを組み合わせる |
{% endraw %}

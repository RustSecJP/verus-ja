{% raw %}
# Verus の概要 {#verus-overview}

Verus は、Rust で書かれたコードの正しさを検証するためのツールです。
主な目標は、低レベルのシステムコードについて、機能面での完全な正しさ（full functional correctness）を検証することです。
そのために、次のような既存の検証フレームワークの考え方を土台にしています。
[Dafny](https://github.com/dafny-lang/dafny)、
[Boogie](https://github.com/boogie-org/boogie)、
[F*](https://github.com/FStarLang/FStar)、
[VCC](https://www.microsoft.com/en-us/research/project/vcc-a-verifier-for-concurrent-c/)、
[Prusti](https://github.com/viperproject/prusti-dev)、
[Creusot](https://github.com/xldenis/creusot)、
[Aeneas](https://github.com/AeneasVerif/aeneas)、
[Cogent](https://github.com/NICTA/cogent)、
[Rocq](https://rocq-prover.org/)、
そして
[Isabelle/HOL](https://isabelle.in.tum.de/overview.html) です。
検証は静的に行われます。Verus は実行時の検査を一切追加しません。
その代わりに、計算機支援の定理証明を使い、実行可能な Rust コードが、利用者の与えた仕様（specification）を、コードのあり得るすべての実行において常に満たすことを静的に検証します。

もう少し詳しく言うと、Verus は次のことを目指しています。
- 仕様を表現するための、純粋に数学的な言語を提供する
  （Dafny、Creusot、F*、Rocq、Isabelle/HOL と同様）
- 証明（proof）を表現するための数学的な言語を提供する
  （Dafny、F*、Rocq、Isabelle/HOL と同様）。
  この言語は古典論理だけに基づく（Dafny と同様）
- 実行可能コードを表現するための、低レベルで命令型の言語を提供する（VCC と同様）。
  この言語は Rust に基づく（Prusti、Creusot、Aeneas と同様）
- [Z3](https://microsoft.github.io/z3guide/docs/logic/intro) のような SMT ソルバーが効率よく解ける、小さく単純な検証条件（verification condition）を生成する。
  そのために、次の原則に従う
  - 数学的な仕様言語を、SMT ソルバーの数学的な言語に近いものに保つ（Boogie と同様）
  - メモリとエイリアスについての推論には、SMT ソルバーによる求解ではなく、軽量な線形型検査を使う
    （Cogent、Creusot、Aeneas、[linear Dafny](https://github.com/secure-foundations/dafny/tree/betr/docs/Linear) と同様）

私たちは、Rust がこれらの目標を達成するのに適した言語だと考えています。
Rust は、手動のメモリ管理を含む低レベルのデータ操作と、高度で高水準な、安全な型システムとを併せ持っています。
この型システムには、高水準の検証用言語でよく見られる機能が含まれています。
たとえば、代数的データ型（とパターンマッチ）、型クラス、第一級関数です。
そのおかげで、仕様や証明を自然な形で簡単に表現できます。
さらに重要なのは、Rust の型システムが線形型と借用（borrow）を高度に支援していることです。
これが、メモリとエイリアスについての推論の大部分を引き受けてくれます。
その結果、残りの推論ではメモリやエイリアスの問題をほとんど無視でき、Rust のコードを純粋関数型言語で書かれたコードであるかのように扱えます。
これにより、検証が容易になります。

現時点で、私たちは次のことを行うつもりはありません。
- Rust のすべての機能とライブラリに対応すること（その代わりに、利用者を支えるのに必要な、価値の高い機能とライブラリに集中します）
- 検証器そのものを検証すること
- Rust / LLVM のコンパイラを検証すること


# このガイドについて {#this-guide}

このガイドは、読者が Rust プログラミングの基本にすでにある程度なじんでいることを前提としています。
（そうでない場合は、[Learn Rust](https://www.rust-lang.org/learn) のページに 2、3 時間ほど取り組むことをおすすめします。）
Rust になじんでいると、Verus を使ううえで役に立ちます。
Verus は、仕様、証明、実行可能コードを表現するのに、Rust の構文と Rust の型システムを土台にしているからです。
実のところ、仕様と証明のための別の言語はありません。
仕様と証明は Rust の構文で書かれ、Rust の型検査器で型検査されます。
ですから、すでに Rust を知っていれば、Verus を始めるのが楽になります。

とはいえ、Rust コードの正しさを検証するには、通常の実行可能な Rust コードを書くだけでは済まない概念と技法が必要です。
たとえば Verus は、（マクロを通じて）Rust の構文を拡張し、仕様と証明を書くための新しい概念を加えています。
`forall`、`exists`、`requires`、`ensures` などです。
また、数学的な整数型である `int` や `nat` のような新しい型も導入しています。
Rust の関数がその事後条件（postcondition。`ensures` 節）を満たすことや、関数の呼び出しがその関数の事前条件（precondition。`requires` 節）を満たすことを証明するのは、難しい場合があります。
そこで、このガイドのチュートリアルでは、さまざまな概念と技法を順を追って説明します。
比較的単純な概念（整数についての基本的な証明）から始め、中程度に難しい課題（データ構造についての帰納的な証明）に進み、さらに、`forall` と `exists` を使った配列についての証明や、並行コードについての証明といった、より高度な話題へと進みます。

これらの証明はすべて、自動定理証明器の助けを借りて行われます
（具体的には [Z3](https://microsoft.github.io/z3guide/docs/logic/intro) です。
これは satisfiability-modulo-theories ソルバー、略して「SMT ソルバー」です）。
SMT ソルバーは、真偽値や整数演算についての基本的な性質のような単純な性質であれば、プログラマーの追加の助けなしに証明できることがよくあります。
しかし、より複雑な証明では、プログラマーと SMT ソルバーの両方の努力が必要になることがよくあります。
そこで、このガイドは、SMT ソルバーによる求解の強みと限界を理解する助けにもなるようにしています。
また、SMT ソルバーが自動では扱えない証明の部分をどう埋めるかについても助言します。
（たとえば、SMT ソルバーは通常、帰納法による証明を自動では行えません。
しかし、`ensures` 節が帰納法の仮定を表すような再帰的な Rust の関数を書くだけで、帰納法による証明を書けます。）
{% endraw %}

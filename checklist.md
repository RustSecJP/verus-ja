{% raw %}
# チェックリスト: 証明がうまくいかないときにすること {#checklist-what-to-do-when-proofs-go-wrong}

**証明が失敗していますが、失敗するとは思っていませんでした。何がいけないのでしょうか。**

 * `--expand-errors` を付けて Verus を実行し、何が失敗しているのかについて、より具体的な情報を得てみてください。
 * Verus の出力に、`recommends` の失敗やそのほかの注記が出ていないか確認してください。
 * `assert` 文をさらに追加してください。何が失敗しているのかについての情報が増えることもあれば、それだけで証明が直ることもあります。[このガイド](./develop_proofs.html)を参照してください。
 * 量化子を使っていますか。[トリガーの仕組み](./forall.html)を理解しているか確かめてください。
 * 非線形算術を使っていますか。[非線形算術](./nonlinear.html)のための方策のどれかを試してください。
 * ビット演算や、`as` による切り詰めを使っていますか。[bit_vector ソルバー](./bitvec.html)を試してください。
 * コンテナ型（`Seq` や `Map` など）の等しさに頼っていますか。[外延的等価性（extensional equality）](./extensional_equality.html)を試してください。
 * 再帰関数を使っていますか。[燃料（fuel）の仕組み](./recursion.html#fuel-and-reasoning-about-recursive-functions)を理解しているか確かめてください。

**検証器が「rlimit exceeded」と言っています。どうすればよいでしょうか。**

 * [量化子プロファイラ](./profiling.html)を試して、問題のあるトリガーのパターンを突き止めてください。
 * [証明を小さな部分に分けて](./breaking_proofs_into_pieces.html)みてください。
 * [`rlimit` を増やして](./reference-attributes.html#verifierrlimitn-and-verifierrlimitinfinity)みてください。証明が本当にただ大きめなだけで、Verus にもう少し労力をかけてほしい、ということもあります。

**証明が「不安定（flaky）」です。通るときもあるのに、関係のない箇所を変えると壊れます。**

 * 関数に `#[verifier::spinoff_prover]` を付けてみてください。これで少し安定することがあります。
 * [証明を小さな部分に分けて](./breaking_proofs_into_pieces.html)みてください。
{% endraw %}

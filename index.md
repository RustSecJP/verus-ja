# Verus ガイド（公式ドキュメント日本語訳）

Rust コードの形式検証ツール「Verus」の公式ガイド「Verus Tutorial and Reference」の、非公式な日本語訳です。Verus は、Rust で書かれたコードの正しさを静的に検証するためのツールです。実行時の検査を追加せず、計算機支援の定理証明を使って、実行可能な Rust コードが利用者の与えた仕様を常に満たすことを確かめます。このガイドは、仕様と証明の書き方を順を追って説明するチュートリアルと、仕様言語や属性などのリファレンスからなります。

> **ご注意**: この翻訳は Verus の開発者の確認や公認を受けたものではありません。コード、コマンド、出力例は原文のまま載せています。内容が食い違う場合は原文が優先されます。実際に使うときは、必ず最新の原文を確かめてください。

## 章の一覧 {#chapters}

原文の目次と同じ構成で並べています。原文の目次に題だけがあり、本文がまだ書かれていない項目は載せていません（下位の章をまとめる項目は、題だけを残しています）。

- [Verus の概要]({{ '/overview.html' | relative_url }})

### はじめに

- [はじめに]({{ '/getting_started.html' | relative_url }})
    - [コマンドラインで始める]({{ '/getting_started_cmd_line.html' | relative_url }})
    - [VS Code で始める]({{ '/getting_started_vscode.html' | relative_url }})

### チュートリアル: 基礎

- [verus! マクロ]({{ '/verus_macro_intro.html' | relative_url }})
- [基本的な仕様]({{ '/specs.html' | relative_url }})
    - [事前条件（requires 節）]({{ '/requires_ensures.html' | relative_url }})
    - [仕様のための式と演算子]({{ '/operators.html' | relative_url }})
    - [整数型]({{ '/integers.html' | relative_url }})
    - [等価性]({{ '/equality.html' | relative_url }})
- [仕様コード、証明コード、実行可能コード]({{ '/modes.html' | relative_url }})
    - [spec 関数]({{ '/spec_functions.html' | relative_url }})
    - [proof 関数]({{ '/proof_functions.html' | relative_url }})
    - [spec 関数と proof 関数の比較]({{ '/spec_vs_proof.html' | relative_url }})
    - [ゴーストコードと exec コード]({{ '/ghost_vs_exec.html' | relative_url }})
    - [const 宣言]({{ '/const.html' | relative_url }})
    - [すべてをまとめる]({{ '/triangle.html' | relative_url }})
- [再帰とループ]({{ '/recursion_loops.html' | relative_url }})
    - [再帰関数、decreases、燃料]({{ '/recursion.html' | relative_url }})
    - [再帰的な exec 関数と proof 関数、帰納法による証明]({{ '/induction.html' | relative_url }})
        - [軽量な停止性検査]({{ '/exec_termination.html' | relative_url }})
    - [ループと不変条件]({{ '/while.html' | relative_url }})
        - [break を伴うループ]({{ '/break.html' | relative_url }})
        - [for ループ]({{ '/for.html' | relative_url }})
        - [イテレータ]({{ '/iterators.html' | relative_url }})
    - [辞書式の decreases 節と相互再帰]({{ '/lex_mutual.html' | relative_url }})
- [データ型: 構造体と列挙型]({{ '/datatypes.html' | relative_url }})
    - [構造体]({{ '/datatypes_struct.html' | relative_url }})
    - [列挙型]({{ '/datatypes_enum.html' | relative_url }})
- [ライブラリ]({{ '/vstd.html' | relative_url }})
    - [仕様ライブラリ: Seq、Set、ISet、Map、IMap]({{ '/spec_lib.html' | relative_url }})
    - [実行可能ライブラリ: Vec]({{ '/exec_lib.html' | relative_url }})
- [spec クロージャ]({{ '/spec_closures.html' | relative_url }})
- [量化子]({{ '/quants.html' | relative_url }})
    - [forall とトリガー]({{ '/forall.html' | relative_url }})
    - [複数の変数、複数のトリガー、マッチングループ]({{ '/multitriggers.html' | relative_url }})
    - [exists と choose]({{ '/exists.html' | relative_url }})
    - [forall と exists についての証明]({{ '/quantproofs.html' | relative_url }})
    - [例: 二分探索]({{ '/binary_search.html' | relative_url }})
    - [`broadcast` で証明環境に常時有効な事実を加える]({{ '/broadcast_proof.html' | relative_url }})

### チュートリアル: 証明の開発

- [証明を作り上げる]({{ '/develop_proofs.html' | relative_url }})
    - [assert と assume を使って証明を作り上げる]({{ '/assert_assume.html' | relative_url }})
    - [ループ不変条件を考え出す]({{ '/invariants.html' | relative_url }})
    - [オーバーフローが起きないことを証明する]({{ '/overflow.html' | relative_url }})
- [SMT による求解と自動化]({{ '/smt_failures.html' | relative_url }})
    - [整数: 非線形算術]({{ '/nonlinear.html' | relative_url }})
    - [ビットベクトルとビット演算]({{ '/bitvec.html' | relative_url }})
    - [外延性による等価性]({{ '/extensional_equality.html' | relative_url }})
- [証明の性能を管理する方法と、それが重要な理由]({{ '/smt_perf_overview.html' | relative_url }})
    - [検証の性能を測る]({{ '/performance.html' | relative_url }})
    - [量化子のプロファイリング]({{ '/profiling.html' | relative_url }})
    - [モジュール、隠蔽、opaque、reveal]({{ '/opaque.html' | relative_url }})
    - [`assert(...) by { ... }` で局所的な証明を隠す]({{ '/assert_by.html' | relative_url }})
    - [計算による構造化された証明]({{ '/calc.html' | relative_url }})
    - [計算による証明]({{ '/assert_by_compute.html' | relative_url }})
    - [SMT ソルバーのインスタンスを別に起動する]({{ '/spinoff_prover.html' | relative_url }})
    - [証明を小さな部分に分ける]({{ '/breaking_proofs_into_pieces.html' | relative_url }})
- [LLM を使って Verus プロジェクトの開発を助ける]({{ '/llms.html' | relative_url }})
    - [LLM を使って Verus の証明を書く作業を助ける]({{ '/llmforverusproof.html' | relative_url }})
- [チェックリスト: 証明がうまくいかないときにすること]({{ '/checklist.html' | relative_url }})

### チュートリアル: 検証と Rust

- [変更、参照、借用]({{ '/mutation-references-borrowing.html' | relative_url }})
    - [可変参照]({{ '/mutable-references.html' | relative_url }})
    - [可変参照についてのアサーション]({{ '/assert-mut-ref.html' | relative_url }})
- [トレイト]({{ '/traits.html' | relative_url }})
- [独自の型のためのイテレータの仕様]({{ '/iterator-specs.html' | relative_url }})
    - [有限イテレータのイテレータ仕様を実装する]({{ '/iterator-specs-finite.html' | relative_url }})
    - [無限イテレータのイテレータ仕様を実装する]({{ '/iterator-specs-infinite.html' | relative_url }})
- [高階の実行可能関数]({{ '/higher-order-fns.html' | relative_url }})
    - [関数を値として渡す]({{ '/exec_funs_as_values.html' | relative_url }})
    - [クロージャ]({{ '/exec_closures.html' | relative_url }})
- [文字列]({{ '/strings.html' | relative_url }})
- [unsafe なコードと複雑な所有権]({{ '/complex_ownership.html' | relative_url }})
    - [内部可変性]({{ '/interior_mutability.html' | relative_url }})
    - [ポインタとセル]({{ '/pointers.html' | relative_url }})
    - [並行性]({{ '/concurrency.html' | relative_url }})
- [論理的アトミック性]({{ '/logatom.html' | relative_url }})
    - [アトミックな仕様]({{ '/logatom-spec.html' | relative_url }})
    - [線形化点を発動する]({{ '/logatom-open.html' | relative_url }})
    - [論理的にアトミックな関数を呼び出す]({{ '/logatom-call.html' | relative_url }})
- [コンテナライブラリを検証する]({{ '/container_bst.html' | relative_url }})
    - [単純な二分探索木]({{ '/container_bst_first_draft.html' | relative_url }})
    - [型不変条件で整形式性をカプセル化する]({{ '/container_bst_type_invariant.html' | relative_url }})
    - [ジェネリックにする]({{ '/container_bst_generic.html' | relative_url }})
    - [Clone の実装]({{ '/container_bst_clone.html' | relative_url }})
    - [コンテナの中の可変参照]({{ '/container_bst_mut_refs.html' | relative_url }})
    - [例のソースコード全文]({{ '/container_bst_all_source.html' | relative_url }})
- [未検証コードとのやり取り]({{ '/interacting-with-unverified-code.html' | relative_url }})
    - [検証済みコードから未検証コードを呼び出す]({{ '/calling-unverified-from-verified.html' | relative_url }})
    - [未検証コードから検証済みコードを呼び出す]({{ '/calling-verified-from-unverified.html' | relative_url }})
- [検証済みプログラムの保証を理解する]({{ '/guarantees.html' | relative_url }})
    - [Verus が保証すること]({{ '/verus-guarantees.html' | relative_url }})
    - [仮定と、信頼される構成要素]({{ '/tcb.html' | relative_url }})
    - [メモリ安全性は検証を条件とする]({{ '/memory-safety.html' | relative_url }})
    - [未検証コードから検証済みコードを呼び出す]({{ '/call-from-unverified-code.html' | relative_url }})

### インストール、設定、ツール

- インストールとセットアップ
    - [Verus の IDE 対応]({{ '/ide_support.html' | relative_url }})
    - [Singular のインストールと設定]({{ '/install-singular.html' | relative_url }})
- [プロジェクトのセットアップと開発]({{ '/projects.html' | relative_url }})
    - [Cargo 経由で Verus を使う]({{ '/cargo_verus.html' | relative_url }})
    - [Rustdoc による文書化]({{ '/verusdoc.html' | relative_url }})
    - [ゴースト消去]({{ '/erasure.html' | relative_url }})

### リファレンス

- [対応している Rust の機能]({{ '/features.html' | relative_url }})
- [Verus の構文]({{ '/syntax.html' | relative_url }})
- モード
    - [変数のモード]({{ '/reference-var-modes.html' | relative_url }})
- [コミュニティ提供の拡張機能]({{ '/contributed.html' | relative_url }})
    - [`exec_spec_verified!` マクロと `exec_spec_unverified!` マクロで spec 関数から実行可能コードを作る]({{ '/exec_spec.html' | relative_url }})
    - [実行可能コードのための Verus の属性]({{ '/exec_attr.html' | relative_url }})
        - [実行可能コードに対する spec 関数を自動的に作る]({{ '/exec_to_spec.html' | relative_url }})
- [仕様言語]({{ '/reference-specification-language.html' | relative_url }})
    - [型の数学的な解釈]({{ '/reference-types.html' | relative_url }})
    - [spec 式]({{ '/spec-expressions.html' | relative_url }})
        - [演算子の優先順位]({{ '/spec-operator-precedence.html' | relative_url }})
        - [Rust のサブセット]({{ '/spec-rust-subset.html' | relative_url }})
        - [spec コードにおける算術]({{ '/spec-arithmetic.html' | relative_url }})
        - [ビット演算子]({{ '/spec-bit-ops.html' | relative_url }})
        - [`as` による型強制]({{ '/reference-as.html' | relative_url }})
        - [spec 等価性（`==`）]({{ '/spec-equality.html' | relative_url }})
        - [外延的等価性（`=~=` と `=~~=`）]({{ '/ref-extensional-equality.html' | relative_url }})
        - [前置の and/or（`&&&` と `|||`）]({{ '/prefix-and-or.html' | relative_url }})
        - [連鎖演算子]({{ '/reference-chained-op.html' | relative_url }})
        - [含意（`==>`、`<==`、`<==>`）]({{ '/reference-implication.html' | relative_url }})
        - [spec の量化子（`forall`、`exists`）]({{ '/spec-quantifiers.html' | relative_url }})
        - [〜を満たすもの（`choose`）]({{ '/spec-choose.html' | relative_url }})
        - [トリガー注釈]({{ '/trigger-annotations.html' | relative_url }})
        - [ビュー関数 `@`]({{ '/reference-at-sign.html' | relative_url }})
        - [spec のインデックス演算子 `[]`]({{ '/reference-spec-index.html' | relative_url }})
        - [`has` 演算子]({{ '/reference-has.html' | relative_url }})
        - [`is` 演算子]({{ '/reference-is.html' | relative_url }})
        - [`matches` 演算子]({{ '/reference-matches.html' | relative_url }})
        - [`decreases_to!`]({{ '/reference-decreases-to.html' | relative_url }})
- 証明
    - 証明の文
        - [`assert`]({{ '/reference-assert.html' | relative_url }})
        - [`assume`]({{ '/reference-assume.html' | relative_url }})
        - [assert ... by]({{ '/reference-assert-by.html' | relative_url }})
        - [assert forall ... by]({{ '/reference-assert-forall-by.html' | relative_url }})
        - [assert ... by(...)]({{ '/reference-assert-by-prover.html' | relative_url }})
        - [`reveal`、`reveal_with_fuel`、`hide`]({{ '/reference-reveal-hide.html' | relative_url }})
        - [`reveal_strlit`]({{ '/reference-reveal-strlit.html' | relative_url }})
    - 証明器モード
        - [`bit_vector` 証明器モード]({{ '/reference-prover-mode-bit-vector.html' | relative_url }})
        - [非線形ソルバー]({{ '/reference-prover-mode-nonlinear.html' | relative_url }})
        - [`integer_ring` ソルバー]({{ '/reference-prover-mode-integer-ring.html' | relative_url }})
        - [compute モード]({{ '/reference-prover-mode-compute.html' | relative_url }})
- 関数の仕様
    - 関数のシグネチャ
        - [exec 関数のシグネチャ]({{ '/reference-exec-signature.html' | relative_url }})
        - [proof 関数のシグネチャ]({{ '/reference-proof-signature.html' | relative_url }})
        - [spec 関数のシグネチャ]({{ '/reference-spec-signature.html' | relative_url }})
    - シグネチャの節
        - [returns]({{ '/reference-returns.html' | relative_url }})
        - [opens_invariants]({{ '/reference-opens-invariants.html' | relative_url }})
        - [巻き戻しのシグネチャ]({{ '/reference-unwind-sig.html' | relative_url }})
        - [recommends]({{ '/reference-recommends.html' | relative_url }})
    - [シグネチャの継承]({{ '/reference-signature-inheritance.html' | relative_url }})
    - [FnOnce の仕様]({{ '/reference-signature-fnonce.html' | relative_url }})
- [外部トレイトの仕様]({{ '/external_trait_specifications.html' | relative_url }})
- 再帰と停止性
    - [decreases ... when ... via ...]({{ '/reference-decreases.html' | relative_url }})
- [型不変条件]({{ '/reference-type-invariants.html' | relative_url }})
- [属性]({{ '/reference-attributes.html' | relative_url }})
- 指示
    - [assume_specification]({{ '/reference-assume-specification.html' | relative_url }})
    - [「global」指示]({{ '/reference-global.html' | relative_url }})
- そのほかの Rust の機能
    - [static 項目]({{ '/static.html' | relative_url }})
    - [共用体]({{ '/reference-unions.html' | relative_url }})
    - [ポインタとセル]({{ '/reference-pointers-cells.html' | relative_url }})
- コマンドライン
    - [record フラグ]({{ '/reference-flag-record.html' | relative_url }})

### そのほか

原文の目次には載っていないページです。

- [基本的なライブラリ]({{ '/pervasive.html' | relative_url }})

## 出典

- 原文: [Verus Tutorial and Reference](https://verus-lang.github.io/verus/guide/)（ソースは [verus-lang/verus](https://github.com/verus-lang/verus) の `source/docs/guide`）
- 原著者: Verus の貢献者
- ライセンス: MIT
- 訳した版: 2026-11-22 時点（コミット `1ce6a59`）

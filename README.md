# Verus ガイド（公式ドキュメント日本語訳）

> **これは Rust コードの形式検証ツール [Verus](https://github.com/verus-lang/verus) の公式ガイド [Verus Tutorial and Reference](https://verus-lang.github.io/verus/guide/) の、非公式な日本語訳です。** Verus の開発者による確認や公認を受けたものではありません。このリポジトリにツール本体のコードはありません。
>
> **読むのはこちら: https://rustsecjp.github.io/verus-ja/**
>
> This is an unofficial Japanese translation of the "Verus Tutorial and Reference". It does not contain the tool itself.

Verus は、Rust で書かれたコードの正しさを検証するためのツールです。検証は静的に行われ、実行時の検査は一切追加されません。計算機支援の定理証明を使い、実行可能な Rust コードが、利用者の与えた仕様を、あり得るすべての実行において常に満たすことを確かめます。

ガイドは、仕様と証明の書き方を順を追って説明するチュートリアルと、仕様言語・証明の文・属性などのリファレンスからなります。全 160 章を訳しています。

原文のガイドには独立した README が無いので、この README は日本語訳の案内だけを載せています。

## 訳した版

| | |
|---|---|
| 原文の版 | 2026-11-22 時点（コミット [`1ce6a59`](https://github.com/verus-lang/verus/commit/1ce6a59bd72713cf486a7660097fe5ac57519617)） |
| 翻訳日 | 2026-11-22 |

原文がこの版より後に更新されている場合、訳は古い内容のままです。

## 出典とライセンス

- 原文: https://verus-lang.github.io/verus/guide/ （ソース: [verus-lang/verus](https://github.com/verus-lang/verus) の `source/docs/guide`）
- 原著者: Verus の貢献者（The Verus Contributors）
- ライセンス: MIT。全文は [LICENSE-MIT.txt](LICENSE-MIT.txt)。
- 訳した版: 2026-11-22 時点（コミット 1ce6a59bd72713cf486a7660097fe5ac57519617）
- 変更点: 日本語への翻訳。取り込み記法で参照されているコード例の展開。表示用の書式の調整。詳しくは [NOTICE.md](NOTICE.md) にあります。

誤訳の指摘や訳語の提案は、Issue か Pull Request でお願いします。

RustSecJP は非公式のコミュニティです。Verus の開発者、RustSec、Rust Project、Rust Foundation とは提携していません。

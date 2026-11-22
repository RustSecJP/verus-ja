{% raw %}
# 対応している Rust の機能 {#supported-rust-features}

対応している Rust の機能の早見表です。この一覧は _Verus の_ 機能をすべて含んでいるわけではないことに注意してください。Verus には、標準の Rust に対応するものが無い仕様用・証明用の機能が数多くありますが、この一覧が扱うのは Rust の機能だけです。実行可能な Rust コード、仕様コード、証明コードという Verus の区別について詳しくは、[ガイド](./modes.html)を参照してください。

Verus は活発に開発されていることに注意してください。ある機能が未対応である場合、それは本当に難しいのかもしれませんし、単に優先度が低いだけかもしれません。予定されている機能についての情報は、[GitHub の issue](https://github.com/verus-lang/verus/issues) または [discussions](https://github.com/verus-lang/verus/discussions) を参照してください。

**最終更新: 2026-05-13**

<div class="table-wrapper"><table>
  <thead><tr><th colspan="2"><strong>アイテム</strong></th></tr></thead>
  <tbody>
  <tr>
    <td>関数、メソッド、関連関数</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>関連定数</td>
    <td>一部対応</td>
  </tr>
  <tr>
    <td>構造体</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>列挙型</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>const 関数</td>
    <td>一部対応</td>
  </tr>
  <tr>
    <td>async 関数</td>
    <td>未対応</td>
  </tr>
  <tr>
    <td>マクロ</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>型エイリアス</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>const アイテム</td>
    <td><a href="const.html">一部対応</a></td>
  </tr>
  <tr>
    <td>static アイテム</td>
    <td><a href="static.html">一部対応</a></td>
  </tr>
  </tbody>
  <thead><tr><th colspan="2"><strong>構造体・列挙型の定義</strong></th></tr></thead>
  <tbody>
  <tr>
    <td>型パラメータ</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>where 節</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>ライフタイムパラメータ</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>const ジェネリクス</td>
    <td>一部対応</td>
  </tr>
  <tr>
    <td>独自の判別値</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>公開・非公開のフィールド</td>
    <td>一部対応</td>
  </tr>
  </tbody>
  <thead><tr><th colspan="2"><strong>式と文</strong></th></tr></thead>
  <tbody>
  <tr>
    <td>変数、代入、mut 変数</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>if、else</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>パターン、match、if-let、match ガード</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>ブロック式</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>入れ子のアイテム</td>
    <td>一部対応</td>
  </tr>
  <tr>
    <td><code>loop</code>、<code>while</code></td>
    <td><a href="while.html">対応</a></td>
  </tr>
  <tr>
    <td><code>for</code></td>
    <td>一部対応</td>
  </tr>
  <tr>
    <td><code>?</code></td>
    <td>対応</td>
  </tr>
  <tr>
    <td>async ブロック</td>
    <td>未対応</td>
  </tr>
  <tr>
    <td>await</td>
    <td>未対応</td>
  </tr>
  <tr>
    <td>unsafe ブロック</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>共有借用（<code>&</code>）</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>可変借用（<code>&mut</code>）</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>場所式</td>
    <td>対応</td>
  </tr>
  <tr>
    <td><a href="https://doc.rust-lang.org/reference/expressions/operator-expr.html?highlight=assignment#destructuring-assignments">分配代入</a></td>
    <td>未対応</td>
  </tr>
  <tr>
    <td><code>==</code>、<code>!=</code></td>
    <td>対応</td>
  </tr>
  <tr>
    <td>型キャスト（<code>as</code>）</td>
    <td>一部対応</td>
  </tr>
  <tr>
    <td>複合代入（<code>+=</code> など）</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>配列式</td>
    <td>一部対応（<code>const</code> 引数を使う埋め尽くし式には未対応）</td>
  </tr>
  <tr>
    <td>範囲式</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>添字式</td>
    <td>一部対応</td>
  </tr>
  <tr>
    <td>タプル式</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>構造体・列挙型のコンストラクタ</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>フィールドアクセス</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>関数とメソッドの呼び出し</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>クロージャ</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>ラベル、break、continue</td>
    <td><a href="break.html">対応</a></td>
  </tr>
  <tr>
    <td>return 文</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>暗黙の型強制、参照外し、借用</td>
    <td>対応</td>
  </tr>
  </tbody>
  <thead><tr><th colspan="2"><strong>整数演算</strong></th></tr></thead>
  <tbody>
  <tr>
    <td>符号なし整数の算術演算</td>
    <td><a href="integers.html">対応</a></td>
  </tr>
  <tr>
    <td>符号付き整数の算術演算</td>
    <td><a href="integers.html">対応</a></td>
  </tr>
  <tr>
    <td>ビット演算（<code>&</code>、<code>|</code>、<code>!</code>、<code>&gt;&gt;</code>、<code>&lt;&lt;</code>）</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>アーキテクチャ依存の型（<code>usize</code>、<code>isize</code>）</td>
    <td>対応</td>
  </tr>
  </tbody>
  <thead><tr><th colspan="2"><strong>型と標準ライブラリの機能</strong></th></tr></thead>
  <tbody>
  <tr>
    <td>整数型</td>
    <td>対応</td>
  </tr>
  <tr>
    <td><code>bool</code></td>
    <td>対応</td>
  </tr>
  <tr>
    <td>文字列</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>Vec</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>Option / Result</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>浮動小数点数</td>
    <td>一部対応</td>
  </tr>
  <tr>
    <td>スライス</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>配列</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>ポインタ</td>
    <td>一部対応</td>
  </tr>
  <tr>
    <td>共有参照（<code>&T</code>）</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>可変参照（<code>&mut T</code>）</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>never 型</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>関数ポインタ型</td>
    <td>未対応</td>
  </tr>
  <tr>
    <td>クロージャ型</td>
    <td>一部対応（可変キャプチャには未対応）</td>
  </tr>
  <tr>
    <td>トレイトオブジェクト（dyn）</td>
    <td>一部対応</td>
  </tr>
  <tr>
    <td>impl 型</td>
    <td>一部対応</td>
  </tr>
  <tr>
    <td>Cell、RefCell</td>
    <td><a href="https://verus-lang.github.io/verus/verusdoc/vstd/cell/index.html">vstd の代替機能</a></td>
  </tr>
  <tr>
    <td>イテレータ</td>
    <td>一部対応</td>
  </tr>
  <tr>
    <td><code>Vec</code>、<code>HashMap</code>、<code>HashSet</code>、<code>VecDeque</code></td>
    <td>対応</td>
  </tr>
  <tr>
    <td>スマートポインタ（<code>Box</code>、<code>Rc</code>、<code>Arc</code>）</td>
    <td>対応</td>
  </tr>
  <tr>
    <td><code>Pin</code></td>
    <td>未対応</td>
  </tr>
  <tr>
    <td>ハードウェア組み込み関数</td>
    <td>未対応</td>
  </tr>
  <tr>
    <td>出力、入出力</td>
    <td>未対応</td>
  </tr>
  <tr>
    <td>パニックの巻き戻し</td>
    <td>一部対応</td>
  </tr>
  </tbody>
  <thead><tr><th colspan="2"><strong>トレイト</strong></th></tr></thead>
  <tbody>
  <tr>
    <td>利用者定義のトレイト</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>既定の実装</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>トレイト宣言のトレイト境界</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>型引数を持つトレイト</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>関連型</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>ジェネリック関連型</td>
    <td>一部対応（ライフタイムだけに対応）</td>
  </tr>
  <tr>
    <td>高階トレイト境界</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>マーカートレイト（<code>Copy</code>、<code>Send</code>、<code>Sync</code>）</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>標準的なトレイト（<code>Clone</code>、<code>Default</code>、<code>Step</code>、<code>From</code>、<code>TryFrom</code>、<code>Into</code>、<code>PartialEq</code>、<code>Eq</code>、<code>PartialOrd</code>、<code>Ord</code>、<code>Neg</code>、<code>Not</code>、<code>Add</code>、<code>Sub</code>、<code>Mul</code>、<code>Div</code>、<code>Rem</code>、<code>BitAnd</code>、<code>BitOr</code>、<code>BitXor</code>、<code>Shl</code>、<code>Shr</code>）</td>
    <td>一部対応</td>
  </tr>
  <tr>
    <td>標準的なトレイト（<code>Debug</code>、<code>serde::Serialize</code>）</td>
    <td>未対応</td>
  </tr>
  <tr>
    <td>利用者定義のデストラクタ（<code>Drop</code>）</td>
    <td>未対応</td>
  </tr>
  <tr>
    <td><code>Sized</code>（<code>size_of</code>、<code>align_of</code>）</td>
    <td>対応</td>
  </tr>
  <tr>
    <td><code>Deref</code></td>
    <td>対応</td>
  </tr>
  <tr>
    <td><code>DerefMut</code></td>
    <td>対応</td>
  </tr>
  </tbody>
  <thead><tr><th colspan="2"><strong>マルチスレッド</strong></th></tr></thead>
  <tbody>
  <tr>
    <td><code>Mutex</code>、<code>RwLock</code>（標準ライブラリのもの）</td>
    <td>未対応 </td>
  </tr>
  <tr>
    <td>検証済みのロックの実装</td>
    <td>対応 </td>
  </tr>
  <tr>
    <td>アトミック型</td>
    <td><a href="https://verus-lang.github.io/verus/verusdoc/vstd/atomic_ghost/index.html">vstd の代替機能</a></td>
  </tr>
  <tr>
    <td>spawn と join</td>
    <td><a href="https://verus-lang.github.io/verus/verusdoc/vstd/thread/index.html">対応</a></td>
  </tr>
  <tr>
    <td>内部可変性</td>
    <td><a href="interior_mutability.html">対応</a></td>
  </tr>
  </tbody>
  <thead><tr><th colspan="2"><strong>unsafe</strong></th></tr></thead>
  <tbody>
  <tr>
    <td>生ポインタ</td>
    <td><a href="https://verus-lang.github.io/verus/verusdoc/vstd/raw_ptr/index.html">一部対応</a></td>
  </tr>
  <tr>
    <td>transmute</td>
    <td>未対応</td>
  </tr>
  <tr>
    <td>共用体</td>
    <td><a href="reference-unions.html">対応</a></td>
  </tr>
  <tr>
    <td><code>UnsafeCell</code></td>
    <td><a href="https://verus-lang.github.io/verus/verusdoc/vstd/cell/struct.PCell.html">vstd の代替機能</a></td>
  </tr>
  </tbody>
  <thead><tr><th colspan="2"><strong>クレートとコードの構成</strong></th></tr></thead>
  <tbody>
  <tr>
    <td>複数クレートのプロジェクト</td>
    <td>一部対応</td>
  </tr>
  <tr>
    <td>検証済みクレート + 未検証クレート</td>
    <td>一部対応</td>
  </tr>
  <tr>
    <td>モジュール</td>
    <td>対応</td>
  </tr>
  <tr>
    <td>rustdoc</td>
    <td>対応</td>
  </tr>
  </tbody>
</table></div>
{% endraw %}

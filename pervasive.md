{% raw %}
# 基本的なライブラリ {#basic-libraries}

この章では、列（sequence）、集合（set）、マップ（map）、ベクタ（vector）のための、Verus の組み込みライブラリの一部を紹介します。
これらのライブラリは、現在、[vstd](https://github.com/verus-lang/verus/tree/main/source/vstd/) ディレクトリにある
次のモジュール群に置かれています。
- [vstd::seq](https://github.com/verus-lang/verus/tree/main/source/vstd/seq.rs)
- [vstd::seq_lib](https://github.com/verus-lang/verus/tree/main/source/vstd/seq_lib.rs)
- [vstd::set](https://github.com/verus-lang/verus/tree/main/source/vstd/set.rs)
- [vstd::set_lib](https://github.com/verus-lang/verus/tree/main/source/vstd/set_lib.rs)
- [vstd::map](https://github.com/verus-lang/verus/tree/main/source/vstd/map.rs)
- [vstd::vec](https://github.com/verus-lang/verus/tree/main/source/vstd/vec.rs)

詳しくは、[API ドキュメント](https://verus-lang.github.io/verus/verusdoc/vstd/index.html)を参照してください。

例として、[次のコード](https://github.com/verus-lang/verus/tree/main/examples/guide/vstd_example.rs)は
`vstd::seq` モジュールを使っています。

```rust
#[allow(unused_imports)]
use verus_builtin::*;
#[allow(unused_imports)]
use verus_builtin_macros::*;
use vstd::seq::*;

verus! {

fn main() {
    proof {
        let s: Seq<int> = seq![0, 10, 20, 30, 40];
        assert(s.len() == 5);
        assert(s[2] == 20);
        assert(s[3] == 30);
    }
}

} // verus!
```

便利なように、`vstd::prelude` モジュールには、`verus_builtin_macros` と `verus_builtin` に加えて、
`seq`、`set`、`map`、`option`、`result`、`string` の型が含まれています。

```rust
use vstd::prelude::*;

verus! {

fn main() {
    proof {
        let s: Seq<int> = seq![0, 10, 20, 30, 40];
        assert(s.len() == 5);
        assert(s[2] == 20);
        assert(s[3] == 30);
    }
}

} // verus!
```

# 糖衣構文: spec_index と view {#some-syntactic-sugar-spec_index-and-view}

ゴーストコードでは、Verus は、ライブラリで使われている 2 つの糖衣構文（syntactic sugar）に対応しています。
- `[...]` は、`.spec_index(...)` の省略形です。
  これは、たとえば、列から要素を取り出すのに使われます。`s[3]` は `s.spec_index(3)` を意味します。
- `@` は、`.view()` の省略形です。
  これは、具体的な `exec` オブジェクトの抽象的な中身を表す `spec` の値を得るのによく使われます。
  たとえば、`Vec<T>` 型の `exec` のベクタ `v` について、値 `v@` は、
  `v` の要素を含む `Seq<T>` 型の `spec` の列です。
{% endraw %}

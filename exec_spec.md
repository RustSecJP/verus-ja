{% raw %}
# `exec_spec_verified!` マクロと `exec_spec_unverified!` マクロで spec 関数から実行可能コードを作る {#producing-executable-code-from-spec-functions-with-the-exec_spec_verified-and-exec_spec_unverified-macros}

Verus で証明を書いていると、単純な関数やデータ構造を exec モードと spec モードの両方で実装し、そのうえで両者が等価であることを示さなければならない場面がときどきあります。
単純な関数の場合、この作業は面倒になりがちです。

`exec_spec_verified!` マクロは、この作業を簡単にします。
必要な関数・構造体・列挙型を、`exec_spec_verified!` が対応している範囲の spec モードで書くだけで済みます。
するとマクロが、これらの spec の項目に対応する exec 版と、等価性の証明を自動的に生成します。
この証明は、停止性、算術オーバーフローが起きないこと、事前条件の違反が起きないことも保証します。

`exec_spec_unverified!` マクロも spec コードを exec コードへ自動的にコンパイルしますが、等価性の証明は付きません。
つまり、`exec_spec_unverified!` が生成する仕様付きの実行可能コードには、すべて `#[verifier::external_body]` が付きます。そのため、spec コードと exec コードの等価性は Verus によって証明されません。
`exec_spec_unverified!` マクロは、現時点では Verus のより広い範囲に対応しています。
コンパイルされたコードを検証済みのプロジェクトから呼び出さない場面（たとえば、入力例を使って仕様をテストする場合）に最適です。
証明の生成を省くことで、コンパイルされた exec コードで起こりうる証明のデバッグを避けられます。

## `exec_spec_verified!` マクロと `exec_spec_unverified!` マクロにおける `vstd` 関数の未検証の変換 {#unverified-translations-of-vstd-functions-within-the-exec_spec_verified-and-exec_spec_unverified-macros}

表現力を高めるために、`exec_spec_verified!` マクロと `exec_spec_unverified!` マクロは、`vstd` の型（`Seq` など）に対するいくつかの spec 関数をマクロの中で使えるようにしています。
つまり、対応している spec 関数の exec への変換は、マクロがあらかじめ用意しています。
これらの変換は、そうした関数を呼び出す spec コードを exec コードへコンパイルするときに使われます。
それぞれのマクロが対応している関数の完全な一覧は、後述します。

ただし、多くの `vstd` 関数については、spec コードから対応する exec コードへの変換が未検証です。
該当するものには、以下で \* をはっきり付けています。
Verus が対応する Rust の言語機能が増えるにつれて、未検証の変換の数を減らしていきたいと考えています。

## `exec_spec_verified!` マクロ {#the-exec_spec_verified-macro}

例を示します。
```rust
use vstd::contrib::exec_spec::*;
use vstd::prelude::*;

verus! {

exec_spec_verified! {

struct Point {
    x: i64,
    y: i64,
}

spec fn on_line(points: Seq<Point>) -> bool {
    forall |i: usize| #![auto] 0 <= i < points.len()
        ==> points[i as int].y == points[i as int].x
}

}

} // verus!
```
この例では、与えられた `Point` の列がすべて同じ座標を持つかどうかを調べる、単純な spec 関数 `on_line` を定義しています。

この例の `exec_spec_verified!` マクロ呼び出しは、スコープ内のすべての spec の項目を受け取り、おおむね次の定義のような実行可能な対応物を導出します。
```
struct ExecPoint {
    x: i64,
    y: i64,
}

impl DeepView for ExecPoint {
    type V = Point;
    ...
}

fn exec_on_line(points: &[ExecPoint]) -> (res: bool)
    ensures res == on_line(points.deep_view())
{ ... }
```

マクロを呼び出したあとには、元の spec の項目（`Point` と `on_line`）がこれまでどおり存在し、それに加えて、適切な等価性が検証された新しい項目 `ExecPoint` と `exec_on_line` も得られます。
この等価性は、次のような簡単な確認でテストできます。
```rust
verus! {

fn main() {
    let p1 = ExecPoint { x: 1, y: 1 };
    let p2 = ExecPoint { x: 2, y: 2 };
    let points = vec![p1, p2];
    assert(on_line(points.deep_view()));
    let b = exec_on_line(points.as_slice());
    assert(b);
}

} // verus!
```

現在、`exec_spec_verified!` は次の基本的な機能に対応しています。
  - 基本的な算術演算
  - 論理演算子（`&&`、`||`、`&&&`、`|||`、`!`、`==>`）
  - if、match、「matches」
  - フィールド式
  - spec 関数の呼び出しと再帰
  - `forall |i: <type>| <lower> <op> i <op> <upper> ==> <expr>` および `exists |i: <type>| <lower> <op> i <op> <upper> && <expr>` という形の有界な量化子（quantifier）。ここで、
    - `<op>` は `<=` か `<` のどちらかです
    - `<type>` は Rust のプリミティブ整数型（`i<N>`、`isize`、`u<N>`、`usize`）です
  - `SpecString`（`Seq<char>` の別名で、`String`/`&str` がほしいことを構文上で示すためのもの）、等価性\*、インデックス、len、文字列リテラル
  - `Option<T>` と、次の関数
    - 等価性、`unwrap`
  - `Seq<T>`（文脈に応じて `Vec<T>` か `&[T]` にコンパイルされます）、`seq!` リテラル、および次の `Seq` の関数
    - 等価性\*、`len`、インデックス、`subrange`\*、`add`\*、`push`\*、`update`\*、`empty`、`to_multiset`\*、`drop_first`\*、`drop_last`\*、`take`\*、`skip`\*、`first`、`last`、`is_suffix_of`\*、`is_prefix_of`\*、`contains`\*、`index_of`\*、`index_of_first`\*、`index_of_last`\*
  - `Map<K, V>`（`HashMap<K, V>` にコンパイルされます）と、次の `Map` の関数
    - 等価性\*、`len`\*、インデックス\*、`empty`、`dom`\*、`insert`\*、`remove`\*、`get`\*
    - 注意: インデックスは、`K` がプリミティブ型（たとえば `usize`）である `Map<K, V>` でのみ対応しています。それ以外の型 `K` では、代わりに `get` を使ってください。
  - `Set<T>`（`HashSet<T>` にコンパイルされます）と、次の `Set` の関数
    - 等価性\*、`len`\*、`empty`、`contains`\*、`insert`\*、`remove`\*、`union`\*、`intersect`\*、`difference`\*
  - `Multiset<T>`（`ExecMultiset<T>` にコンパイルされます。これは `vstd::contrib::exec_spec` で実装されている型で、内部表現は `HashMap` です）と、次の `Multiset` の関数
    - 等価性\*、`len`\*、`count`\*、`empty`\*、`singleton`\*、`add`\*、`sub`\*
  - ユーザー定義の構造体と列挙型。これらの型は、フィールドに spec と互換性のある型（たとえば `Seq`）を使って、マクロの中で定義する必要があります。そうした型は `Exec-` 版にコンパイルされ、そこでは各フィールドの型の exec 版（たとえば `Vec` やスライス）が使われます。
  - プリミティブな整数型・真偽値型（`i<N>`、`isize`、`u<N>`、`usize`、`char` など）。`int`、`nat`、`real` は、`exec_spec_verified!` でも `exec_spec_unverified!` でも使えないことに注意してください。
  - `is` 式と `isnt` 式。`Option`/`Result` に対しても、ユーザー定義の列挙型に対しても使えます。
  - 参照外し式（`*x`）。
  - ユーザー定義の構造体と列挙型に spec メソッドを定義する `impl` ブロック。
    - 対応しているのは `&self` メソッドだけです。
    - 各 spec メソッド `T::method` は exec メソッド `ExecT::exec_method` にコンパイルされ、マクロ内の `x.method(...)` のようなメソッド呼び出しは `x.exec_method(...)` にコンパイルされます。
    - ジェネリックな実装とトレイトの実装には対応していません。
    - `Self` は、コンストラクターとして（たとえば `Self(self.0)`）使うことも、ほかのメソッドを呼び出すために（たとえば `Self::get(self)`）使うこともできます。
  - `MemContents<T>`（`ExecMemContents<T>` にコンパイルされます。これは `vstd::contrib::exec_spec` で実装されている、exec 側の対応型です）。`Init`/`Uninit` コンストラクター、パターンマッチ、`is`/`isnt`、および次の関数に対応しています。
    - 等価性、`is_init`、`is_uninit`、`value`
  - `Result<T, E>`。`Ok`/`Err` コンストラクター、パターンマッチ、等価性に対応しています。
  - match ガード（たとえば `Some(v) if v > 3 => ...`）。
    - 列挙型のフィールドに対する矢印式。`x->Variant_field`（たとえば `e->B_q`、`o->Some_0`、`r->Ok_0`）と、フィールド名が 1 つのバリアントにしか現れない場合の `x->field`（たとえば `e->z`、`o->0`）です。これらは、ユーザー定義の列挙型、`Option`、`Result`、`MemContents` で使えます。`exec_spec_verified!` では、使う場所でバリアントが確定していなければなりません（たとえば `recommends e is A` によって）。
  - `is_variant(x, "Variant")`。`x is Variant` と同じようにコンパイルされます。
  - spec 等価性と外延的等価性（`===`、`=~=`、`=~~=`）およびその否定（`!==`、`!~=`、`!~~=`）。`==`/`!=` と同じようにコンパイルされます。
  - 64 ビットまでのプリミティブ整数に対する固定幅の算術関数 `add`、`sub`、`mul`。`+`、`-`、`*` とは違って、結果はオペランドの型のままで、オーバーフロー時にはラップアラウンドします。

\* の付いた関数は、spec コードから exec コードへの変換が未検証です。

## `exec_spec_unverified!` マクロ {#the-exec_spec_unverified-macro}

上と同じ例を、`exec_spec_unverified!` マクロで書くと次のようになります。
```rust
use vstd::contrib::exec_spec::*;
use vstd::prelude::*;

verus! {

exec_spec_unverified! {

struct Point {
    x: i64,
    y: i64,
}

spec fn on_line(points: Seq<Point>) -> bool {
    forall |i: usize| #![auto] 0 <= i < points.len()
        ==> points[i as int].y == points[i as int].x
}

}

} // verus!
```

`exec_spec_verified!` マクロの場合と同じ exec の項目が生成されますが、実行可能な関数に付く仕様は Verus によって証明されません。
```
#[verifier::external_body]
fn exec_on_line(points: &[ExecPoint]) -> (res: bool)
    ensures res == on_line(points.deep_view())
{ ... }
```

実行可能な関数 `exec_on_line` を実行すると、コンパイルされた `on_line` の仕様が特定の入力に対して返す結果を、次のようにして簡単に確認できます。
```rust
fn main() {
    let p1 = ExecPoint { x: 1, y: 1 };
    let p2 = ExecPoint { x: 2, y: 2 };
    let points = vec![p1, p2];
    let b = exec_on_line(points.as_slice());
    assert_eq!(b, true);
}
```

`exec_spec_unverified!` マクロは、`exec_spec_verified!` と同じ Verus の機能すべてに加えて、次の機能にも対応しています。
  - より一般的な有界量化子。量化子の式は、`forall |x1: <type1>, x2: <type2>, ..., xN: <typeN>| <guard1> && <guard2> && ... && <guardN> ==> <body>` または `exists |x1: <type1>, x2: <type2>, ..., xN: <typeN>| <guard1> && <guard2> && ... && <guardN> && <body>` という形に一致しなければなりません。ここで、
    - `<guardI>` は `<lowerI> <op> xI <op> <upperI>` という形です。ここで、
        - `<op>` は `<=` か `<` のどちらかです
        - `<lowerI>` と `<upperI>` では、`J < I` であるすべての `xJ` に言及できます
    - `<typeI>` は Rust のプリミティブ整数型（`i<N>`、`isize`、`u<N>`、`usize`）または `char` です。`exec_spec_verified!` の量化子では `char` に対応していないことに注意してください。

## `exec_spec_type!` でコンパイル後の型に名前を付ける {#naming-compiled-types-with-exec_spec_type}

`exec_spec_type!(T)` は、spec の型 `T` のコンパイル先である exec の型に展開されます。`exec_spec_type!(ref T)` は、引数に使われる借用の形に展開されます。たとえば、`exec_spec_type!(Seq<u32>)` は `Vec<u32>`、`exec_spec_type!(ref Seq<u32>)` は `&[u32]`、`exec_spec_type!(Point)` は `ExecPoint` です。これは、生成された関数を呼び出す exec コードを書くときに役立ちます。

## よくあるエラー {#common-errors}

### `Seq` のインデックス {#indexing-on-seq}

`Seq` にインデックスでアクセスするとき、コンパイル処理は、spec コードの中で `int` にキャストされた `usize` を期待します。
`int` へのキャストはコンパイル処理で取り除かれますが、spec コードが Verus に受け入れられるためには必要です。`Seq::index` が `int` を期待するからです。

正しい例を示します。
```
pub open spec fn my_index(s: Seq<u8>, i: usize) -> u8 
    recommends 0 <= i < s.len()
{
    s[i as int]
}
```
次の例は、`i` が `usize` ではなく `u64` なので誤りです。これは型エラーになります。
```
pub open spec fn my_index_err1(s: Seq<u8>, i: u64) -> u8 
    recommends 0 <= i < s.len()
{
    s[i as int]
}
```
次の例は誤りです。コンパイル処理は、spec コードの中でインデックスが `int` にキャストされていることを期待するからです。
これも型エラーになります。
```
pub open spec fn my_index_err2(s: Seq<u8>, i: usize) -> u8 
    recommends 0 <= i < s.len()
{
    s[i]
}
```

### `Map` のインデックス {#indexing-on-map}

`Map<K, V>` にインデックスでアクセスするには、キーの型 `K` がプリミティブ型でなければなりません。
`K` にプリミティブでない型を使うと、インデックスを使ったときに型エラーになります。

`K` にプリミティブでない型を使わなければならない場合は、回避策として `Map::get` を使ってください。

次のコードには、マクロは対応していません。
```
pub struct Data {
    x: usize
}

pub open spec fn my_index_err(m: Map<Data, u8>, i: Data) -> u8 
    recommends m.dom().contains(i)
{
    m[i]
}
```

代わりに次のように書きます。
```
pub struct Data {
    x: usize
}

pub open spec fn my_index_map(m: Map<Data, u8>, i: Data) -> u8
    recommends m.dom().contains(i)
{
    m.get(i).unwrap()
}
```

### 算術演算子 {#arithmetic-operators}

[Verus は spec コードの算術で型を広げる](spec-arithmetic.html)ので、算術式（`a + b` など）の型がオペランドの型と異なることがあります。そのため、両方のオペランドの型が対応している型（たとえば `u64`）であっても、算術式の型が `exec_spec_verified!` マクロと `exec_spec_unverified!` マクロの対応していない型（つまり `int` や `nat`）になることがあります。

たとえば、次のコードは型エラーになります。`x: u64` と `y: u64` に対して、Verus は spec コードの `x + y` を `int` として型付けするからです。
```
pub open spec fn my_arith_err(x: u64, y: u64) -> u64 
{
    x + y
}
```

これを直すには、結果を `u64` にキャストしなければなりません。
```
pub open spec fn my_arith(x: u64, y: u64) -> u64 
{
    (x + y) as u64
}
```

あるいは、`add(x, y)`（または `sub`/`mul`）を使います。これは型が `u64` で、オーバーフロー時にはラップアラウンドします。

### 完全修飾パス {#fully-qualified-paths}

`exec_spec_verified!` マクロと `exec_spec_unverified!` マクロは、現時点では `vstd::multiset::Multiset` のような完全修飾パスに対応していません。別のモジュールの型を使うには、ファイルの先頭でそのモジュールをインポートし、修飾なしのパス（先ほどの例では `Multiset`）を使ってください。完全修飾パスを使うと型エラーになります。

### `impl` ブロックの制限 {#impl-block-limitations}

マクロが対応している `impl` ブロックには、いくつかの制約があります。

- レシーバーとして対応しているのは `&self` だけです。値渡しの `self` や `&mut self` を使うとエラーになります。
- ジェネリックな実装とトレイトの実装には対応していません。

たとえば、次のものはすべて拒否されます。
```
impl W {
    pub open spec fn get(self) -> u32 { self.v }       // self by value
}

impl W {
    pub open spec fn get(&mut self) -> u32 { self.v }  // &mut self
}

impl<T> W { ... }                                      // generics

impl MyTrait for S { ... }                             // trait impl
```

## `exec_spec_unverified!` が生成したコードを実行する {#running-code-generated-by-exec_spec_unverified}

`exec_spec_unverified!` マクロは、コンパイルされた実行可能コードについて、停止性、算術オーバーフローが起きないこと、関数の事前条件が満たされることの Verus による証明を生成しません。この点が `exec_spec_verified!` と異なります。その代わり、`exec_spec_unverified!` が生成したコードは、事前条件に違反した場合（たとえば `None` を unwrap した場合）やオーバーフローが起きた場合に、実行時にパニックします。

### tracked とゴーストの状態 {#tracked-and-ghost-state}

`tracked` 引数、`tracked struct`/`tracked enum` のデータ型、フィールドの `tracked`/`ghost` 指定、`let tracked`/`let ghost` 束縛は拒否されます。パーミッション（permission）には実行可能な対応物が無いからです。`ghost struct`/`ghost enum` のデータ型は使えます。`PointsTo` のようなパーミッションについて推論するには、代わりにそのビュー（view）に対する spec 関数を書いてください。たとえば、`mem_contents()` が返す `MemContents<T>` に対して書きます。

## 関連 {#related}

`exec_spec_verified!` と `exec_spec_unverified!` は spec の項目を exec の項目へコンパイルしますが、これではあなたの用途に合わないかもしれません。
たとえば、[exec から spec へ](exec_to_spec.html)変換したい場合や、[`when_used_as_spec` 属性](reference-attributes.html)を使いたい場合があります。
{% endraw %}

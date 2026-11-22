{% raw %}
# 実行可能コードのための Verus の属性 {#verus-attributes-for-executable-code}

`#[verus_spec]` 属性マクロを使うと、読みやすさを損なわずに、既存の実行可能コードへ Verus の仕様と証明を追加できます。

## `verus!` マクロではなく `#[verus_spec]` を使う場面 {#when-to-use-verus_spec-instead-of-the-verus-macro}

Verus のコードを書く標準的な方法は、`verus!` マクロを使うことです。
しかし、`verus!` を使って仕様と証明を実行可能コードに直接埋め込む方法は、本番環境では必ずしも理想的ではありません。
とくに、開発者が既存の Rust プロジェクトに検証を組み込みたいと考えていて、次のことを目指す場合がそうです。

1. 実行可能コードへの変更を最小限にする — tracked 引数やゴースト引数を含めるために関数の API を変更することを避け、保守しやすいように Rust 本来の構文を保ちます。
2. 検証を段階的に導入する — 関数の API を全面的に書き直すことなく、既存の大きなコードベースへ Verus を少しずつ適用します。
3. 読みやすさを保つ — Verus に詳しくない開発者や、検証関連の注釈を無視したい開発者にとっても、検証済みのコードがすっきりと理解しやすいままであるようにします。
4. 依存関係の管理を最適化する — Verus の構成要素（`verus_builtin`、`verus_builtin_macros`、`vstd`）をモジュール式に使い、プロジェクトが独自のスタブマクロを定義したり、フィーチャーフラグで Verus への依存を制御したりできるようにします。

## 関数に仕様を追加する {#adding-specifications-to-a-function}

関数シグネチャに仕様を付けるには、`#[verus_spec(requires ... ensures ...)]` を使います。

例を示します。

```rust
#[verus_spec(sum => 
     requires 
         x < 100, 
         y < 100, 
     ensures 
         sum < 200, 
)]
fn my_exec_fun(x: u32, y: u32) -> u32 
{ 
    x + y 
}
```

## 検証のヒントを追加する {#adding-verification-hints}

`#[verus_verify(...)]` を使うと、検証器にヒントを与えて、関数を `external`、`external_body`、`spinoff_prover` として指定したり、`rlimit(amount)` で別のリソース上限を指定したりできます。

さらに、[`verus_verify(dual_spec)`](exec_to_spec.html) 属性を使うと、デュアルモード（spec + exec）の関数を作れます。この属性は、実行可能な関数から spec 関数を生成しようとします。

## 関数の中に証明を追加する {#adding-proofs-inside-functions}

関数に `#[verus_spec(...)]` 属性が付いていれば、以下で説明する証明用のマクロを使って、実行可能な関数の中に証明を直接書けます。

Rust が（Verus を使わずに）コードをビルドするときには、`#[verus_spec(...)]` 属性によって、すべての証明コードが確実に消去されます。

### 単純な証明ブロック {#simple-proof-blocks}

証明ブロックを追加するには `proof!` を使います。これは、`verus!` マクロの中で `proof { ... }` を使うのと等価です。
したがって、`proof!` の中で定義したゴースト変数や tracked 変数は、その証明ブロックに局所的であり、ほかの証明ブロックでは使えません。

### 証明ブロックをまたぐゴースト変数・tracked 変数 {#ghosttracked-variables-across-proof-blocks}

ゴースト変数や tracked 変数を複数の証明ブロックにまたがって使う必要があるときは、`proof_decl!` を使います。
次の例のように、証明ブロックを導入して、関数スコープのゴースト変数・tracked 変数を宣言できます。
```rust
#[verus_spec]
fn exec_with_proof() {
   proof_decl!{
     let ghost mut i = 0int;
     assert(true);
   }
   test_for_loop(10);
   proof!{
     assert(i == 0);
   }
}
```

`proof_decl!` の中で、文の位置でマクロを呼び出すことには対応していません。
その出力が、検証中には実行され、コンパイル中には消去されるということが起こりうるからです。
`proof_decl!` に展開されるマクロは、外側のスコープで直接呼び出してください。
証明の文に展開されるマクロは、`proof!` の中に置いてください。
ゴースト変数・tracked 変数の初期化子の中のマクロ式には、引き続き対応しています。

### 実行可能な関数の呼び出しにゴースト変数・tracked 変数を追加する {#adding-ghosttracked-variables-to-executable-function-calls}

* `#[verus_spec(with ...)]`: tracked 変数やゴースト変数を、実行可能な関数の引数または戻り値として追加します。

   これにより、元の関数の 2 つの版が生成されます。
   * 検証済みの版（ゴースト引数・tracked 引数付き）。検証済みのコードで使われます。
   * 未検証の版（元のシグネチャのまま）。未検証のコードから呼び出せます。

* `proof_with!`
   `verus_spec` と組み合わせて働き、tracked 変数・ゴースト変数を呼び出し先に渡します。
   
   関数呼び出しの前に `proof_with!` があると、検証済みの版が使われます。
   そうでなければ、未検証の版が選ばれます。
   未検証の版には `requires false` という事前条件が含まれています。そのため、`proof_with!` の使い方が不適切な場合、検証済みのコードから呼び出すと必ず検証に失敗します。

例を示します。

```rust
#[verus_spec(ret =>
with
  Tracked(y): Tracked<&mut int>,
  Ghost(w): Ghost<u32> 
     -> z: Ghost<u32>
requires
  x < 100,
  *old(y) < 100,
ensures
  *final(y) == x,
  ret == x + 1,
  z@ == x,
)]
fn exec_tracked(x: u32) -> u32 {
  proof! {
    *y = x as int;
  }
  proof_with!(|= Ghost(x));
  (x + 1)
}


#[verus_spec]
fn exec_tracked_test(x: u32) {
  proof_decl!{
    let ghost mut z = 0u32;
    let tracked mut y = 0;
  }

  proof_with!{Tracked(&mut y), Ghost(0) => Ghost(z)}
  let x = exec_tracked(1);

  proof!{
    assert(y == 1);
    assert(z == 1);
    assert(x == 2);
  }
}

fn exec_external_test(x: u32) -> u32 {
   exec_tracked(1)
}

```

### `#[verus_spec]` と `verus!` を混ぜて使う {#using-a-mix-of-verus_spec-and-verus}

`#[verus_spec]` と `verus!` の望ましい使い方は、実行可能な関数にはすべて `#[verus_spec]` を使い、spec 関数と proof 関数には `verus!` を使うことです。

注意: `#[verus_spec(with ...)]` と `proof_with!` の組み合わせには、関係する関数がゴースト変数・tracked 変数を受け取ったり返したりする場合、`verus!` の中で定義された実行可能な関数との互換性の問題が現時点であります。

具体的には、`proof_with!` は、`#[verus_spec]` で検証された exec 関数に対して働きます。
`verus!` で検証された exec 関数に `proof_with!` を使ってゴースト変数・tracked 変数を渡すと、次のエラーになります。

```text
[E0425]: cannot find function `_VERUS_VERIFIED_xxx` in this scope.
```

これは、`verus!` がつねに関数のシグネチャを実際に変更し、関数定義を 1 つしか持たないのに対して、`#[verus_spec]` は検証済みの版と未検証の版の 2 つがあることを前提にしているからです。

`verus!` で検証された関数を使うための回避策は、信頼された（trusted）ラッパー関数を作り、それを使うことです。

```
#[verus_verify(external_body)]
#[verus_spec(v =>
   with Tracked(perm): Tracked<&mut PointsTo<T>>
   requires
        old(perm).ptr() == ptr,
        old(perm).is_init(),
    ensures
        perm.ptr() == ptr,
        perm.is_uninit(),
        v == old(perm).value(),
)]
fn ptr_mut_read<'a, T>(ptr: *const T)  -> T
{
   vstd::raw_ptr::ptr_mut_read(ptr, Tracked::assume_new())
}
```


## そのほかの例とテスト {#more-examples-and-tests}

```rust
#![feature(proc_macro_hygiene)]
#![allow(unused_imports)]

use verus_builtin::*;
use verus_builtin_macros::*;
use vstd::{modes::*, prelude::*, seq::*, *};

fn main() {}

/// functions may be declared exec (default), proof, or spec, which contain
/// exec code, proof code, and spec code, respectively.
///   - exec code: compiled, may have requires/ensures
///   - proof code: erased before compilation, may have requires/ensures
///   - spec code: erased before compilation, no requires/ensures, but may have recommends
/// exec and proof functions may name their return values inside parentheses, before the return type
#[verus_spec(sum =>
    requires
        x < 100,
        y < 100,
    ensures
        sum < 200,
)]
fn my_exec_fun(x: u32, y: u32) -> u32
{
    x + y
}

verus! {

proof fn my_proof_fun(x: int, y: int) -> (sum: int)
    requires
        x < 100,
        y < 100,
    ensures
        sum < 200,
{
    x + y
}

spec fn my_spec_fun(x: int, y: int) -> int
    recommends
        x < 100,
        y < 100,
{
    x + y
}

} // verus!

/// exec code cannot directly call proof functions or spec functions.
/// However, exec code can contain proof blocks (proof { ... }),
/// which contain proof code.
/// This proof code can call proof functions and spec functions.
#[verus_spec(
    requires
        x < 100,
        y < 100,
)]
fn test_my_funs(x: u32, y: u32)
{
    // my_proof_fun(x, y); // not allowed in exec code
    // let u = my_spec_fun(x, y); // not allowed exec code
    proof! {
        let u = my_spec_fun(x as int, y as int);  // allowed in proof code
        my_proof_fun(u / 2, y as int);  // allowed in proof code
    }
}

verus! {

/// spec functions with pub or pub(...) must specify whether the body of the function
/// should also be made publicly visible (open function) or not visible (closed function).
pub open spec fn my_pub_spec_fun1(x: int, y: int) -> int {
    // function and body visible to all
    x / 2 + y / 2
}

/* TODO
pub open(crate) spec fn my_pub_spec_fun2(x: u32, y: u32) -> u32 {
    // function visible to all, body visible to crate
    x / 2 + y / 2
}
*/

// TODO(main_new) pub(crate) is not being handled correctly
// pub(crate) open spec fn my_pub_spec_fun3(x: int, y: int) -> int {
//     // function and body visible to crate
//     x / 2 + y / 2
// }
pub closed spec fn my_pub_spec_fun4(x: int, y: int) -> int {
    // function visible to all, body visible to module
    x / 2 + y / 2
}

pub(crate) closed spec fn my_pub_spec_fun5(x: int, y: int) -> int {
    // function visible to crate, body visible to module
    x / 2 + y / 2
}

} // verus!

/// Recursive functions must have decreases clauses so that Verus can verify that the functions
/// terminate.
#[verus_spec(
    requires
        0 < x < 100,
        y < 100 - x,
    decreases x,
)]
fn test_rec(x: u64, y: u64)
{
    if x > 1 {
        test_rec(x - 1, y + 1);
    }
}

verus! {

/// Multiple decreases clauses are ordered lexicographically, so that later clauses may
/// increase when earlier clauses decrease.
spec fn test_rec2(x: int, y: int) -> int
    decreases x, y,
{
    if y > 0 {
        1 + test_rec2(x, y - 1)
    } else if x > 0 {
        2 + test_rec2(x - 1, 100)
    } else {
        3
    }
}

/// To help prove termination, recursive spec functions may have embedded proof blocks
/// that can make assertions, use broadcasts, and call lemmas.
spec fn test_rec_proof_block(x: int, y: int) -> int
    decreases x,
{
    if x < 1 {
        0
    } else {
        proof {
            assert(x - 1 >= 0);
        }
        test_rec_proof_block(x - 1, y + 1) + 1
    }
}

/// Decreases and recommends may specify additional clauses:
///   - decreases .. "when" restricts the function definition to a condition
///     that makes the function terminate
///   - decreases .. "via" specifies a proof function that proves the termination
///     (although proof blocks are usually simpler; see above)
///   - recommends .. "when" specifies a proof function that proves the
///     recommendations of the functions invoked in the body
spec fn add0(a: nat, b: nat) -> nat
    recommends
        a > 0,
    via add0_recommends
{
    a + b
}

spec fn dec0(a: int) -> int
    decreases a,
    when a > 0
    via dec0_decreases
{
    if a > 0 {
        dec0(a - 1)
    } else {
        0
    }
}

#[via_fn]
proof fn add0_recommends(a: nat, b: nat) {
    // proof
}

#[via_fn]
proof fn dec0_decreases(a: int) {
    // proof
}

} // verus!

/// variables may be exec, tracked, or ghost
///   - exec: compiled
///   - tracked: erased before compilation, checked for lifetimes (advanced feature, discussed later)
///   - ghost: erased before compilation, no lifetime checking, can create default value of any type
/// Different variable modes may be used in different code modes:
///   - variables in exec code are always exec
///   - variables in proof code are ghost by default (tracked variables must be marked "tracked")
///   - variables in spec code are always ghost
/// For example:
#[verus_spec(
    requires
        a < 100,
        b < 100,
)]
fn test_my_funs2(
    a: u32,  // exec variable
    b: u32,  // exec variable
)
{
    let s = a + b;  // s is an exec variable
    proof! {
        let u = a + b;  // u is a ghost variable
        my_proof_fun(u / 2, b as int);  // my_proof_fun(x, y) takes ghost parameters x and y
    }
}

verus! {

/// assume and assert are treated as proof code even outside of proof blocks.
/// "assert by" may be used to provide proof code that proves the assertion.
#[verifier::opaque]
spec fn f1(i: int) -> int {
    i + 1
}

} // verus!

#[verus_spec()]
fn assert_by_test() {
    proof! {
        assert(f1(3) > 3) by {
            reveal(f1);  // reveal f1's definition just inside this block
        }
        assert(f1(3) > 3);
    }
}

/// "assert by" can also invoke specialized provers for bit-vector reasoning or nonlinear arithmetic.
#[verus_spec()]
fn assert_by_provers(x: u32) {
    proof! {
        assert(x ^ x == 0u32) by (bit_vector);
        assert(2 <= x && x < 10 ==> x * x > x) by (nonlinear_arith);
    }
}

verus! {

/// "let ghost" currently requires the verus! macro
/// Variables in exec code may be exec, ghost, or tracked.
fn test_ghost(x: u32, y: u32)
    requires
        x < 100,
        y < 100,
{
    let ghost u: int = my_spec_fun(x as int, y as int);
    let ghost mut v = u + 1;
    assert(v == x + y + 1);
    proof {
        v = v + 1;  // proof code may assign to ghost mut variables
    }
    let ghost w = {
        let temp = v + 1;
        temp + 1
    };
    assert(w == x + y + 4);
}

/// Ghost(...) expressions and patterns currently require the verus! macro
/// Ghost(...) and Tracked(...) patterns can unwrap Ghost<...> and Tracked<...> values:
fn test_ghost_unwrap(
    x: u32,
    Ghost(y): Ghost<u32>,
)  // unwrap so that y has typ u32, not Ghost<u32>
    requires
        x < 100,
        y < 100,
{
    // Ghost(u) pattern unwraps Ghost<...> values and gives u and v type int:
    let Ghost(u): Ghost<int> = Ghost(my_spec_fun(x as int, y as int));
    let Ghost(mut v): Ghost<int> = Ghost(u + 1);
    assert(v == x + y + 1);
    proof {
        v = v + 1;  // assign directly to ghost mut v
    }
    let Ghost(w): Ghost<int> = Ghost(
        {
            // proof block that returns a ghost value
            let temp = v + 1;
            temp + 1
        },
    );
    assert(w == x + y + 4);
}

} // verus!

/// Trait functions may have specifications
trait T {
    #[verus_spec(r =>
        requires
            0 <= i < 10,
            0 <= j < 10,
        ensures
            i <= r,
            j <= r,
    )]
    fn my_uninterpreted_fun2(&self, i: u8, j: u8) -> u8;
}

#[verus_spec(ret =>
    with
        Tracked(y): Tracked<&mut int>, Ghost(w): Ghost<u64> -> z: Ghost<u32>
    requires
        x < 100,
        *old(y) < 100,
    ensures
        *final(y) == x,
        ret == x,
        z == x,
)]
fn test_mut_tracked(x: u32) -> u32 {
    proof!{
        *y = x as int;
    }
    #[verus_spec(with |=Ghost(x))]
    x
}

fn test_cal_mut_tracked(x: u32) {
    proof_decl!{
        let ghost mut z;
        let tracked mut y = 0;
        z = 0u32;
    }
    #[verus_spec(with Tracked(&mut y), Ghost(0) => Ghost(z))]
    let _ = test_mut_tracked(0u32);

    (#[verus_spec(with Tracked(&mut y), Ghost(0))]
    test_mut_tracked(0u32));

    return;
}
```

```rust
#![feature(rustc_private)]
#[macro_use]
mod common;
use common::*;

test_verify_one_file_with_options! {
    #[test] param_patterns_named_return ["vstd"] => code! {
        use vstd::prelude::*;

        #[verus_spec(r => ensures r == s)]
        fn strings((s, t): (String, String)) -> String {
            let _ = t;
            s
        }

        #[verus_spec(result => ensures result == value)]
        fn f<T>(_: T, value: u32, _: &mut u64) -> u32 {
            value
        }

        #[verus_spec(result => ensures result.1)]
        fn g(_: impl Sized, _: &impl Sized) -> (impl Sized, bool) {
            (true, true)
        }

        #[verus_spec]
        fn caller() {
            let mut value = 30u64;
            let result = f(false, 20, &mut value);
            proof! { assert(result == 20); }
            let result = g(true, &false);
            proof! { assert(result.1); }
        }
    } => Ok(())
}

test_verify_one_file_with_options! {
    #[test] verus_verify_basic_while ["exec_allows_no_decreases_clause"] =>  code! {
        #[verus_spec]
        fn test1() {
            let mut i = 0;
            #[verus_spec(
                invariant
                    i <= 10
            )]
            while i < 10
            {
                i = i + 1;
            }
            proof!{assert(i == 10);}
        }
    } => Ok(())
}

test_verify_one_file_with_options! {
    #[test] verus_verify_basic_loop ["exec_allows_no_decreases_clause"] => code! {
        #[verus_spec]
        fn test1() {
            let mut i = 0;
            let mut ret = 0;
            #[verus_spec(
                invariant i <= 10,
                invariant_except_break i <= 9,
                ensures i == 10, ret == 10
            )]
            loop
            {
                i = i + 1;
                if (i == 10) {
                    ret = i;
                    break;
                }
            }
            proof!{assert(ret == 10);}
        }
    } => Ok(())
}

test_verify_one_file_with_options! {
    #[test] verus_verify_basic_for_loop_verus_spec ["exec_allows_no_decreases_clause"] =>  code! {
        use vstd::prelude::*;
        #[verus_spec(v =>
            ensures
                v.len() == n,
                forall|i: int| 0 <= i < n ==> v[i] == i
        )]
        fn test_for_loop(n: u32) -> Vec<u32>
        {
            let mut v: Vec<u32> = Vec::new();
            #[verus_spec(
                invariant
                    v@ =~= Seq::new(i as nat, |k| k as u32),
            )]
            for i in 0..n
            {
                v.push(i);
            }
            v
        }
    } => Ok(())
}

test_verify_one_file_with_options! {
    #[test] verus_verify_for_loop_verus_spec_naming_iter ["exec_allows_no_decreases_clause"] =>  code! {
        use vstd::prelude::*;
        #[verus_spec(v =>
            ensures
                v.len() == n,
                forall|i: int| 0 <= i < n ==> v[i] == 0
        )]
        fn test(n: u32) -> Vec<u32>
        {
            let mut v: Vec<u32> = Vec::new();
            #[verus_spec(iter =>
                invariant
                    v@ =~= Seq::new(iter.index() as nat, |k| 0u32),
            )]
            for _ in 0..n
            {
                v.push(0);
            }
            v
        }
    } => Ok(())
}

test_verify_one_file_with_options! {
    #[test] verus_verify_basic_while_fail1 ["exec_allows_no_decreases_clause"] => code! {
        #[verus_spec]
        fn test1() {
            let mut i = 0;
            while i < 10 {
                i = i + 1;
            }
            proof!{assert(i == 10);} // FAILS
        }
    } => Err(err) => assert_one_fails(err)
}

test_verify_one_file_with_options! {
    #[test] basic_while_false_invariant ["exec_allows_no_decreases_clause"] => code! {
        #[verus_verify]
        fn test1() {
            let mut i = 0;
            #[verus_spec(
                invariant
                    i <= 10, false
            )]
            while i < 10 {
                i = i + 1;
            }
        }
    } => Err(err) => assert_any_vir_error_msg(err, "invariant not satisfied before loop")
}

test_verify_one_file_with_options! {
    #[test] verus_verify_invariant_on_func ["exec_allows_no_decreases_clause"] => code! {
        #[verus_spec(
            invariant true
        )]
        fn test1() {}
    } => Err(err) => assert_any_vir_error_msg(err, "unexpected token")
}

test_verify_one_file! {
    #[test] test_use_macro_attributes code!{
        struct Abc<T> {
            t: T,
        }

        trait SomeTrait {
            #[verus_spec(ret =>
                requires true
                ensures ret
            )]
            fn f(&self) -> bool;
        }

        impl<S> Abc<S> {
            fn foo(&self)
                where S: SomeTrait
            {
                let ret = self.t.f();
                proof!{assert(ret);}
            }
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_bad_macro_attributes_in_trait code!{
        trait SomeTrait {
            #[verus_spec(
                ensures
                    true
            )]
            type T;
        }
    } => Err(err) => assert_any_vir_error_msg(err, "Misuse of #[verus_spec]")
}

test_verify_one_file_with_options! {
    #[test] test_no_verus_verify_attributes_in_trait_impl ["--no-external-by-default"] => code!{
        struct Abc<T> {
            t: T,
        }

        #[verus_verify]
        trait SomeTrait {
            #[verus_spec(requires true)]
            fn f(&self) -> bool;
        }

        impl<S> Abc<S> {
            fn foo(&self)
                where S: SomeTrait
            {
                let ret = self.t.f();
                proof!{assert(ret);}
            }
        }
    } => Err(err) => assert_vir_error_msg(err, "assertion failed")
}

test_verify_one_file! {
    #[test] test_failed_ensures_macro_attributes code!{
        #[verus_verify]
        trait SomeTrait {
            #[verus_spec(ret =>
                requires true
                ensures true, ret
            )]
            fn f(&self) -> bool;
        }

        #[verus_verify]
        impl SomeTrait for bool {
            fn f(&self) -> bool {
                *self
            }
        }
    } => Err(err) => assert_any_vir_error_msg(err, "postcondition not satisfied")
}

test_verify_one_file! {
    #[test] test_default_fn_use_macro_attributes code!{
        #[verus_verify]
        struct Abc<T> {
            t: T,
        }

        #[verus_verify]
        trait SomeTrait {
            #[verus_spec(ret =>
                requires true
                ensures ret
            )]
            fn f(&self) -> bool {
                true
            }
        }

        #[verus_verify]
        impl<S> Abc<S> {
            fn foo(&self)
                where S: SomeTrait
            {
                let ret = self.t.f();
                proof!{assert(ret);}
            }
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_default_failed_fn_use_macro_attributes code!{
        #[verus_verify]
        struct Abc<T> {
            t: T,
        }

        #[verus_verify(rlimit(2))]
        trait SomeTrait {
            #[verus_spec(ret =>
                requires true
                ensures ret, !ret
            )]
            fn f(&self) -> bool {
                true
            }
        }
    } => Err(err) => assert_vir_error_msg(err, "postcondition not satisfied")
}

test_verify_one_file! {
    #[test] test_external_body code!{
        #[verus_verify(external_body)]
        #[verus_spec(ret =>
            requires true
            ensures ret
        )]
        fn f() -> bool {
            false
        }

        fn g() {
            let r = f();
            proof!{assert(r);}
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_external_with_unsupported_features code!{
        #[verus_verify(external)]
        fn f<'a>(v: &'a mut [usize]) -> &'a mut usize {
            unimplemented!()
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_prover_attributes code!{
        #[verus_verify(spinoff_prover, rlimit(2))]
        #[verus_spec(
            ensures
                true
        )]
        fn test()
        {}
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_invalid_combination_attr code!{
        #[verus_verify(external, spinoff_prover)]
        fn f<'a>(v: &'a mut [usize]) -> &'a mut usize {
            unimplemented!()
        }
    } => Err(e) => assert_any_vir_error_msg(e, "conflict parameters")
}

test_verify_one_file! {
    #[test] test_proof_decl code!{
        #[verus_spec]
        fn f() {}
        #[verus_spec]
        fn test() {
            proof!{
                let x = 1 as int;
                assert(x == 1);
            }
            proof_decl!{
                let ghost mut x;
                let tracked y = false;
                x = 2int;
                assert(!y);
                if x == 1 {
                    assert(false);
                }
            }

            f();
            proof!{
                assert(!y);
                assert(x == 2);
            }
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_proof_decl_reject_exec code!{
        #[verus_spec]
        fn f() {}
        #[verus_spec]
        fn test() {
            proof_decl!{
                f();
            }
            f();
        }
    } => Err(e) => assert_vir_error_msg(e, "cannot call function `test_crate::f` with mode exec")
}

test_verify_one_file! {
    #[test] test_proof_decl_reject_exec_local code!{
        #[verus_spec]
        fn test() {
            proof_decl!{
                let x = true;
            }
        }
    } => Err(e) => assert_vir_error_msg(e, "Exec local is not allowed in proof_decl")
}

test_verify_one_file! {
    #[test] test_proof_decl_reject_macro_assignment code!{
        macro_rules! bump {
            ($x:ident) => { $x = 1; };
        }

        #[verus_spec(r => ensures r == 1)]
        fn test() -> u32 {
            let mut x = 0u32;
            proof_decl! {
                bump!(x);
            }
            x
        }
    } => Err(e) => {
        let primary = e.errors[0].spans.iter().find(|s| s.is_primary).unwrap();
        assert_eq!(primary.line_start, primary.line_end);
        assert!(primary.text[0].text.contains("bump!(x)"));
        assert_vir_error_msg(e, "Macro statements are not supported in proof_decl!");
    }
}

test_verify_one_file! {
    #[test] test_with code!{
        #[verus_spec(ret =>
            with
                Tracked(y): Tracked<&mut int>,
                Ghost(w): Ghost<u32>,
                -> z: Ghost<u32>
            requires
                x < 100,
                *old(y) < 100,
            ensures
                *final(y) == x,
                ret == x,
                z@ == x,
        )]
        fn test_mut_tracked(x: u32) -> u32 {
            proof!{
                *y = x as int;
            }
            #[verus_spec(with |= Ghost(x))]
            x
        }

        #[verus_spec]
        fn test_call_mut_tracked(x: u32) {
            proof_decl!{
                let tracked mut y = 0;
            }
            {#[verus_spec(with Tracked(&mut y), Ghost(0) => _)]
            test_mut_tracked(1);
            };

            if x < 100 && #[verus_spec(with Tracked(&mut y), Ghost(0) => _)]test_mut_tracked(x) == 0 {
                return;
            }

            #[verus_spec(with Tracked(&mut y), Ghost(0) => Ghost(z))]
            let _ = test_mut_tracked(1);

            proof!{
                assert(y == 1);
                assert(z == 1);
            }
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_trait_signature code!{
        trait X {
            #[verus_spec(ret =>
                with
                    Tracked(y): Tracked<&mut u32>,
                    Ghost(w): Ghost<u32>,
                    -> z: Ghost<u32>
            )]
            fn f(&self, x: u32) -> bool;
        }
    } => Err(e) => assert_any_vir_error_msg(e, "`with` does not support trait")
}

test_verify_one_file! {
    #[test] test_unverified_code_signature code!{
        #[verus_spec(ret =>
            with
                Tracked(y): Tracked<&mut int>,
                Ghost(w): Ghost<u32>,
                -> z: Ghost<u32>
        )]
        fn test_mut_tracked(x: u32) -> u32 {
            proof!{
                *y = x as int;
            }
            #[verus_spec(with |= Ghost(x))]
            x
        }

        #[verifier::external]
        fn external_call_with_dummy(x: u32) -> u32 {
            #[verus_spec(with Tracked::assume_new(), Ghost::assume_new() => _)]
            test_mut_tracked(0)
        }

        #[verifier::external]
        fn external_call_untouched(x: u32) -> u32 {
            test_mut_tracked(0)
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_verified_call_unverified_signature code!{
        #[verus_spec(ret =>
            with
                Tracked(y): Tracked<&mut int>,
                Ghost(w): Ghost<u32>,
                -> z: Ghost<u32>
        )]
        fn test_mut_tracked(x: u32) -> u32 {
            proof!{
                *y = x as int;
            }
            #[verus_spec(with |= Ghost(x))]
            x
        }

        #[verus_spec]
        fn verified_call_unverified(x: u32) {
            test_mut_tracked(0); // FAILS
        }
    } => Err(e) => {
        assert!(e.errors[0].rendered.contains("with"));
        assert_one_fails(e)
    }
}

test_verify_one_file! {
    #[test] test_with2 code!{
        #[verus_spec(ret =>
            with
                Tracked(y): Tracked<&mut int>,
                Ghost(w): Ghost<u32>,
                ->  z: Ghost<u32>
            requires
                x < 100,
                *old(y) < 100,
            ensures
                *final(y) == x,
                ret == x,
                z@ == x,
        )]
        fn test_mut_tracked(x: u32) -> u32 {
            proof!{
                *y = x as int;
            }
            #[verus_spec(with |= Ghost(x))]
            x
        }

        #[verus_spec]
        fn test_cal_mut_tracked(x: u32) {
            proof_decl!{
                let ghost mut z = 0u32;
                let tracked mut y = 0;
            }
            if #[verus_spec(with Tracked(&mut y), Ghost(0) => Ghost(z))] test_mut_tracked(1) == 0 {
                proof!{
                    assert(z == 1);
                }
                return;
            }

            proof!{
                assert(y == 1);
            }
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_proof_with code!{
        #[verus_spec(ret =>
            with
                Tracked(y): Tracked<&mut int>,
                Ghost(w): Ghost<u32>,
                ->  z: Ghost<u32>
            requires
                x < 100,
                *old(y) < 100,
            ensures
                *final(y) == x,
                ret == x,
                z@ == x,
        )]
        fn test_mut_tracked(x: u32) -> u32 {
            proof!{
                *y = x as int;
            }
            proof_with!{|= Ghost(x)}
            x
        }

        #[verus_spec]
        fn test_cal_mut_tracked(x: u32) {
            proof_decl!{
                let ghost mut z = 0u32;
                let tracked mut y = 0;
            }
            if {
                proof_with!{Tracked(&mut y), Ghost(0) => Ghost(z)} test_mut_tracked(1)
            } == 0 {
                proof!{
                    assert(z == 1);
                }
                return;
            }

            proof!{
                assert(y == 1);
            }
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_proof_with_try code!{
        use vstd::prelude::*;
        #[verus_spec(
            with Tracked(y): Tracked<&mut u32>
        )]
        fn f() -> Result<(), ()> {
            Ok(())
        }

        #[verus_spec(
            with Tracked(y): Tracked<&mut u32>
        )]
        fn test_try_call() -> Result<(), ()> {
            proof_with!{Tracked(y)}
            let _ = f()?;
            Ok(())
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_proof_with_callee_accepted code!{
        use vstd::prelude::*;

        #[verus_spec(r => with Ghost(g): Ghost<u8> ensures r == x)]
        pub fn f(x: u8) -> u8 { x }

        pub mod m {
            use vstd::prelude::*;
            #[verus_spec(r => with Ghost(g): Ghost<u8> ensures r == x)]
            pub fn h(x: u8) -> u8 { x }
        }

        #[verus_spec(r => with Ghost(g): Ghost<u8> ensures r.is_none())]
        pub fn none<T>() -> Option<T> { None }

        #[verus_spec(r => with Ghost(g): Ghost<u8>)]
        pub fn mk<T: Copy>(x: T) -> T { x }

        #[verus_spec(r => with Ghost(g): Ghost<u8>)]
        pub fn run<F: Fn(u8) -> u8>(cl: F) -> u8 { 0 }

        #[verus_spec(r =>
            with Ghost(g): Ghost<u8> -> out: (Ghost<u8>, Ghost<u8>)
            ensures r == x, out.0@ == g
        )]
        pub fn two(x: u8) -> u8 { proof_with!{|= (Ghost(g), Ghost(g))} x }

        #[verus_verify]
        pub trait Tr {
            #[verus_spec(r => with Ghost(g): Ghost<u8> ensures r == 1)]
            fn t(&self) -> u8 { 1 }
        }

        #[verus_verify]
        pub struct A;

        #[verus_verify]
        impl Tr for A {}

        #[verus_spec]
        pub fn call(a: A) {
            proof_with!{Ghost(0u8)}
            let x = m::h(1);
            // generic argument inferred from the expected type
            let n: Option<u32> = {
                proof_with!{Ghost(0u8)}
                none()
            };
            // exec result that is itself a tuple
            let p: (u8, u16) = {
                proof_with!{Ghost(0u8)}
                mk((1, 2))
            };
            // closures in the arguments
            proof_with!{Ghost(0u8)}
            let q = run(|x: u8| { let k = |y: u8| y; k(x) });
            // nested proof_with!
            proof_with!{Ghost(0u8)}
            let r = f({ proof_with!{Ghost(1u8)} f(2) });
            // ghost outputs bound and ignored
            proof_with!{Ghost(5u8) => (Ghost(o1), Ghost(o2))}
            let s = two(9);
            proof_with!{Ghost(5u8)}
            two(9);
            // trait default method
            proof_with!{Ghost(0u8)}
            let t = a.t();
            proof!{ assert(x == 1 && n.is_none() && r == 2 && s == 9 && o1 == 5 && t == 1); }
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_proof_with_callee_shadowed_by_local_fn code!{
        use vstd::prelude::*;

        #[verus_spec(r =>
            with Ghost(g): Ghost<u8>
            ensures r == 1
        )]
        pub fn f(x: u8) -> u8 { 1 }

        #[verus_spec(r => ensures r == 1)]
        pub fn call() -> u8 {
            // The compiled call `f(7)` resolves to this `f`, but `_VERUS_VERIFIED_f` does not.
            #[verus_spec(r => ensures r == 0)]
            fn f(x: u8) -> u8 { 0 }

            proof_with!{Ghost(0u8)}
            f(7)
        }
    } => Err(err) => assert_vir_error_msg(err, "in the compiled program, `f` here resolves to `call::f`, not to `f` (declared with `#[verus_spec(with ...)]`)")
}

test_verify_one_file_with_options! {
    #[test] test_proof_with_callee_incremental ["--incremental"] => code!{
        // With incremental compilation, rustc sets the parent of item spans,
        // so the spans of `f` and `_VERUS_VERIFIED_f` differ in their parent field
        #[verus_verify]
        struct S { x: u64 }

        #[verus_verify]
        impl S {
            #[verus_spec(
                with Tracked(perm): Tracked<&mut u64>
                requires old(self).x < 10,
            )]
            fn write(&mut self, v: u64) {
                self.x = 1;
            }

            #[verus_spec(
                with Tracked(perm): Tracked<&mut u64>
                requires old(self).x < 10,
            )]
            fn caller(&mut self) {
                proof_with!(Tracked(perm));
                self.write(3);
            }
        }

        #[verus_spec(r =>
            with Ghost(g): Ghost<u8>
            ensures r == 1
        )]
        pub fn f(x: u8) -> u8 { 1 }

        #[verus_spec(r => ensures r == 1)]
        pub fn call() -> u8 {
            proof_with!{Ghost(0u8)}
            f(7)
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_proof_with_callee_ghost_arg_fixes_generic code!{
        use vstd::prelude::*;

        verus! {
        pub trait Tr { spec fn sv() -> u8; fn v() -> (r: u8) ensures r == Self::sv(); }
        impl Tr for u64 { open spec fn sv() -> u8 { 1 } fn v() -> u8 { 1 } }
        impl Tr for i32 { open spec fn sv() -> u8 { 0 } fn v() -> u8 { 0 } }
        }

        #[verus_spec(r =>
            with Ghost(g): Ghost<T>
            ensures r == T::sv()
        )]
        pub fn h<T: Tr>(x: T) -> u8 { T::v() }

        #[verus_spec(r => ensures r == 1)]
        pub fn call() -> u8 {
            // the verified call infers T = u64 from the ghost argument,
            // the compiled call `h(3)` infers T = i32
            proof_with!{Ghost(5u64)}
            h(3)
        }
    } => Err(err) => assert_vir_error_msg(err, "in the compiled program, this call to `h` has different type arguments than in the verified program")
}

test_verify_one_file! {
    #[test] test_proof_with_trait_default_method_overridden code!{
        use vstd::prelude::*;

        #[verus_verify]
        pub trait Tr {
            #[verus_spec(r =>
                with Ghost(g): Ghost<u8>
                ensures r == 1
            )]
            fn f(&self) -> u8 { 1 }
        }

        #[verus_verify]
        pub struct A;

        #[verus_verify]
        impl Tr for A {
            // `f` is called by the compiled program, `_VERUS_VERIFIED_f` by the verified one
            fn f(&self) -> u8 { 0 }
        }

        #[verus_spec(r => ensures r == 1)]
        pub fn call<X: Tr>(a: X) -> u8 {
            proof_with!{Ghost(0u8)}
            a.f()
        }
    } => Err(err) => assert_vir_error_msg(err, "cannot override `Tr::f`: a trait method declared with `#[verus_spec(with ...)]` cannot be overridden")
}

test_verify_one_file! {
    #[test] test_proof_with_callee_handwritten_extra_stmt code!{
        use vstd::prelude::*;

        #[verus_spec(r =>
            with -> out: Ghost<u8>
            ensures r == 1
        )]
        pub fn g(x: u8) -> u8 { proof_with!{|= Ghost(0u8)} 1 }

        #[verus_spec(r => ensures r == 0)]
        pub fn other(x: u8) -> u8 { 0 }

        #[verus_spec]
        pub fn call() {
            let r = verus_builtin::proof_with_callee(
                { let (tmp, o) = _VERUS_VERIFIED_g(7); other(1); tmp },
                g(7),
            );
        }
    } => Err(err) => assert_vir_error_msg(err, "ill-formed `proof_with!` call")
}

test_verify_one_file! {
    #[test] test_proof_with_ghost_output_sealed code!{
        use vstd::prelude::*;

        pub struct S;
        impl verus_builtin::ProofWithGhostOutput for S {}
    } => Err(err) => assert_rust_error_msg(err, "the trait bound `S: verus_builtin::proof_with_sealed::SealedOutput` is not satisfied")
}

test_verify_one_file! {
    #[test] test_proof_with_err code!{
        #[verus_spec]
        fn test_mut_tracked(x: u32) -> u32 {
            proof_declare!{
                let ghost y = x;
            }
            proof_with!{Ghost(y)}
            x
        }
    } => Err(e) => assert_any_vir_error_msg(e, "with ghost inputs/outputs cannot be applied to a non-call expression")
}

test_verify_one_file! {
    #[test] test_dual_spec code!{
        #[verus_verify(dual_spec(spec_f))]
        #[verus_spec(
            requires
                x < 100,
                y < 100,
            returns f(x, y)
        )]
        fn f(x: u32, y: u32) -> u32 {
            proof!{
                assert(true);
            }
            {
                proof!{assert(true);}
                x + y
            }
        }

        #[verus_verify(dual_spec)]
        #[verus_spec(
            requires
                x < 100,
            returns
                f2(x),
        )]
        pub fn f2(x: u32) -> u32 {
            f(x, 1)
        }

        #[verus_verify]
        struct S;

        impl S {
            #[verus_verify(dual_spec)]
            #[verus_spec(
                returns
                    self.foo(x),
            )]
            fn foo(&self, x: u32) -> u32 {
                x
            }
        }

        verus!{
        proof fn lemma_f(x: u32, y: u32)
        requires
            x < 100,
        ensures
            y == 1 ==> f(x, y) == f2(x),
            f(x, y) == spec_f(x, y),
            f2(x) == __VERUS_SPEC_f2(x),
            f(x, y) == (x + y) as u32,
            __VERUS_SPEC_f2(x) == x + 1,
        {}

        mod inner {
            use super::*;
            proof fn lemma_f(x: u32)
            requires
                x < 100,
            ensures
                f2(x) == __VERUS_SPEC_f2(x),
                __VERUS_SPEC_f2(x) == (x + 1),
            {}
        }
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_dual_spec_open code!{
        mod m {
            use super::*;
            #[verus_verify(dual_spec, open)]
            #[verus_spec(returns f(x))]
            pub fn f(x: u32) -> u32 {
                x / 2
            }
        }

        verus!{
        proof fn lemma_f(x: u32)
            ensures crate::m::f(x) == x / 2,
        {}
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_dual_spec_closed code!{
        mod m {
            use super::*;
            #[verus_verify(dual_spec, closed)]
            #[verus_spec(returns f(x))]
            pub fn f(x: u32) -> u32 {
                x / 2
            }
        }

        verus!{
        proof fn lemma_f(x: u32)
            ensures crate::m::f(x) == x / 2,
        {}
        }
    } => Err(e) => assert_vir_error_msg(e, "postcondition not satisfied")
}

test_verify_one_file! {
    #[test] test_dual_spec_opaque code!{
        #[verus_verify(dual_spec(f_spec), open, opaque)]
        #[verus_spec(returns f(x))]
        pub fn f(x: u32) -> u32 {
            proof!{ reveal(f_spec); }
            x / 2
        }

        verus!{
        proof fn lemma_f(x: u32)
            ensures f(x) == x / 2,
        {
            reveal(f_spec);
        }
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_dual_spec_opaque_not_revealed code!{
        #[verus_verify(dual_spec, open, opaque)]
        #[verus_spec(returns f(x))]
        pub fn f(x: u32) -> u32 {
            proof!{ reveal(__VERUS_SPEC_f); }
            x / 2
        }

        verus!{
        proof fn lemma_f(x: u32)
            ensures f(x) == x / 2,
        {}
        }
    } => Err(e) => assert_vir_error_msg(e, "postcondition not satisfied")
}

test_verify_one_file! {
    #[test] test_dual_spec_open_not_pub code!{
        #[verus_verify(dual_spec, open)]
        #[verus_spec(returns f(x))]
        fn f(x: u32) -> u32 {
            x / 2
        }
    } => Err(e) => assert_vir_error_msg(e, "function is marked `open` but not marked `pub`")
}

test_verify_one_file! {
    #[test] test_publish_without_dual_spec code!{
        #[verus_verify(open)]
        pub fn f(x: u32) -> u32 {
            x / 2
        }
    } => Err(e) => assert_vir_error_msg(e, "publish modifiers in #[verus_verify(...)] require `dual_spec`")
}

test_verify_one_file! {
    #[test] test_dual_spec_unsupported_body code!{
        #[verus_verify(dual_spec(spec_f))]
        #[verus_spec(
            requires
                *x < 100,
                y < 100,
            returns
                f(x, y),
        )]
        fn f(x: &mut u32, y: u32) -> u32 {
            *x = *x + y;
            *x
        }
    } => Err(e) => assert_vir_error_msg(e, "`when_used_as_spec` not supported for function with a mutable reference parameter")
}

test_verify_one_file! {
    #[test] test_dual_spec_unsupported_trait code!{
        trait X {
            fn f(&self) -> u32;
        }

        impl X for u32 {
            #[verus_verify(dual_spec(spec_f))]
            #[verus_spec(
                ensures
                    self.spec_f(),
            )]
            fn f(&self) -> u32{
                *self
            }
        }

    } => Err(e) => assert_rust_error_msg(e, "method `spec_f` is not a member of trait `X`")
}

test_verify_one_file! {
    #[test] test_proof_decl_via_macro code!{
        macro_rules! inline_proof {
            ($val: expr => $x: ident, $y: ident) => {
                proof_decl!{
                    let tracked $x: int = $val;
                    let ghost $y: int = $val;
                    assert($x == $y);
                }
             }
        }

        #[verus_spec]
        fn test_inline_proof_via_macro() {
            inline_proof!(0 => x, y);
            proof!{
                assert(x == 0);
                assert(y == 0);
            }
        }

    } => Ok(())
}

test_verify_one_file! {
    #[test] basic_test code! {
        use vstd::prelude::*;

        fn testfn() {

            let f =
            #[verus_spec(z: u64 =>
                requires y == 2
                ensures z == 2
            )]
            |y: u64| { y };

            proof!{
                assert(f.requires((2,)));
                assert(!f.ensures((2,),3));
            }

            let t = f(2);
            proof!{
                assert(t == 2);
            }
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_no_ret_fails code! {
        use vstd::prelude::*;

        fn testfn() {

            let f =
            #[verus_spec(
                requires y == 2
            )]
            |y: u64| {  };
        }
    } => Err(e) => assert_any_vir_error_msg(e, "Closure must have a return type")
}

test_verify_one_file! {
    #[test] test_no_ret_ok code! {
        use vstd::prelude::*;

        fn testfn() {

            let f1 =
            #[verus_spec(
                requires y == 2
            )]
            |y: u64| -> () {  };

            let f2 =
            #[verus_spec(ret: () =>
                requires y == 2
            )]
            |y: u64| {  };
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_const_fn_eval_via_proxy code!{
        use vstd::prelude::*;
        #[verus_spec(ret =>
            ensures ret == x
        )]
        #[allow(unused_variables)]
        pub const fn const_fn(x: u64) -> u64 {
            proof!{
                assert(true);
            }
            {
                proof!{assert(true);}
            }
            x
        }

        pub const X: u64 = const_fn(1);
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_const_fn_with_verus_spec_and_then_verus_verify code!{
        use vstd::prelude::*;
        #[verus_spec(ret =>
            ensures ret == x
        )]
        #[verus_verify(rlimit(1))]
        pub const fn const_fn(x: u64) -> u64 {
            proof!{
                assert(true);
            }
            {
                proof!{assert(true);}
            }
            x
        }
    } => Err(e) => assert_any_vir_error_msg(e, "#[verus_verify] attributes should be applied before #[verus_spec].")
}

test_verify_one_file! {
    /// A bad style to mix verus and verus_spec
    /// This is not recommanded, but we allow it for this test
    /// to avoid confusing error messages.
    /// But we will raise a warning message saying #[verus_spec] is used inside verus!
    #[test] test_const_fn_inside_verus_macro_with_warning code!{
        use vstd::prelude::*;
        verus!{
        #[verus_spec(ensures false)]
        pub const fn const_fn(x: u64) -> u64
        {
            x
        }

        #[verus_spec]
        const C: u8 = 8;
        }
    } => Err(e) => assert!(e.warnings.iter().any(|x| x.message.contains("#[verus_spec] is likely used inside a verus! block.")))
}

test_verify_one_file! {
    #[test] test_duplicated_verus_spec code!{
        use vstd::prelude::*;
        verus!{
        #[verus_spec(ret =>
            ensures ret == x
        )]
        #[verus_spec]
        pub fn const_fn(x: u64) -> u64
        {
            x
        }
        }
    } => Err(e) => assert_any_vir_error_msg(e, "Multiple #[verus_spec] attributes are not allowed.")
}

test_verify_one_file! {
    #[test] test_const_fn_with_ghost code!{
        use vstd::prelude::*;
        #[verus_spec(ret =>
            with Ghost(g): Ghost<u64>
        )]
        #[allow(unused_variables)]
        pub const fn const_fn(x: u64) -> u64 {
            proof!{
                assert(true);
            }
            {
                proof!{assert(true);}
            }
            x
        }

        #[verus_spec(
            with Ghost(g): Ghost<u64>
        )]
        pub const fn call_const_fn(x: u64) -> u64 {
            proof_with!{Ghost(g)}
            const_fn(x)
        }


        // external call to const_fn does not need ghost var.
        pub const X: u64 = const_fn(1);
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_const_fn_with_ghost_direct_call code!{
        use vstd::prelude::*;
        #[verus_spec(with Ghost(g): Ghost<u8>)]
        pub const fn f(x: u8) -> u8 { x }

        #[verus_spec]
        pub fn direct() {
            let _r = f(3); // FAILS
        }
    } => Err(err) => assert_one_fails(err)
}

test_verify_one_file! {
    #[test] test_proof_with_callee_in_const_fn_compiled_typing code!{
        use vstd::prelude::*;
        // The proof block makes `v: u8` in `_VERUS_VERIFIED_f`, but `v: i32` in the compiled `f`
        #[verus_spec(r => with Ghost(g): Ghost<u8>)]
        pub const fn f() -> usize {
            let v = 0;
            proof!{
                let x: u8 = v;
                let _y: u8 = g@; // The executable copy must erase this ghost input reference
            }
            if !v > 0 { 1 } else { 2 }
        }

        #[verus_spec]
        pub const fn g() -> usize {
            proof_with!{Ghost(0u8)}
            f()
        }

        #[verus_spec(ensures true)]
        pub const K: usize = g();

        // Const evaluation of `K` runs the compiled `f`, which returns 2
        #[verus_spec(r => ensures r == 2)]
        pub fn check() -> usize {
            let a = [0u8; K];
            a.len()
        }
    } => Ok(())
}

test_verify_crates! {
    #[test] test_proof_with_callee_in_const_fn_ghost_outputs_cross_crate [
        "a" => code! {
            use vstd::prelude::*;

            #[verus_spec(r =>
                with Ghost(g): Ghost<u8> -> out: (Ghost<u8>, Ghost<u8>)
                ensures r == 2, out.0@ == g, out.1@ == g
            )]
            pub const fn f() -> usize {
                proof_with!{|= (Ghost(g), Ghost(g))}
                2
            }
        },
        "b" => code! {
            use vstd::prelude::*;

            #[verus_spec(r => ensures r == 2)]
            pub const fn g() -> usize {
                // Ignoring outputs exercises erasure of the tuple returned by the verified call
                { proof_with!{Ghost(0u8)} a::f(); }
                proof_with!{Ghost(5u8) => (Ghost(o1), Ghost(o2))}
                let r = a::f();
                proof!{ assert(o1 == 5 && o2 == 5); }
                r
            }

            #[verus_spec(ensures true)]
            pub const K: usize = g();

            #[verus_spec(r => ensures r == 2)]
            pub fn check() -> usize {
                let a = [0u8; K];
                a.len()
            }
        },
    ] => Ok(())
}

test_verify_one_file! {
    #[test] test_impl_method code!{
        use vstd::prelude::*;

        pub struct Foo;

        #[verus_verify]
        impl Foo {
            #[verus_spec(ret =>
                with
                    Tracked(c): Tracked<&mut ()>
                requires
                    true,
                ensures
                    ret == 1,
            )]
            fn test(a: u64, b: u64) -> u64 {
                1
            }
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_no_verus_verify_attribute_on_impl_block_fails code!{
        use vstd::prelude::*;

        pub struct Foo;

        impl Foo {
            #[verus_verify]
            #[verus_spec(ret =>
                with
                    Tracked(c): Tracked<&mut ()>
                requires
                    true,
                ensures
                    ret == 1,
            )]
            fn test(a: u64, b: u64) -> u64 {
                1
            }
        }
    } => Err(_) => {}
}

test_verify_one_file! {
    #[test] test_unverified_in_impl code!{
        use vstd::prelude::*;

        pub struct X;

        #[verus_verify]
        impl X {
            fn unverified() {}
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_item_const_dual code!{
        use vstd::prelude::*;

        #[verus_spec]
        const CONST_ITEM: u64 = 42;

        #[verus_spec]
        fn test() {
            let v = CONST_ITEM;
            proof! {
                assert(v == 42);
                assert(CONST_ITEM == 42);
            }
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_item_const_error code!{
        use vstd::prelude::*;

        const CONST_ITEM: u64 = 42;

        #[verus_spec]
        fn test() {
            let v = CONST_ITEM;
            proof! {
                assert(v == 42);
            }
        }
    } => Err(e) => assert_any_vir_error_msg(e, "cannot use function `test_crate::CONST_ITEM` which is ignored")
}

test_verify_one_file! {
    #[test] test_item_const_ensures code!{
        use vstd::prelude::*;

        #[verus_spec(ret=>
            ensures ret == 42
        )]
        const fn const_item_fn() -> u64 {
            42
        }

        #[verus_spec(
            ensures CONST_ITEM == 42
        )]
        const CONST_ITEM: u64 = const_item_fn();

        #[verus_spec]
        fn test() {
            let v = CONST_ITEM;
            proof! {
                assert(v == 42);
            }
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_item_const_ensures_is_exec_mode code!{
        use vstd::prelude::*;

        #[verus_spec(ret=>
            ensures ret == 42
        )]
        const fn const_item_fn() -> u64 {
            42
        }

        #[verus_spec(
            ensures CONST_ITEM == 42
        )]
        const CONST_ITEM: u64 = const_item_fn();

        #[verus_spec]
        fn test() {
            proof! {
                assert(CONST_ITEM == 42);
            }
        }
    } => Err(e) => assert_any_vir_error_msg(e, "cannot read const with mode exec")
}

test_verify_one_file! {
    #[test] test_impl_item_const_dual code!{
        use vstd::prelude::*;

        struct X;

        #[verus_verify]
        impl X {
            const A: usize = 1;
            const B: usize = Self::A + 1;
            const C: usize = Self::B + 1;
        }

        #[verus_verify]
        trait HasValue {
            const VALUE: usize;
        }

        #[verus_verify]
        impl HasValue for u8 {
            const VALUE: usize = 3 + 4;
        }

         #[verus_spec(ensures true)]
        fn test() {
            let v = X::C;
            let trait_value = <u8 as HasValue>::VALUE;
            proof! {
                assert(v == 3);
                assert(X::A == 1);
                assert(X::B == 2);
                assert(X::C == 3);
                assert(trait_value == 7);
            }
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_impl_item_const_requires_erasure code!{
        use vstd::prelude::*;

        struct X;

        #[verus_verify]
        impl X {
            const A: usize = 1;
        }

        // This requires to use const_proxy in const.
        #[verus_verify]
        enum Y {
            A,
            B = (1 << X::A) - 1,
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_impl_item_const_use_unverified code!{
        use vstd::prelude::*;

        struct X;

        const fn const_fn() -> u64 {
            42
        }

        #[verus_verify]
        impl X {
            const CONST_ITEM: u64 = const_fn();
        }
    } => Err(e) => assert_any_vir_error_msg(e, "cannot use function `test_crate::const_fn` which is ignored")
}

test_verify_one_file! {
    #[test] test_impl_item_const_external code!{
        use vstd::prelude::*;

        struct X;

        const fn const_fn() -> u64 {
            42
        }

        #[verus_verify]
        impl X {
            #[verus_verify(external)]
            const CONST_ITEM: u64 = const_fn();
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_impl_item_const_ensures code!{
        use vstd::prelude::*;

        struct X;

        #[verus_spec(ret=>
            ensures ret == 42
        )]
        const fn const_fn() -> u64 {
            42
        }

        #[verus_verify]
        impl X {
            #[verus_spec(ensures Self::CONST_ITEM != 42)]
            const CONST_ITEM: u64 = const_fn();
        }
    } => Err(e) => assert_any_vir_error_msg(e, "postcondition not satisfied")
}

test_verify_one_file! {
    #[test] test_verus_spec_with_trailing_comma_simple code! {
        use vstd::prelude::*;

        #[verus_spec(ret =>
            with
                Tracked(y): Tracked<&mut u32>,
                Ghost(w): Ghost<u32>,
            requires
                x < 100,
            ensures
        )]
        fn foo(x: u32) -> u32 {
            (x + 1)
        }

        #[verus_spec(ret =>
            with
                Tracked(y): Tracked<&mut u32>,
            requires
                x < 100,
        )]
        fn bar(x: u32) -> u32 {
            (x + 1)
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_verus_spec_with_trailing_comma_complex code!{
        use vstd::prelude::*;

        #[verus_spec(ret =>
            with
                Tracked(y): Tracked<&mut u32>,
                Ghost(w): Ghost<u32>,
                    -> z: Ghost<u32>,
            requires
                x < 100,
            ensures
                ret == x + 1,
                z@ == x,
            )]
        fn foo(x: u32) -> u32 {
            proof_with!(|= Ghost(x));
            (x + 1)
        }

        #[verus_spec(ret =>
            with
                Tracked(y): Tracked<&mut u32>,
                Ghost(w): Ghost<u32>
                    -> z: Ghost<u32>,
            requires
                x < 100,
            ensures
                ret == x + 1,
                z@ == x,
        )]
        fn bar(x: u32) -> u32 {
            proof_with!(|= Ghost(x));
            (x + 1)
        }

        #[verus_spec(ret =>
            with
                Tracked(y): Tracked<&mut u32>,
                Ghost(w): Ghost<u32>,
                    -> z: Ghost<u32>
            requires
                x < 100,
            ensures
                ret == x + 1,
                z@ == x,
        )]
        fn baz(x: u32) -> u32 {
            proof_with!(|= Ghost(x));
            (x + 1)
        }

        #[verus_spec(ret =>
            with
                Tracked(y): Tracked<&mut u32>,
                Ghost(w): Ghost<u32>
                    -> z: Ghost<u32>
            requires
                x < 100,
            ensures
                ret == x + 1,
                z@ == x,
        )]
        fn qux(x: u32) -> u32 {
            proof_with!(|= Ghost(x));
            (x + 1)
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_verus_verify_external_type_specification code!{
        #[verus_verify(external)]
        struct MyExtStruct<T> { t: T }

        #[verus_verify(external_type_specification)]
        struct ExMyExtStruct<U>(MyExtStruct<U>);

        verus! {
        fn test_ext_type_spec() {
            let s = MyExtStruct::<u64> { t: 5 };
            assert(s.t == 5);
        }
        } // verus!
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_verus_verify_ext_equal code!{
        #[verus_verify(ext_equal)]
        struct MyStruct {
            x: u32,
            y: u64
        }

        verus! {
        fn test_ext_equal(s1: MyStruct, s2: MyStruct)
            requires
                s1.x == s2.x,
                s1.y == s2.y,
            ensures
                s1 == s2,
        {
            assert(s1 =~= s2);
        }
        } // verus!
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_verus_verify_reject_recursive_types code!{
        #[verus_verify(reject_recursive_types(T))]
        enum X<T> {
            ZZ(T),
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_verus_verify_external_type_spec_with_reject_recursive code!{
        #[verus_verify(external)]
        struct MyExtBody<T> { t: T }

        #[verus_verify(external_type_specification, external_body, reject_recursive_types(U))]
        struct ExMyExtBody<U>(MyExtBody<U>);
    } => Ok(())
}

test_verify_one_file! {
    // Check that a postcondition failure in #[verus_spec] points at the specific
    // failing ensures clause, not at the `ensures` keyword.
    #[test] test_verus_spec_ensures_span_on_failure code!{
        #[verus_spec(ret =>
            ensures
                ret > 0,
                ret < 0, // FAILS
        )]
        fn returns_one() -> i8 {
            1
        }
    } => Err(err) => assert_one_fails(err)
}

test_verify_one_file! {
    #[test] test_erase_unverified_code code!{
        use vstd::prelude::*;
        #[verus_spec(
            with Tracked(x): Tracked<()>,
            ensures true,
        )]
        fn foo() {
            proof!{
                let abcd = Tracked(x);
                let y = x;
            }
            #[cfg_attr(not(customized_cfg), verus_spec(
                invariant x == (),
            ))]
            for i in 0..10 {
            }
        }
    } => Ok(())
}

// test forloop without verus_spec invariant, which should be allowed with exec_allows_no_decreases_clause
test_verify_one_file! {
    #[test] test_for_loop_without_loop_spec code!{
        use vstd::prelude::*;
        #[verus_spec]
        fn test_for_loop()
        {
            for i in 0..10
            {
            }
        }
    } => Ok(())
}

// test forloop without verus_spec on either function or loop
test_verify_one_file! {
    #[test] test_verus_verify_on_func_for_loop code!{
        use vstd::prelude::*;
        #[verus_verify]
        fn test_for_loop()
        {
            for i in 0..10
            {
            }
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_skip_desugar_loop_with_external_body code!{
        #[verus_verify]
        struct A;

        impl Iterator for A {
            type Item = u32;
            fn next(&mut self) -> Option<Self::Item> {
                None
            }
        }

        #[verus_verify(external_body)]
        #[verus_spec(ensures false)]
        fn test_for_loop()
        {
            fn child() {
                for _ in 0..1 {}
            }
            const C: fn() = || {
                for _ in 0..1 {}
            };
            static S: fn() = || {
                for _ in 0..1 {}
            };

            let a = A;
            for i in a
            {
            }
            child();
            C();
            S();
        }

        #[verus_spec]
        fn checked_parent() {
            struct Local;

            #[verifier::external]
            impl Local {
                fn ignored() {
                    for _ in 0..1 {}
                }
            }

            impl Local {
                #[verus_spec(r => ensures r == 7)]
                fn seven() -> u64 {
                    7
                }
            }

            let seven = Local::seven();
            proof! {
                assert(seven == 7);
            }
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_proof_with_struct code!{
        use vstd::prelude::*;
        use vstd::raw_ptr::PointsToRaw;

        #[verus_verify]
        pub struct STest {
            pub u: u32,
            #[cfg(verus_keep_ghost_body)]
            pub p: Tracked<PointsToRaw>,
        }

        // Test with trailing comma in struct literal
        #[verus_spec(result =>
            with
                Tracked(p): Tracked<PointsToRaw>,
            ensures
                result.u == u,
        )]
        pub fn make_s_test_trailing(u: u32) -> STest
        {
            proof_with!{ p: Tracked(p) }
            STest {
                u,
            }
        }

        // Test without trailing comma in struct literal
        #[verus_spec(result =>
            with
                Tracked(p): Tracked<PointsToRaw>,
            ensures
                result.u == u,
        )]
        pub fn make_s_test_no_trailing(u: u32) -> STest
        {
            proof_with!{ p: Tracked(p) }
            STest { u }
        }

        // Test with let binding
        #[verus_spec(result =>
            with
                Tracked(p): Tracked<PointsToRaw>,
            ensures
                result.u == u,
        )]
        pub fn make_s_test_let(u: u32) -> STest
        {
            proof_with!{ p: Tracked(p) }
            let s = STest { u };
            s
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_proof_with_struct_and_follows code!{
        use vstd::prelude::*;

        #[verus_verify]
        pub struct TestStruct {
            pub x: u32,
            #[cfg(verus_keep_ghost_body)]
            pub y: Ghost<int>,
        }

        // Test combined struct fields and |= follow output in a single proof_with!
        #[verus_spec(s =>
            with
                -> g: Ghost<int>
            requires
                x < u32::MAX,
            ensures
                s.x == x + 1,
                s.y == 2 * s.x,
                g == 3 * x,
        )]
        fn make_struct_with_follows(x: u32) -> TestStruct
        {
            proof_decl! {
                let ghost g = 3 * x;
            }
            proof_with! {
                y: Ghost(2 * (x + 1)),
                |= Ghost(g)
            }
            TestStruct{
                x: x + 1,
            }
        }

        // Test combined struct fields and |= follow output with let binding
        #[verus_spec(s =>
            with
                -> g: Ghost<int>
            requires
                x < u32::MAX,
            ensures
                s.x == x + 1,
                s.y == 2 * s.x,
                g == 3 * x,
        )]
        fn make_struct_with_follows_let(x: u32) -> TestStruct
        {
            proof_decl! {
                let ghost g = 3 * x;
            }
            proof_with! { y: Ghost(2 * (x + 1))}
            let s = TestStruct{
                x: x + 1,
            };
            proof_with! { |= Ghost(g) }
            s
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_proof_invalid_ghost_with_struct code!{
        use vstd::prelude::*;

        #[verus_verify]
        pub struct TestStruct {
            pub x: u32,
            #[cfg(verus_keep_ghost_body)]
            pub y: int,
        }

        #[verus_spec]
        fn make_struct_with_follows_let(x: u32) -> TestStruct
        {
            proof_with! { y: 1 }
            let s = TestStruct{
                x: x + 1,
            };
        }
    } => Err(e) => assert_any_vir_error_msg(e, "A ghost/tracked field must be a tracked/ghost expression" )
}

test_verify_one_file! {
    // Regression test for https://github.com/verus-lang/verus/issues/2283
    // proof_with! with only |= follow output before a struct constructor
    // should not attempt to parse the follow as struct fields.
    #[test] test_proof_with_follows_only_before_struct code!{
        use vstd::prelude::*;

        #[verus_verify]
        pub struct S {}

        #[verus_spec(
            with
                -> res: Ghost<int>
            ensures
                res == 0,
        )]
        pub fn f() -> S {
            proof_with!(|= Ghost(0int));
            S {}
        }
    } => Ok(())
}

test_verify_one_file! {
    // Test that `|` in a struct field expression inside proof_with! doesn't
    // get confused with `|=` follow syntax. Uses bitwise OR on integers.
    #[test] test_proof_with_struct_field_with_bitor code!{
        use vstd::prelude::*;

        #[verus_verify]
        pub struct TestInt {
            pub x: u32,
            #[cfg(verus_keep_ghost_body)]
            pub g: Ghost<u32>,
        }

        #[verus_spec(s =>
            requires
                x < u32::MAX,
            ensures
                s.x == x + 1,
                s.g == (x | 1u32),
        )]
        fn make_with_bitor(x: u32) -> TestInt
        {
            proof_with! { g: Ghost(x | 1u32) }
            TestInt {
                x: x + 1,
            }
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_verus_verify_on_const code! {
        #[verus_verify]
        const MY_CONST1: u64 = 1u64;

        #[verus_verify(external_body)]
        #[verus_spec(
            ensures MY_CONST2 == 1
        )]
        const MY_CONST2: u64 = 0;

        #[verus_verify]
        #[verus_spec]
        const MY_CONST3: u64 = 0;

        #[verus_verify]
        #[cfg_attr(not(customized_cfg), verus_spec(
            ensures MY_CONST4 == 0
        ))]
        const MY_CONST4: u64 = 0;

        #[verus_spec]
        fn test_use_const() {
            let x = MY_CONST1;
            let y = MY_CONST2;
            let z = MY_CONST3;
            let w = MY_CONST4;
            proof!{
                assert(x == 1);
                assert(y == 1);
                assert(z == 0);
                assert(w == 0);
            }
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_verus_verify_on_static code! {
        #[verus_verify]
        static MY_STATIC1: u64 = 1u64;

        #[verus_verify]
        #[verus_spec]
        static MY_STATIC3: u64 = 0;

        #[verus_verify]
        #[cfg_attr(not(customized_cfg), verus_spec(
            ensures MY_STATIC4 == 0
        ))]
        static MY_STATIC4: u64 = 0;

        #[verus_spec]
        fn test_use_static() {
            let x = MY_STATIC1;
            let y = MY_STATIC3;
            let z = MY_STATIC4;
            proof!{
                assert(u64::MIN <= x <= u64::MAX);
                assert(u64::MIN <= y <= u64::MAX);
                assert(z == 0);
            }
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] test_verus_verify_on_static_failed code! {
        #[verus_spec]
        static MY_STATIC1: u64 = 0;

        #[verus_spec(ensures
            MY_STATIC2 == 1 // FAILS
        )]
        static MY_STATIC2: u64 = 0;

        #[verus_spec]
        fn test_use_static_failed() {
            let x = MY_STATIC1;
            proof!{
                assert(x == 0); // FAILS
            }
        }


    } => Err(e) => assert_fails(e, 2)
}

test_verify_one_file! {
    #[test] test_verus_verify_on_static_with_external_body code! {
        #[verus_verify(external_body)]
        static MY_STATIC2: u64 = 0;
    } => Err(e) => assert_any_vir_error_msg(e, "`#[verifier::external_body]` doesn't make sense for this item type -- it is only applicable to functions and datatype declarations" )
}

test_verify_one_file! {
    // specifications gated on a cfg (`cfg_attr(verus_only, verus_spec(..))`, for code also built
    // without Verus) inside an impl marked #[verus_verify]: the impl must not add a second
    // #[verus_spec] to the method or its loops
    #[test] test_cfg_attr_verus_spec_in_impl code!{
        #[verus_verify]
        struct S { x: u64 }

        #[verus_verify]
        impl S {
            #[cfg_attr(verus_only, verus_spec(r =>
                requires self.x < 100,
                ensures r == self.x + 1,
            ))]
            fn inc(&self) -> u64 {
                let mut i: u64 = 0;
                #[cfg_attr(verus_only, verus_spec(invariant i <= self.x, decreases self.x - i))]
                while i < self.x { i += 1; }
                i + 1
            }

            #[cfg_attr(verus_only, verus_spec(r => ensures r == 2))] // FAILS
            fn two(&self) -> u64 {
                3
            }
        }
    } => Err(err) => assert_one_fails(err)
}

test_verify_one_file! {
    // as above, but the cfg_attr predicate does not hold (`any()`): the item still needs the
    // empty #[verus_spec] that #[verus_verify] adds, here for its `for` loop
    #[test] test_cfg_attr_verus_spec_in_impl_pred_false code!{
        use vstd::prelude::*;
        #[verus_verify]
        struct S { x: u64 }

        #[verus_verify]
        impl S {
            #[cfg_attr(any(), verus_spec(r => ensures r == 0))]
            fn count(&self) -> u64 {
                let mut n: u64 = 0;
                for i in 0..10u64 {
                    n = i;
                }
                n
            }
        }
    } => Ok(())
}

test_verify_one_file! {
    #[test] verus_verify_external_body_with_spec_with_decreases code! {
        struct X { }

        #[verus_verify]
        impl X {
            #[verus_verify(external_body)]
            #[verus_spec(ret =>
                requires true,
                ensures true,
                decreases 255 - i,
            )]
            fn test(i: u8) -> bool {
                proof!{
                    let x = 5int;
                }

                if i < 255 {
                    Self::test(i + 1)
                } else {
                    true
                }
            }
        }
    } => Ok(())
}
```
{% endraw %}

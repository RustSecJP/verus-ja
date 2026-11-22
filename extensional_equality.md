{% raw %}
# 外延性による等価性 {#equality-via-extensionality}

[仕様ライブラリ](spec_lib.html)の節では、`Seq`、`Set`、`Map` のようなコレクション型が同値であることを検査するための、外延的等価性（extensional equality）の演算子 `=~=` を紹介しました。
外延的等価性は、2 つのコレクションが等しい要素を含むことを証明することで、それらのコレクションが等しいことを証明します。
たとえばこれを使って、列 `s1`、`s2`、`s3` が、異なる方法で作られたにもかかわらず、同じ要素（0, 10, 20, 30, 40）を持つので等しいことを証明できます。

```rust
proof fn check_eq(x: Seq<int>, y: Seq<int>)
    requires
        x == y,
{
}

proof fn test_eq2() {
    let s1: Seq<int> = seq![0, 10, 20, 30, 40];
    let s2: Seq<int> = seq![0, 10] + seq![20] + seq![30, 40];
    let s3: Seq<int> = Seq::new(5, |i: int| 10 * i);
    assert(s1 =~= s2);
    assert(s1 =~= s3);
    check_eq(s1, s2); // succeeds
    check_eq(s1, s3); // succeeds
}
```

`assert(s1 =~= s2)` のような表明は、外延性によって等価性を証明するときによく使われます。
なお、Verus は既定で、`assert`、`ensures`、`invariant` の中の `==` を `=~=` に格上げします。そのため、たとえば `assert(s1 == s2)` は、実際には `assert(s1 =~= s2)` を意味します。
これは、外延的等価性がもっと大きな証明の中の小さな一歩にすぎず、等価性についての低レベルな細部で証明を散らかしたくない、という多くの場合に便利です。
外延的等価性が鍵となる手順である証明では、記録として残す意味で、`assert(s1 =~= s2)` と明示的に書くとよいでしょう。

（たとえば、外延的等価性が本当に必要なのはどこなのかを正確に知りたいなどの理由で、Verus に `==` を `=~=` へ自動的に格上げさせたくない場合は、属性 `#[verifier::auto_ext_equal(...)]` と `#![verifier::auto_ext_equal(...)]` で Verus の既定の振る舞いを上書きできます。
例は [ext_equal.rs](https://github.com/verus-lang/verus/blob/main/source/rust_verify_test/tests/ext_equal.rs) を参照してください。）

## 構造体と列挙型の外延的等価性 {#extensional-equality-for-structs-and-enums}

`struct` または `enum` のデータ型が `Seq`、`Set`、`Map` を含むフィールドを持っていて、そのデータ型の 2 つの値が等しいことを証明したいとしましょう。
これは、各フィールドに個別に `=~=` を使えばできます。

```rust
    struct Foo {
        a: Seq<int>,
        b: Set<int>,
    }

    proof fn ext_equal_struct() {
        let f1 = Foo { a: seq![1, 2, 3], b: set!{4, 5, 6} };
        let f2 = Foo { a: seq![1, 2].push(3), b: set!{5, 6}.insert(4) };
        // assert(f1 == f2);    // FAILS -- need to use =~= first
        assert(f1.a =~= f2.a);  // succeeds
        assert(f1.b =~= f2.b);  // succeeds
        assert(f1 == f2);  // succeeds, now that we've used =~= on .a and .b
    }
```

しかし、同値であることを検査するたびに各フィールドに `=~=` を使うのは、かなり面倒です。
これを助けるために、Verus は `#[verifier::ext_equal]` 属性に対応しています。この属性は、`Seq`、`Set`、`Map`、`Multiset`、`spec_fn` のフィールドや、ほかの `#[verifier::ext_equal]` データ型のフィールドに対して外延性が必要なデータ型に印を付けます。
たとえば次のとおりです。

```rust
#[verifier::ext_equal]  // necessary for invoking =~= on the struct
struct Foo {
    a: Seq<int>,
    b: Set<int>,
}

proof fn ext_equal_struct() {
    let f1 = Foo { a: seq![1, 2, 3], b: set!{4, 5, 6} };
    let f2 = Foo { a: seq![1, 2].push(3), b: set!{5, 6}.insert(4) };
    assert(f1 =~= f2);  // succeeds
}
```

（注: `#[verifier::ext_equal]` を付けても、`==` の意味は変わりません。データ型についての `==` を証明するのに、`=~=` を使いやすくするだけです。）

## 深い外延的等価性 {#deep-extensional-equality}

列や集合のようなコレクションのデータ型は、ほかのコレクションのデータ型を要素として含むことができます（たとえば、列の列や、列の集合）。
演算子 `=~=` は、最上位のコレクションにだけ外延性を適用し、コレクションの入れ子になった要素には適用しません。
要素にも外延性を適用するために、Verus は「深い」外延的等価性の演算子 `=~~=` を用意しています。この演算子は、コレクション、`spec_fn`、データ型の任意の入れ子を扱います。
たとえば次のとおりです。

```rust
proof fn ext_equal_nested() {
    let inner: Set<int> = set!{1, 2, 3};
    let s1: Seq<Set<int>> = seq![inner];
    let s2 = s1.update(0, s1[0].insert(1));
    let s3 = s1.update(0, s1[0].insert(2).insert(3));
    // assert(s2 =~= s3); // FAILS
    assert(s2 =~~= s3);  // succeeds
    let s4: Seq<Seq<Set<int>>> = seq![s1];
    let s5: Seq<Seq<Set<int>>> = seq![s2];
    assert(s4 =~~= s5);  // succeeds
}
```

次のように、`spec_fn` にも同じことが当てはまります。

```rust
#[verifier::ext_equal]  // necessary for invoking =~= on the struct
struct Bar {
    a: spec_fn(int) -> int,
}

proof fn ext_equal_fnspec(n: int) {
    // basic case
    let f1 = (|i: int| i + 1);
    let f2 = (|i: int| 1 + i);
    assert(f1 =~= f2);  // succeeds
    // struct case
    let b1 = Bar { a: |i: int| if i == 1 { i } else { 1 } };
    let b2 = Bar { a: |i: int| 1int };
    assert(b1 =~= b2);  // succeeds
    // nested case
    let i1 = (|i: int| i + 2);
    let i2 = (|i: int| 2 + i);
    let n1: Seq<spec_fn(int) -> int> = seq![i1];
    let n2: Seq<spec_fn(int) -> int> = seq![i2];
    // assert(n1 =~= n2); // FAILS
    assert(n1 =~~= n2);  // succeeds
}
```
{% endraw %}

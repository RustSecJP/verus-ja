{% raw %}
# 等価性 {#equality}

等価性は、ゴーストコードと実行可能コードとで、違った振る舞いをします。
実行可能コードでは、Rust は `==` を、`PartialEq` トレイトの `eq` 関数の呼び出しを意味するものと定義しています。

```rust
fn equal1(x: u8, y: u8) {
    let eq1 = x == y;  // means x.eq(y) in Rust
    let eq2 = y == x;  // means y.eq(x) in Rust
    assert(eq1 ==> eq2);  // succeeds
}
```

`u8` のような組み込みの整数型では、`x.eq(y)` 関数は期待どおりに定義されていて、`x` と `y` が同じ整数を保持していれば `true` を返します。
しかし、利用者が定義した型では、`eq` は別の振る舞いをするかもしれません。
たとえ、その型が Rust の [`Eq` トレイト](https://doc.rust-lang.org/std/cmp/trait.Eq.html)を実装していても、副作用を持ったり、非決定的に振る舞ったり、同値関係を実装するという約束を果たさなかったりするかもしれません。

```rust
fn equal2<A: Eq>(x: A, y: A) {
    let eq1 = x == y; // means x.eq(y) in Rust
    let eq2 = y == x; // means y.eq(x) in Rust
    assert(eq1 ==> eq2); // won't work; we can't be sure that A is an equivalence relation
}
```

これに対して、ゴーストコードでは、`==` 演算子は常に同値関係です
（つまり、反射的、対称的、推移的です）。

```rust
fn equal3(x: u8, y: u8) {
    assert({
        let eq1 = x == y;
        let eq2 = y == x;
        eq1 ==> eq2
    });
}
```

Verus は、ゴーストコードの `==` が、次の場合に真になると定義しています。
- 2 つの整数または真偽値については、値が等しいとき
- 2 つの構造体または列挙型については、型が同じで、フィールドが等しいとき
- 2 つの `&` 参照、2 つの Box の値、2 つの Rc の値、2 つの Arc の値については、指している先の値が同じとき
- 2 つの RefCell の値、または 2 つの Cell の値については、内部のデータへのポインターが等しいとき（内部の中身が等しいときではありません）

これに加えて、`Seq<T>`、`Set<T>`、`Map<Key, Value>` のようなコレクションのデータ型は、独自の `==` の定義を持っています。
2 つの列、2 つの集合、2 つのマップは、その要素が等しければ等しくなります。
[仕様ライブラリ](spec_lib.html)と[外延的等価性](extensional_equality.html)で詳しく説明するように、2 つの列、2 つの集合、2 つのマップの間の等価性を証明するには、「外延的等価性」（extensional equality）の演算子 `=~=` の助けが必要になることがあります。
{% endraw %}

{% raw %}
# 型不変条件で整形式性をカプセル化する {#encapsulating-well-formedness-with-type-invariants}

前の章の仕様を思い出してください。

```rust
impl<V> TreeMap<V> {
    pub fn new() -> (tree_map: Self)
        ensures
            tree_map.well_formed(),
            tree_map@ == Map::<u64, V>::empty(),

    pub fn insert(&mut self, key: u64, value: V)
        requires
            old(self).well_formed(),
        ensures
            final(self).well_formed(),
            final(self)@ == old(self)@.insert(key, value),

    pub fn delete(&mut self, key: u64)
        requires
            old(self).well_formed(),
        ensures
            final(self).well_formed(),
            final(self)@ == old(self)@.remove(key),

    pub fn get(&self, key: u64) -> Option<&V>
        requires
            self.well_formed(),
        returns
            (if self@.dom().contains(key) { Some(&self@[key]) } else { None }),

    pub fn get_mut(&mut self, key: u64) -> (ret: Option<&mut V>)
        requires
            self.well_formed(),
        ensures
            (match ret {
                Some(r) =>
                    old(self)@.dom().contains(key)
                      && *r == old(self)@[key]
                      && final(self).well_formed()
                      && final(self).as_map() == old(self)@.insert(key, *final(r)),
                None =>
                    !old(self)@.dom().contains(key)
                      && *final(self) == *old(self)
            })
}
```

この `tree_map.well_formed()` という述語があること、特に `requires` 節にあることに注目してください。その結果、クライアントは、自分のコードのいたるところで、この `tree_map.well_formed()` 述語を扱わなければなりません。たとえば次のとおりです。

```rust
fn test2() {
    let mut tree_map = TreeMap::<bool>::new();
    test_callee(tree_map);
}

fn test_callee(tree_map: TreeMap<bool>)
    requires
        tree_map.well_formed(),
{
    let mut tree_map = tree_map;
    tree_map.insert(25, true);
    tree_map.insert(100, true);
}
```

`requires` 節が無ければ、上のコード片は検証に失敗します。

しかし直感的には、そもそもなぜこの述語を持ち回らなければならないのか、と疑問に思うかもしれません。なにしろ、カプセル化のおかげで、クライアントが `well_formed()` の成り立た _ない_ `tree_map` を作ることは、決してできないのです。

この節では、Verus の[型不変条件（type invariant）](./reference-type-invariants.html)の機能を使って、クライアントからこの負担を取り除く方法を示します。

### `type_invariant` 属性を適用する。 {#applying-the-type_invariant-attribute}

`well_formed()` 述語を自動的に推論してほしいと Verus に伝えるには、この述語に `#[verifier::type_invariant]` 属性を付けます。

```rust
impl<V> TreeMap<V> {
    #[verifier::type_invariant]
    spec fn well_formed(self) -> bool {
        match self.root {
            Some(node) => node.well_formed(),
            None => true, // empty tree always well-formed
        }
    }
}
```

これには 2 つの効果があります。

 * *すべての* `TreeMap` オブジェクトが常に `well_formed()` の条件を満たす、という追加の制約を加えます。この制約は、`TreeMap` オブジェクトが構築されたり変更されたりするたびに、Verus によって検査されます。
 * プログラマーが、`requires` 節に書かれていないときでも、常に `well_formed()` の条件を _仮定_ できるようにします。

`type_invariant` 属性を加えたことに加えて、`well_formed` から **`pub` 指定子も取り除いた** ことに注意してください。これで、クライアントからは本体が見えないだけでなく、名前さえも見えなくなりました。そもそも私たちの意図は、クライアントがこれについて推論しなくて済むようにすることです。そうであれば、これを公開のインターフェースを通じて見せる理由はまったくありません。

もちろん、これを可能にするには、`TreeMap` のさまざまな `pub` メソッドの仕様を更新する必要があります。

### コードを更新する: `new` {#updating-the-code-new}

簡単なものから始めましょう。`new` です。

```rust
impl<V> TreeMap<V> {
    pub fn new() -> (s: Self)
        ensures
            s@ == Map::<u64, V>::empty()
    {
        TreeMap::<V> { root: None }
    }
}
```

ここで行ったのは、`s.well_formed()` という事後条件を取り除くことだけです。先に述べたとおり、これはもう必要ありません。

重要なのは、Verus が依然として、`s.well_formed()` が成り立つことの _証明_ を要求するということです。具体的には、`well_formed` に `#[verifier::type_invariant]` が付いているので、Verus は、`TreeMap` のコンストラクタが戻るときに `well_formed()` が成り立つことを検査します。以前と同じように、Verus はこの条件をごく簡単に検査できます。

### コードを更新する: `get` {#updating-the-code-get}

次に `get` を見てみましょう。最初に気づくのは、`requires self.well_formed()` 節を取り除いていることです。

```rust
impl<V> TreeMap<V> {
    pub fn get(&self, key: u64) -> Option<&V>
        returns
            (if self@.dom().contains(key) { Some(&self@[key]) } else { None }),
    {
        proof { use_type_invariant(&*self); }
        Node::<V>::get_from_optional(&self.root, key)
    }
}
```

事前条件がもう無いのに、`self.well_formed()` を _どうやって_ 導くのでしょうか（これは、`self.root` が整形式であることを証明して `Node::get_from_optional` を呼び出すために必要です）。

これは、組み込みの疑似補題 `use_type_invariant` で行えます。この機能は、どんなオブジェクトに対して呼び出しても、渡されたオブジェクトがその型不変条件を満たすことを保証します。

### コードを更新する: `insert` {#updating-the-code-insert}

次に `TreeMap::insert` を確認しましょう。思い出してほしいのですが、これは木を変更しなければなりません。

```rust
impl<V> TreeMap<V> {
    pub fn insert(&mut self, key: u64, value: V)
        ensures
            final(self)@ == old(self)@.insert(key, value),
    {
        proof { use_type_invariant(&*self); }
        let mut root = None;
        std::mem::swap(&mut root, &mut self.root);
        Node::<V>::insert_into_optional(&mut root, key, value);
        self.root = root;
    }
}
```

以前と同じように、`use_type_invariant` を使って、`requires` 節が無くても、関数の冒頭で `self.well_formed()` が成り立つことを示します。

`#[verifier::type_invariant]` を使うことで生じるちょっとした難しさは、型不変条件が _すべての_ プログラム地点で成り立つことを強制される点です。このため、途中の計算が少しやりにくくなることがあります。

この場合、これを回避する簡単な方法は、`root` フィールドを `None` と [`swap`](https://doc.rust-lang.org/std/mem/fn.swap.html) し、終わったら元に戻すことです。空の `TreeMap` は整形式性を自明に満たすので、これは許されます。

入れ替えをせずに、単に `Node::<V>::insert_into_optional(&mut self.root, key, value)` としてはなぜいけないのか、と思うかもしれません。この書き方の問題は、`insert_into_optional` の呼び出しが「巻き戻し安全（unwind-safe）」であること、つまり、パニックが起きて `insert_into_optional` が早期に終了しなければならなくなっても、すべての型不変条件が保たれることを、保証しなければならなくなる点です。現時点で、Verus が巻き戻し安全性を保証する方法は 1 つしかありません。それは、巻き戻し（unwind）がまったく起きないことを力ずくで保証する、というものです。したがって、理想的な解決策は、`insert_into_optional` に [`no_unwind`](./reference-unwind-sig.html) を付けることでしょう。しかし、この場合はそれができません。ノードの挿入が `Box::new` を呼び出すからです。

この問題と、巻き戻し安全性に関する Verus の現在の制限とを合わせて考えると、それを避けて通る方法として、`swap` を使うやり方が最も簡単な解決策になります。`type_invariant` 機能の制限について詳しくは、[リファレンスのページ](./reference-type-invariants.html)を確認してください。

### コードを更新する: `delete` {#updating-the-code-delete}

これは `insert` とほとんど同じです。

```rust
impl<V> TreeMap<V> {
    pub fn delete(&mut self, key: u64)
        ensures
            final(self)@ == old(self)@.remove(key),
    {
        proof { use_type_invariant(&*self); }
        let mut root = None;
        std::mem::swap(&mut root, &mut self.root);
        Node::<V>::delete_from_optional(&mut root, key);
        self.root = root;
    }
}
```

### コードを更新する: `get_mut` {#updating-the-code-get_mut}

こちらはもう少し興味深いものです。更新後のマップの具体的な値は、呼び出しの終わりには分からないからです。返された可変参照が解決されて初めて分かります。重要なのは、型不変条件がマップの中の _値_ に依存しないということです。そのため、クライアントが返された可変参照を使って型不変条件を破る方法はありません。言い換えると、型不変条件は、可変参照のありうる最終的な値の _すべて_ について成り立ちます。

```rust
impl<V> Node<V> {
    fn get_mut_from_optional(node: &mut Option<Box<Node<V>>>, key: u64) -> (ret: Option<&mut V>)
        requires
            node.is_some() ==> node.unwrap().well_formed(),
        ensures
            (match ret {
                Some(r) =>
                    old(node).is_some()
                      && old(node).unwrap().as_map().dom().contains(key)
                      && *r == old(node).unwrap().as_map()[key]
                      && final(node).is_some()
                      && final(node).unwrap().well_formed()
                      && final(node).unwrap().as_map() == old(node).unwrap().as_map().insert(key, *final(r))
                      && Node::optional_as_map(*final(node)).dom() =~= Node::optional_as_map(*old(node)).dom(),
                None =>
                    (old(node).is_some() ==> !old(node).unwrap().as_map().dom().contains(key))
                      && *final(node) == *old(node)
            }),
        decreases *node,
        no_unwind
    {
        match node {
            None => None,
            Some(node) => {
                node.get_mut(key)
            }
        }
    }

    fn get_mut(&mut self, key: u64) -> (ret: Option<&mut V>)
        requires
            self.well_formed(),
        ensures
            (match ret {
                Some(r) =>
                    old(self).as_map().dom().contains(key)
                      && *r == old(self).as_map()[key]
                      && final(self).well_formed()
                      && final(self).as_map() == old(self).as_map().insert(key, *final(r))
                      && final(self).as_map().dom() =~= old(self).as_map().dom(),
                None =>
                    !old(self).as_map().dom().contains(key)
                      && *final(self) == *old(self)
            })
        decreases *self,
        no_unwind
    {
        if key == self.key {
            Some(&mut self.value)
        } else if key < self.key {
            proof { assert(!Node::<V>::optional_as_map(self.right).dom().contains(key)); }
            Self::get_mut_from_optional(&mut self.left, key)
        } else {
            proof { assert(!Node::<V>::optional_as_map(self.left).dom().contains(key)); }
            Self::get_mut_from_optional(&mut self.right, key)
        }
    }
}

impl<V> TreeMap<V> {
    pub fn get_mut(&mut self, key: u64) -> (ret: Option<&mut V>)
        ensures
            (match ret {
                Some(r) =>
                    old(self)@.dom().contains(key)
                      && *r == old(self)@[key]
                      && final(self).as_map() == old(self)@.insert(key, *final(r)),
                None =>
                    !old(self)@.dom().contains(key)
                      && *final(self) == *old(self)
            })
    {
        proof { use_type_invariant(&*self); }
        Node::<V>::get_mut_from_optional(&mut self.root, key)
    }
}
```

### 新しいシグネチャと仕様 {#the-new-signatures-and-specifications}

すべてをまとめると、公開 API の仕様は次のようになります。

```rust
impl<V> TreeMap<V> {
    pub fn new() -> (s: Self)
        ensures
            s@ == Map::<u64, V>::empty()

    pub fn insert(&mut self, key: u64, value: V)
        ensures
            final(self)@ == old(self)@.insert(key, value),

    pub fn delete(&mut self, key: u64)
        ensures
            final(self)@ == old(self)@.remove(key),

    pub fn get(&self, key: u64) -> Option<&V>
        returns
            (if self@.dom().contains(key) { Some(&self@[key]) } else { None }),

    pub fn get_mut(&mut self, key: u64) -> (ret: Option<&mut V>)
        ensures
            (match ret {
                Some(r) =>
                    old(self)@.dom().contains(key)
                      && *r == old(self)@[key]
                      && final(self).as_map() == old(self)@.insert(key, *final(r)),
                None =>
                    !old(self)@.dom().contains(key)
                      && *final(self) == *old(self)
            })
}
```

これらは、以前のものとほとんど同じです。唯一の違いは、`well_formed()` の節がすべて取り除かれていることです。

都合のよいことに、`requires` 節はもう _1 つも_ 無いので、これらの関数はどれも、いつでも呼び出せます。これは、API が Rust の意味で「安全」であることを証明したい場合にも重要です（[このページ](./memory-safety.html)を参照してください）。

### 新しいクライアントのコード {#the-new-client-code}

以前と同じように、クライアントのコードは、`TreeMap` をただの [`Map`](https://verus-lang.github.io/verus/verusdoc/vstd/map/struct.Map.html) であるかのように推論できます。ただし今回は、`tree_map.well_formed()` について推論しなくてよいので、少し単純になっています。

```rust
fn test() {
    let mut tree_map = TreeMap::<bool>::new();
    tree_map.insert(17, false);
    tree_map.insert(18, false);
    tree_map.insert(17, true);

    assert(tree_map@ == map![17u64 => true, 18u64 => false]);

    tree_map.delete(17);

    assert(tree_map@ == map![18u64 => false]);

    let elem17 = tree_map.get(17);
    let elem18 = tree_map.get(18);
    assert(elem17.is_none());
    assert(elem18 == Some(&false));

    *tree_map.get_mut(18).unwrap() = true;
    assert(tree_map@ == map![18u64 => true]);

    test2(tree_map);
}

fn test2(tree_map: TreeMap<bool>) {
    let mut tree_map = tree_map;
    tree_map.insert(25, true);
    tree_map.insert(100, true);
}
```

## ソースの全体 {#full-source}

この例のソースの全体は、[こちら](./container_bst_all_source.html#version-with-type-invariants)にあります。
{% endraw %}

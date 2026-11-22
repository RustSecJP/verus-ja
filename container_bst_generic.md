{% raw %}
# ジェネリックにする {#making-it-generic}

これまでの節では、固定のキーの型（`u64`）を使う `TreeMap<V>` を作りました。この節では、キーの型 `K` についてジェネリックな `TreeMap<K, V>` の作り方を示します。

## 「全順序」を定義する {#defining-a-total-order}

これが難しい主な理由は、BST が、`K` の値を _比較する_ 方法を必要とするからです。等しいかどうかの比較と、順序を得るための比較の両方です。この比較は、実装の中（指定されたキーのノードを見つけるため、あるいはそのようなノードをどこに挿入すべきかを決めるため）でも、BST の順序の性質を強制する整形式性の不変条件の中でも使われます。

[「全順序」](https://en.wikipedia.org/wiki/Total_order)の概念は、トレイトを作ることでジェネリックに定義できます。

```rust
pub enum Cmp { Less, Equal, Greater }

pub trait TotalOrdered : Sized {
    spec fn le(self, other: Self) -> bool;

    proof fn reflexive(x: Self)
        ensures Self::le(x, x);

    proof fn transitive(x: Self, y: Self, z: Self)
        requires Self::le(x, y), Self::le(y, z),
        ensures Self::le(x, z);

    proof fn antisymmetric(x: Self, y: Self)
        requires Self::le(x, y), Self::le(y, x),
        ensures x == y;

    proof fn total(x: Self, y: Self)
        ensures Self::le(x, y) || Self::le(y, x);

    fn compare(&self, other: &Self) -> (c: Cmp)
        ensures (match c {
            Cmp::Less => self.le(*other) && self != other,
            Cmp::Equal => self == other,
            Cmp::Greater => other.le(*self) && self != other,
        }),
        no_unwind;
}
```

このトレイトは、次のことを同時に行います。

 * 二項関係 `le` が存在することを要求する
 * それが全順序の性質を満たすことを要求する
 * `executable`（実行可能）な三方比較の関数が存在することを要求する

ここでは 1 つ単純化をしています。比較の関数における「等しさ」が [spec の等価性](./equality.html)と同じである、と仮定しているのです。これは常に適切とは限りません。データ型によっては、同じ論理的な値を表す方法が 2 つ以上あるからです。より一般的な仕様であれば、任意の同値関係を尊重する順序を許すでしょう。たとえば [`vstd::hash_map::HashMapWithView`](https://verus-lang.github.io/verus/verusdoc/vstd/hash_map/struct.HashMapWithView.html) は、そのように動きます。しかしこの実演では、話を単純にするために、spec の等価性を尊重する全順序を使います。

### 構造体と定義を更新する {#updating-the-struct-and-definitions}

まず、構造体を、ジェネリックなパラメータ `K: TotalOrdered` を取るように更新します。

```rust
struct Node<K: TotalOrdered, V> {
    key: K,
    value: V,
    left: Option<Box<Node<K, V>>>,
    right: Option<Box<Node<K, V>>>,
}

pub struct TreeMap<K: TotalOrdered, V> {
    root: Option<Box<Node<K, V>>>,
}
```

整形式性の条件も、整数の `<=` の代わりにジェネリックな `K::le` を使うように更新します。元の定義で `a < b` を使っていたところでは、今度は `a.le(b) && a != b` を使います。

```rust
impl<K: TotalOrdered, V> Node<K, V> {
    pub closed spec fn well_formed(self) -> bool
        decreases self
    {
        &&& (forall |elem| #[trigger] Node::<K, V>::optional_as_map(self.left).dom().contains(elem) ==> elem.le(self.key) && elem != self.key)
        &&& (forall |elem| #[trigger] Node::<K, V>::optional_as_map(self.right).dom().contains(elem) ==> self.key.le(elem) && elem != self.key)
        &&& (match self.left {
            Some(left_node) => left_node.well_formed(),
            None => true,
        })
        &&& (match self.right {
            Some(right_node) => right_node.well_formed(),
            None => true,
        })
    }
}

impl<K: TotalOrdered, V> TreeMap<K, V> {
    #[verifier::type_invariant]
    spec fn well_formed(self) -> bool {
        match self.root {
            Some(node) => node.well_formed(),
            None => true, // empty tree always well-formed
        }
    }
}
```

一方、`as_map` の定義は順序の関数に依存しないので、以前のまま、手を加えずにおけます。

### 実装と証明を更新する {#updating-the-implementations-and-proofs}

実装の更新には、もう少し手間がかかります。より本格的な証明コードが必要になるからです。Verus は整数の不等式（`<`）についてはよい自動化を備えていますが、新しく手作りした `TotalOrdered` トレイトについては、そのような自動化がありません。そのため、その性質を手で呼び出す証明コードを加える必要があります。

`Node::get` を見てみましょう。

証明の核心は、おおよそ次のように進みます。

探しているキー `key` が、比較すると `self.key` より小さいとします。このとき、左部分木へ再帰すれば正しい答えが得られることを示す必要があります。そのためには、`key` が右部分木に _無い_ ことを示せば十分です。

（背理法のために）`key` が右部分木に _ある_ と仮定します。すると（整形式性の不変条件によって）`key > self.key` でなければなりません。しかし、`key < self.key` であることはすでに示しました。矛盾です。（形式的には、この矛盾は反対称性を呼び出すことで得られます。）

```rust
impl<K: TotalOrdered, V> Node<K, V> {
    fn get_from_optional(node: &Option<Box<Node<K, V>>>, key: K) -> Option<&V>
        requires
            node.is_some() ==> node.unwrap().well_formed(),
        returns
            (match node {
                Some(node) => (if node.as_map().dom().contains(key) { Some(&node.as_map()[key]) } else { None }),
                None => None,
            }),
        decreases node,
    {
        match node {
            None => None,
            Some(node) => {
                node.get(key)
            }
        }
    }

    fn get(&self, key: K) -> Option<&V>
        requires
            self.well_formed(),
        returns
            (if self.as_map().dom().contains(key) { Some(&self.as_map()[key]) } else { None }),
        decreases self,
    {
        match key.compare(&self.key) {
            Cmp::Equal => {
                Some(&self.value)
            }
            Cmp::Less => {
                proof {
                    if Node::<K, V>::optional_as_map(self.right).dom().contains(key) {
                        TotalOrdered::antisymmetric(self.key, key);
                        assert(false);
                    }
                    assert(key != self.key);
                    assert((match self.left {
                            Some(node) => (if node.as_map().dom().contains(key) { Some(&node.as_map()[key]) } else { None }),
                            None => None,
                        }) == (if self.as_map().dom().contains(key) { Some(&self.as_map()[key]) } else { None }));
                }
                Self::get_from_optional(&self.left, key)
            }
            Cmp::Greater => {
                proof {
                    if Node::<K, V>::optional_as_map(self.left).dom().contains(key) {
                        TotalOrdered::antisymmetric(self.key, key);
                        assert(false);
                    }
                    assert(key != self.key);
                    assert((match self.right {
                            Some(node) => (if node.as_map().dom().contains(key) { Some(&node.as_map()[key]) } else { None }),
                            None => None,
                        }) == (if self.as_map().dom().contains(key) { Some(&self.as_map()[key]) } else { None }));
                }
                Self::get_from_optional(&self.right, key)
            }
        }
    }
}
```

`insert`、`delete`、`get_mut` も同じように更新できます。必要なところに、全順序の性質を呼び出す補題の呼び出しを手で挿入します。

## ソースの全体 {#full-source}

この例のソースの全体は、[こちら](./container_bst_all_source.html#version-with-generic-key-type-and-clone-implementation)にあります。
{% endraw %}

{% raw %}
# 単純な二分探索木 {#a-simple-binary-search-tree}

この節では、二分探索木（Binary Search Tree、BST）を実装して検証します。

データ構造の研究では、二分探索木を
[平衡](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree)
[させる](https://en.wikipedia.org/wiki/AVL_tree)
[方法が](https://en.wikipedia.org/wiki/Treap)
[数多く](https://en.wikipedia.org/wiki/Splay_tree)
[知られて](https://en.wikipedia.org/wiki/B-tree)
います。話を単純にするために、ここではそのどれも実装しません。その代わりに、素直な _平衡でない_ 二分探索木を実装します。より効率のよい設計に改良することは、練習問題として残しておきます。

また、実装の最初の草案では、順序付けのできる固定の型 `u64` のキーを、型 `V` の値に対応付けます。後の章では、キーもジェネリックにして、任意の型 `K` と `V` について `K` を `V` に対応付けるように変更します。

## 実装 {#the-implementation}

### 構造体 {#the-structs}

まず、木の形そのものを定義します。木は、それぞれのノードに (キー, 値) の組を 1 つ持ちます。「葉ノード」と「内部ノード」は区別しません。その代わり、どのノードも、あってもなくてもよい左の子と、あってもなくてもよい右の子を持ちます。さらに、木がまったく空のこともあり、その場合は根がありません。

```rust
struct Node<V> {
    key: u64,
    value: V,
    left: Option<Box<Node<V>>>,
    right: Option<Box<Node<V>>>,
}

pub struct TreeMap<V> {
    root: Option<Box<Node<V>>>,
}
```

`pub` が付いているのは `TreeMap` だけであることに注意してください。そのフィールド `root` と、`Node` 型の全体は、実装の詳細であり、そのためモジュールに対して非公開です。

### 抽象的なビュー {#the-abstract-view}

新しいデータ構造を作るとき、通常は最初に行う重要な手順が 2 つあります。

 * データ構造を、仕様を書くために使う何らかの抽象データ型として解釈する方法を定める。
 * データ構造の整形式性（well-formedness）の不変条件を定める。

まず 1 つ目から行います（1 つには、それが実際に 2 つ目の助けになるからです）。この場合は、データ構造を [`Map<u64, V>`](https://verus-lang.github.io/verus/verusdoc/vstd/map/struct.Map.html) として解釈したいと考えます。そのような関数は、再帰的に定義できます。

```rust
impl<V> Node<V> {
    spec fn optional_as_map(node_opt: Option<Box<Node<V>>>) -> Map<u64, V>
        decreases node_opt,
    {
        match node_opt {
            None => Map::empty(),
            Some(node) => node.as_map(),
        }
    }

    spec fn as_map(self) -> Map<u64, V>
        decreases self,
    {
        Node::<V>::optional_as_map(self.left)
          .union_prefer_right(Node::<V>::optional_as_map(self.right))
          .insert(self.key, self.value)
    }
}

impl<V> TreeMap<V> {
    pub closed spec fn as_map(self) -> Map<u64, V> {
        Node::<V>::optional_as_map(self.root)
    }
}
```

ここでも、`pub` が付いているのは `TreeMap::as_map` だけであり、さらにそれが `closed` になっていることに注意してください。`as_map` の定義は、やはり実装の詳細です。

便利のために、[`View` トレイト](https://verus-lang.github.io/verus/verusdoc/vstd/view/trait.View.html)も実装しておくのが慣例です。こうすると、クライアントは `@` 記法を使ってマップの実装を参照できます。たとえば、`tree_map.view()` の省略形として `tree_map@` と書けます。ここでは、仕様を `tree_map.view()` を使って書いていきます。

```rust
impl<V> View for TreeMap<V> {
    type V = Map<u64, V>;

    open spec fn view(&self) -> Map<u64, V> {
        self.as_map()
    }
}
```

### 整形式性を定める {#establishing-well-formedness}

次に、整形式性を定めます。これは結局、BST の順序の性質を保つことです。つまり、どのノード _N_ についても、_N_ の左部分木にあるノードは _N_ より小さいキーを持ち、_N_ の右部分木にあるノードは _N_ より大きいキーを持つ、という性質です。これもやはり、再帰的な `spec` 関数で定義できます。

```rust
impl<V> Node<V> {
    spec fn well_formed(self) -> bool
        decreases self,
    {
        &&& (forall |elem| Node::<V>::optional_as_map(self.left).dom().contains(elem) ==> elem < self.key)
        &&& (forall |elem| Node::<V>::optional_as_map(self.right).dom().contains(elem) ==> elem > self.key)
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

impl<V> TreeMap<V> {
    pub closed spec fn well_formed(self) -> bool {
        match self.root {
            Some(node) => node.well_formed(),
            None => true, // empty tree always well-formed
        }
    }
}
```

### コンストラクタを実装する: `TreeMap::new()` {#implementing-a-constructor-treemapnew}

コンストラクタの定義は単純です。根の無い空の木を作ります。仕様は、返されるオブジェクトが _空の_ マップを表さなければならないことを示しています。

```rust
impl<V> TreeMap<V> {
    pub fn new() -> (tree_map: Self)
        ensures
            tree_map.well_formed(),
            tree_map@ == Map::<u64, V>::empty(),
    {
        TreeMap::<V> { root: None }
    }
}
```

`tree_map@` は `tree_map.as_map()` と等価であることを思い出してください。`tree_map.as_map()` と `Node::optional_as_map()` の定義を見れば、`root` が `None` のときにこれが空のマップになることは明らかなはずです。

この仕様が木の内部にまったく触れていないことに、あらためて注目してください。述べているのは、木が整形式であることと、その抽象的なビューが空のマップであることだけです。

### `insert` を実装する {#implementing-insert}

`insert` も、再帰的な走査を使って実装できます。指定されたノードを探し、その際に整形式性の条件を使って、正しいことをしていると証明します。この走査の中で、正しいキーを持つノードを見つけるか、目的のノードを見つけないまま葉に到達するかのどちらかになります。前者の場合は `value` を更新し、後者の場合は新しいノードを作ります。

（余談: このコードは、`&mut Option<T>` から `&mut T` を借用する代わりに、[`Option::take`](https://doc.rust-lang.org/std/option/enum.Option.html#method.take) を使ってノードの所有権を得ています。木の中への可変参照は、[後で](#implementing-get_mut)見ます。）

```rust
impl<V> Node<V> {
    fn insert_into_optional(node: &mut Option<Box<Node<V>>>, key: u64, value: V)
        requires
            old(node).is_some() ==> old(node).unwrap().well_formed(),
        ensures
            final(node).is_some() ==> final(node).unwrap().well_formed(),
            Node::<V>::optional_as_map(*final(node)) =~= Node::<V>::optional_as_map(*old(node)).insert(key, value),
        decreases *old(node),
    {
        match node.take() {
            None => {
                *node = Some(Box::new(Node::<V> {
                    key: key,
                    value: value,
                    left: None,
                    right: None,
                }));
            }
            Some(mut boxed_node) => {
                (&mut *boxed_node).insert(key, value);
                *node = Some(boxed_node);
            }
        }
    }

    fn insert(&mut self, key: u64, value: V)
        requires
            old(self).well_formed(),
        ensures
            final(self).well_formed(),
            final(self).as_map() =~= old(self).as_map().insert(key, value),
        decreases *old(self),
    {
        if key == self.key {
            self.value = value;

            assert(!Node::<V>::optional_as_map(self.left).dom().contains(key));
            assert(!Node::<V>::optional_as_map(self.right).dom().contains(key));
        } else if key < self.key {
            Self::insert_into_optional(&mut self.left, key, value);

            assert(!Node::<V>::optional_as_map(self.right).dom().contains(key));
        } else {
            Self::insert_into_optional(&mut self.right, key, value);

            assert(!Node::<V>::optional_as_map(self.left).dom().contains(key));
        }
    }
}

impl<V> TreeMap<V> {
    pub fn insert(&mut self, key: u64, value: V)
        requires
            old(self).well_formed(),
        ensures
            final(self).well_formed(),
            final(self)@ == old(self)@.insert(key, value),
    {
        Node::<V>::insert_into_optional(&mut self.root, key, value);
    }
}
```

`TreeMap::insert` の仕様が、[`Map::insert`](https://verus-lang.github.io/verus/verusdoc/vstd/map/struct.Map.html#method.insert) を使って与えられていることに注目してください。

### `delete` を実装する {#implementing-delete}

`delete` の実装は少し難しくなります。内部ノードを取り除く必要がある場合、木の形を少し組み替えなければならないことがあるからです。とはいえ、特定の平衡の戦略に従おうとしているわけではないので、それほど大変ではありません。

```rust
impl<V> Node<V> {
    fn delete_from_optional(node: &mut Option<Box<Node<V>>>, key: u64)
        requires
            old(node).is_some() ==> old(node).unwrap().well_formed(),
        ensures
            final(node).is_some() ==> final(node).unwrap().well_formed(),
            Node::<V>::optional_as_map(*final(node)) =~= Node::<V>::optional_as_map(*old(node)).remove(key),
        decreases *old(node),
    {
        if let Some(mut boxed_node) = node.take() {

            if key == boxed_node.key {
                assert(!Node::<V>::optional_as_map(boxed_node.left).dom().contains(key));
                assert(!Node::<V>::optional_as_map(boxed_node.right).dom().contains(key));

                if boxed_node.left.is_none() {
                    *node = boxed_node.right;
                } else {
                    if boxed_node.right.is_none() {
                        *node = boxed_node.left;
                    } else {
                        let (popped_key, popped_value) = Node::<V>::delete_rightmost(&mut boxed_node.left);
                        boxed_node.key = popped_key;
                        boxed_node.value = popped_value;
                        *node = Some(boxed_node);
                    }
                }
            } else if key < boxed_node.key {
                assert(!Node::<V>::optional_as_map(boxed_node.right).dom().contains(key));
                Node::<V>::delete_from_optional(&mut boxed_node.left, key);
                *node = Some(boxed_node);
            } else {
                assert(!Node::<V>::optional_as_map(boxed_node.left).dom().contains(key));
                Node::<V>::delete_from_optional(&mut boxed_node.right, key);
                *node = Some(boxed_node);
            }
        }
    }

    fn delete_rightmost(node: &mut Option<Box<Node<V>>>) -> (popped: (u64, V))
        requires
            old(node).is_some(),
            old(node).unwrap().well_formed(),
        ensures
            final(node).is_some() ==> final(node).unwrap().well_formed(),
            Node::<V>::optional_as_map(*final(node)) =~= Node::<V>::optional_as_map(*old(node)).remove(popped.0),
            Node::<V>::optional_as_map(*old(node)).dom().contains(popped.0),
            Node::<V>::optional_as_map(*old(node))[popped.0] == popped.1,
            forall |elem| Node::<V>::optional_as_map(*old(node)).dom().contains(elem) ==> popped.0 >= elem,
        decreases *old(node),
    {
        let mut boxed_node = node.take().unwrap();

        if boxed_node.right.is_none() {
            *node = boxed_node.left;
            assert(Node::<V>::optional_as_map(boxed_node.right) =~= Map::empty());
            assert(!Node::<V>::optional_as_map(boxed_node.left).dom().contains(boxed_node.key));
            return (boxed_node.key, boxed_node.value);
        } else {
            let (popped_key, popped_value) = Node::<V>::delete_rightmost(&mut boxed_node.right);
            assert(!Node::<V>::optional_as_map(boxed_node.left).dom().contains(popped_key));
            *node = Some(boxed_node);
            return (popped_key, popped_value);
        }
    }
}

impl<V> TreeMap<V> {
    pub fn delete(&mut self, key: u64)
        requires
            old(self).well_formed(),
        ensures
            final(self).well_formed(),
            final(self)@ == old(self)@.remove(key),
    {
        Node::<V>::delete_from_optional(&mut self.root, key);
    }
}
```

`TreeMap::delete` の仕様が、[`Map::remove`](https://verus-lang.github.io/verus/verusdoc/vstd/map/struct.Map.html#method.remove) を使って与えられていることに注目してください。

### `get` を実装する {#implementing-get}

次に、`TreeMap::get` を実装します。この関数はキーを検索し、`Option<&V>` を返します（キーが `TreeMap` に無ければ `None`）。何も更新しないので、これは比較的素直です。

```rust
impl<V> Node<V> {
    fn get_from_optional(node: &Option<Box<Node<V>>>, key: u64) -> Option<&V>
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

    fn get(&self, key: u64) -> Option<&V>
        requires
            self.well_formed(),
        returns
            (if self.as_map().dom().contains(key) { Some(&self.as_map()[key]) } else { None }),
        decreases self,
    {
        if key == self.key {
            Some(&self.value)
        } else if key < self.key {
            proof { assert(!Node::<V>::optional_as_map(self.right).dom().contains(key)); }
            Self::get_from_optional(&self.left, key)
        } else {
            proof { assert(!Node::<V>::optional_as_map(self.left).dom().contains(key)); }
            Self::get_from_optional(&self.right, key)
        }
    }
}

impl<V> TreeMap<V> {
    pub fn get(&self, key: u64) -> Option<&V>
        requires
            self.well_formed(),
        returns
            (if self@.dom().contains(key) { Some(&self@[key]) } else { None }),
    {
        Node::<V>::get_from_optional(&self.root, key)
    }
}
```

### `get_mut` を実装する {#implementing-get_mut}

最後に、`TreeMap::get_mut` を実装して検証します。

```rust
fn get_mut(&mut self, key: u64) -> (ret: Option<&mut V>);
```

これは `TreeMap::get` によく似ていますが、共有参照の代わりに可変参照を受け渡す点が違います。おそらく最も興味深いのは、仕様です。

[可変借用を返す関数をどう仕様化するか](./mutable-references.html#returning-mutable-borrows)を思い出してください。一般に、次のパターンがうまくいきます。

 * 出力の参照の初期値（`*ret.unwrap()`）は、入力の参照の初期値（`*old(self)`）の関数であるべきです。
 * 入力の参照の最終的な値（`*final(self)`）は、出力の参照の最終的な値（`*final(ret.unwrap())`）の関数であるべきです。

もちろん、`*ret.unwrap()` は（`get` と同じように）マップへの添字アクセスにすぎません。一方、`*final(self)` は、新しい値に基づくマップの更新になります。

```rust
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
```

補助の関数も、すべて同じように仕様化します。全体は次のとおりです。

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
    {
        if key == self.key {
            return Some(&mut self.value);
        } else if key < self.key {
            proof { assert(!Node::<V>::optional_as_map(self.right).dom().contains(key)); }
            return Self::get_mut_from_optional(&mut self.left, key);
        } else {
            proof { assert(!Node::<V>::optional_as_map(self.left).dom().contains(key)); }
            return Self::get_mut_from_optional(&mut self.right, key);
        }
    }
}

impl<V> TreeMap<V> {
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
    {
        Node::<V>::get_mut_from_optional(&mut self.root, key)
    }
}
```

### クライアントとして `TreeMap` を使う {#using-the-treemap-as-a-client}

短いクライアントのプログラムで、`TreeMap` を [`Map`](https://verus-lang.github.io/verus/verusdoc/vstd/map/struct.Map.html) であるかのように推論できることを示します。

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
}
```

## ソースの全体 {#full-source}

この例のソースの全体は、[こちら](./container_bst_all_source.html#first-draft)にあります。
{% endraw %}

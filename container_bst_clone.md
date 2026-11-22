{% raw %}
# Clone の実装 {#implementing-clone}

仕上げとして、`TreeMap<K, V>` に `Clone` を実装しましょう。
ここでの要点は、`TreeMap::<K, V>::clone` の正しい仕様（specification）を見つけ出すことです。

当然ながら、このような実装には `K: Clone` と `V: Clone` の両方が必要です。
しかし、木に対して筋の通った clone の実装を書くには、`K::clone` と `V::clone` の実装が実際に何をするのかを考えなければなりません。

一般に、Verus は `Clone` の実装に何の制約も課しません。
そのため、`clone()` の呼び出しが、入力と仕様上等しい（spec-equal）値を返すとは限りません。

これを踏まえ、この例を簡単にするために、`TreeMap<K, V>::clone` について次のシグネチャを証明することにします。

```rust
impl<K: Copy + TotalOrdered, V: Clone> Clone for TreeMap<K, V> {
    fn clone(&self) -> (res: Self)
        ensures self@.dom() =~= res@.dom(),
            forall |key| #[trigger] res@.dom().contains(key) ==>
                cloned::<V>(self@[key], res@[key]),
    {
        ...
    }
}
```
このシグネチャの詳細を以下で説明します。

### `K::clone` の扱い {#dealing-with-kclone}

すべてのキーを複製するには、`K::clone` が要素の順序を保つ必要があります。
そうでなければ、複製の操作の途中ですべてのキーを並べ替え直して、結果の木が正しい状態になるようにしなければなりません。
しかし、それが望ましい動作であるとは考えにくいでしょう。`K::clone` が `TotalOrdered` の実装に従わないのであれば、それは利用者側のバグである可能性が高いと言えます。

これを扱う一般的な方法は、`Clone` が何らかの意味で全順序と実際に整合していることを要求することでしょう。
しかし、前の節で、すでに「全順序」の仕様を少し簡略化していたことを思い出してください。
ここでも同じように話を簡単にしておくために、`K: Copy` も要求することにします。

その結果、この `TreeMap` の clone の実装は、仕様上の等価性で比べた場合でも、すべてのキーを正確に保てることを証明できます。
つまり、`self@.dom() =~= res@.dom()` を保証できるのです。

### `V::clone` の扱い {#dealing-with-vclone}

では `V` はどうでしょうか。ここでも、`V::clone` が何をするのかは事前には分かりません。
入力と等しくない値を返すかもしれませんし、非決定的でさえあるかもしれません。
したがって、複製した `TreeMap` は、元のものと異なる値を持つ可能性があります。

`TreeMap::<K, V>::clone` をできるだけ汎用的に仕様化するために、その ensures 節を `V::clone` の ensures 節 _を使って_ 書くことにします。
これは [`call_ensures`](./exec_funs_as_values.html) を使えば実現できます。
述語 `call_ensures(V::clone, (&self@[key],), res@[key])` は、実質的に「`self@[key]` と `res@[key]` は、`V::clone` の入力と出力の組としてあり得るものである」と言っています。
この述語は長たらしいので、`vstd` は次の補助関数を用意しています。
<code class="hljs"><a href="https://verus-lang.github.io/verus/verusdoc/vstd/pervasive/fn.cloned.html">cloned::&lt;V&gt;</a>(self@[key], res@[key])</code>

### シグネチャが意味することを理解する {#understanding-the-implications-of-the-signature}

いくつか例を見てみましょう。

まず、`TreeMap::<u64, u32>` を複製する場合を考えます。Verus の標準ライブラリは `u32::clone` の仕様を提供しています。
それはコピーと同じで、複製した `u32` は常に入力と等しくなります。
その結果、`TreeMap::<u64, u32>` を複製すると、その `view` が正確に保たれると導けます。
これは[外延的等価性（extensional equality）](./extensional_equality.html)を使って証明できます。

```rust
fn test_clone_u32(tree_map: TreeMap<u64, u32>) {
    let tree_map2 = tree_map.clone();
    assert(tree_map2@ =~= tree_map@);
}
```

`clone` が仕様上の等価性を保証する型であれば、_どんな_ 型でも同じことができます。
利用者が定義した型を使った例をもう 1 つ示します。

```rust
struct IntWrapper {
    pub int_value: u32,
}

impl Clone for IntWrapper {
    fn clone(&self) -> (s: Self)
        ensures s == *self
    {
        IntWrapper { int_value: self.int_value }
    }
}

fn test_clone_int_wrapper(tree_map: TreeMap<u64, IntWrapper>) {
    let tree_map2 = tree_map.clone();
    assert(tree_map2@ =~= tree_map@);
}
```

これがうまくいくのは、`IntWrapper::clone` の事後条件、つまり `ensures s == *self` があるからです。
この書き方に慣れていないと、`IntWrapper::clone` が `test_clone_int_wrapper` の検証に影響することを、最初は意外に感じるかもしれません。
`test_clone_int_wrapper` は `IntWrapper::clone` を直接呼び出していないからです。
この場合、事後条件は `TreeMap<u64, IntWrapper>::clone` を通じて間接的に参照されています。

もう 1 つ例を見ましょう。今度は、_より精度の低い_ clone 関数を使います。

```rust
pub struct WeirdInt {
    pub int_value: u32,
    pub other: u32,
}

impl Clone for WeirdInt {
    fn clone(&self) -> (s: Self)
        ensures
            s.int_value == self.int_value,
    {
        WeirdInt { int_value: self.int_value, other: 0 }
    }
}

fn test_clone_weird_int(tree_map: TreeMap<u64, WeirdInt>) {
    let tree_map2 = tree_map.clone();

    // assert(tree_map2@ =~= tree_map@); // this would fail

    assert(tree_map2@.dom() == tree_map@.dom());
    assert(forall |k| tree_map@.dom().contains(k) ==>
        tree_map2@[k].int_value == tree_map@[k].int_value);
}
```

この例は少し病的です。構造体 `WeirdInt` には、複製されない余分なフィールドがあります。
このような性質を持つ現実の場面も想像できるでしょう（たとえば、すべての構造体が一意の識別子を持たなければならない場合です）。
ともあれ、`WeirdInt::clone` の事後条件は、2 つのオブジェクトが等しいとは言っておらず、`int_value` フィールドが等しいとだけ言っています。
この事後条件は、示したとおり、マップ内のそれぞれの値について導けます。

### `TreeMap::<K, V>::clone` の実装 {#implementing-treemapk-vclone}

いつものように、実装は再帰関数として書きます。

`Node` に `Clone` を実装することは必須ではありません。代わりに、`TreeMap::clone` の補助として普通の再帰関数を実装するだけでも構いません。
しかし、このやり方のほうが Rust らしい書き方です。
こうすると、`TreeMap::clone` の実装の中で、根である `Option<Box<Node<K, V>>>` に対して `clone` を呼び出せます（`Option::clone` の仕様は vstd が提供しています）。
ただし、回避しなければならない「落とし穴」がいくつかあることが見て取れます。

```rust
impl<K: Copy + TotalOrdered, V: Clone> Clone for Node<K, V> {
    fn clone(&self) -> (res: Self)
        ensures
            self.well_formed() ==> res.well_formed(),
            self.as_map().dom() =~= res.as_map().dom(),
            forall |key| #[trigger] res.as_map().dom().contains(key) ==>
                cloned::<V>(self.as_map()[key], res.as_map()[key]),
        decreases self,
    {
        let res = Node {
            key: self.key,
            value: self.value.clone(),
            // Ordinarily, we would use Option<Node>::clone rather than inlining
            // the case statement here; we write it this way to work around
            // this issue: https://github.com/verus-lang/verus/issues/1346
            left: (match &self.left {
                Some(node) => Some(Box::new((&**node).clone())),
                None => None,
            }),
            right: (match &self.right {
                Some(node) => Some(Box::new((&**node).clone())),
                None => None,
            }),
        };

        proof {
            assert(Node::optional_as_map(res.left).dom() =~= 
                Node::optional_as_map(self.left).dom());
            assert(Node::optional_as_map(res.right).dom() =~= 
                Node::optional_as_map(self.right).dom());
        }

        return res;
    }
}

impl<K: Copy + TotalOrdered, V: Clone> Clone for TreeMap<K, V> {
    fn clone(&self) -> (res: Self)
        ensures self@.dom() =~= res@.dom(),
            forall |key| #[trigger] res@.dom().contains(key) ==>
                cloned::<V>(self@[key], res@[key]),
    {
        proof {
            use_type_invariant(self);
        }

        TreeMap {
            // This calls Option<Node<K, V>>::Clone
            root: self.root.clone(),
        }
    }
}
```

## ソースコード全文 {#full-source}

この例のソースコード全文は[こちら](./container_bst_all_source.html#version-with-generic-key-type-and-clone-implementation)にあります。
{% endraw %}

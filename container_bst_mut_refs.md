{% raw %}
# コンテナの中の可変参照 {#mutable-references-in-a-container}

この章では、`TreeMap<u64, &mut u64>` を使うクライアント側のアプリケーションを見ていきます。
このように可変参照をコンテナに入れることは、あまり一般的ではありませんが、前例が無いわけでもありません。
Verus でこれを行うには、少しだけ追加の知識が必要です。

## `Vec` での寄り道 {#a-detour-through-vec}

`TreeMap` を見る前に、理論の一部をおさらいするために、`Vec` を使った例を見ておきましょう。
ここでは、可変参照のベクタを、3 つのローカル変数の組への「添字アクセス」の手段として使います。

```rust
fn vec_with_mut_refs(i: usize)
    requires 0 <= i < 3
{
    let mut a = 0;
    let mut b = 0;
    let mut c = 0;

    let mut v = vec![&mut a, &mut b, &mut c];

    *v[i] = 1;

    assert(i == 0 ==> (a, b, c) == (1, 0, 0));
    assert(i == 1 ==> (a, b, c) == (0, 1, 0));
    assert(i == 2 ==> (a, b, c) == (0, 0, 1));
}
```

残念ながら、3 つのアサーションは、明らかに真であるにもかかわらず、どれも検証に失敗します。なぜでしょうか。

通常、可変参照がローカル変数に格納されている場合（たとえば `a: &mut u64`）、Verus は `a` への最後の変更を特定し、述語 [`has_resolved(a)`](./mutable-references.html#the-has_resolved-operator-when-mutable-references-are-done) の仮定を挿入します。
これは _解決の仮定（resolution assumption）_ と呼ばれます。更新後の値が借用元の場所に「流れ戻る」ためには、この仮定が必要です。

では、ベクタがある場合はどうなるでしょうか。この場合、ローカル変数は `v: Vec<&mut u64>` なので、Verus は `v` の解決点を決めます。
`*v[i] = 1` の直後が `v` の解決点となり、`has_resolved(v)` が分かります。
このようにコンテナに適用された場合、`has_resolved` は再帰的です。つまり、`v` の _中に_ 含まれるすべての可変参照が解決済みであることを意味します。

これで `v` が解決済みであることが分かりました。では、そこからどうやって `v[0]`、`v[1]`、`v[2]` が解決済みであることを得るのでしょうか。

この場合のために、vstd は公理 [`axiom_vec_has_resolved`](https://verus-lang.github.io/verus/verusdoc/vstd/std_specs/vec/fn.axiom_vec_has_resolved.html) を提供しています。

```rust
pub broadcast axiom fn axiom_vec_has_resolved<T>(vec: Vec<T>, i: int)
    ensures
        0 <= i < vec.len() ==> has_resolved::<Vec<T>>(vec) ==> has_resolved(vec@[i]);
```

したがって、この法則から `has_resolved(v[0])` が導かれ、`a` の更新後の値が得られるはずです。
同様に `has_resolved(v[1])` から `b` の更新後の値が、`has_resolved(v[2])` から `c` の更新後の値が得られるはずです。

上のプログラムを検証に通すには、解決の公理を呼び出すだけで済みます。
これは、公理を直接呼び出すか、単に量化子を[トリガー](./forall.html)することで行えます。

```rust
fn vec_with_mut_refs(i: usize)
    requires 0 <= i < 3
{
    let mut a = 0;
    let mut b = 0;
    let mut c = 0;

    let mut v = vec![&mut a, &mut b, &mut c];

    *v[i] = 1;

    assert(has_resolved(v[0]));
    assert(has_resolved(v[1]));
    assert(has_resolved(v[2]));

    assert(i == 0 ==> (a, b, c) == (1, 0, 0));
    assert(i == 1 ==> (a, b, c) == (0, 1, 0));
    assert(i == 2 ==> (a, b, c) == (0, 0, 1));
}
```

## `TreeMap` のための解決の補題 {#a-resolution-lemma-for-treemap}

それでは、`TreeMap<u64, &mut u64>` について話しましょう。

上の節で学んだことから、可変参照のコンテナを活用するには、何らかの `has_resolved` の法則が必要だと分かります。
したがって、この `TreeMap` ライブラリは、次のような補題を公開するべきです。

```rust
has_resolved(map) ==> map@.dom().contains(k) ==> has_resolved(map@[k])
```

これはどうやって証明するのでしょうか。

Verus は、個々の構造体と列挙型（利用者が定義したものを含みます）について、`has_resolved` に関する公理を自動的に生成します。
たとえば、この `TreeMap` 構造体を見てみましょう。

```rust
pub struct TreeMap<K: TotalOrdered, V> {
    root: Option<Box<Node<K, V>>>,
}
```

Verus は、次の公理を自動的に生成します。

```rust
has_resolved::<TreeMap<K, V>>(map) ==>
    has_resolved::<Option<Box<Node<K, V>>>>(map.root)
```

（少なくとも、フィールドが可視である場合はそうです。）同様に、`Option` についての公理もあり、`Some` の場合に、中に含まれる値の `has_resolved` が得られます。

```rust
has_resolved::<Option<T>>(option) ==>
    (option matches Some(x) ==> has_resolved::<T>(x))
```

そして最後に、Box についての公理です。

```rust
has_resolved::<Box<U>>(box) ==>
    has_resolved::<U>(*box)
```

これらをすべてつなげると、`has_resolved::<TreeMap<K, V>>(map) ==> has_resolved::<Node<K, V>>(*map.root.unwrap())` が得られます。
あとは同じように、`Node` 構造体の中で `left` フィールドと `right` フィールドを再帰的にたどっていけば、値 `V` に到達します。
これらをすべてまとめて、帰納的な証明にするだけです。

```rust
proof fn lemma_node_has_resolved<K: TotalOrdered, V>(node: Node<K, V>, k: K)
    requires
        has_resolved(node),
        node.as_map().dom().contains(k),
    ensures
        has_resolved(node.as_map()[k])
    decreases node,
{
    if node.left.is_some() && node.left.unwrap().as_map().dom().contains(k) {
        lemma_node_has_resolved(*node.left.unwrap(), k);
    }
    if node.right.is_some() && node.right.unwrap().as_map().dom().contains(k) {
        lemma_node_has_resolved(*node.right.unwrap(), k);
    }
}

proof fn lemma_tree_map_has_resolved<K: TotalOrdered, V>(map: TreeMap<K, V>, k: K)
    requires
        has_resolved(map),
        map@.dom().contains(k),
    ensures
        has_resolved(map@[k])
{
    match map.root {
        Some(node) => {
            lemma_node_has_resolved(*node, k);
        }
        None => { }
    }
}
```

それでは、クライアントで試してみましょう。

```rust
fn tree_map_with_mut_refs(i: u64)
    requires 0 <= i < 3
{
    let mut a = 0;
    let mut b = 0;
    let mut c = 0;

    let mut tree_map = TreeMap::<u64, &mut u64>::new();
    tree_map.insert(0, &mut a);
    tree_map.insert(1, &mut b);
    tree_map.insert(2, &mut c);

    **tree_map.get_mut(i).unwrap() = 1;

    proof {
        lemma_tree_map_has_resolved(tree_map, 0);
        lemma_tree_map_has_resolved(tree_map, 1);
        lemma_tree_map_has_resolved(tree_map, 2);
    }

    assert(i == 0 ==> (a, b, c) == (1, 0, 0));
    assert(i == 1 ==> (a, b, c) == (0, 1, 0));
    assert(i == 2 ==> (a, b, c) == (0, 0, 1));
}
```
{% endraw %}

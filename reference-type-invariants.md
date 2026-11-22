{% raw %}
# 型不変条件 {#type-invariants}

構造体と列挙型には、*型不変条件*（type invariant）を付けられます。これは、値が整形式（well-formed）であることを表す真偽値の述語です。型不変条件は、**すべての exec オブジェクトと、tracked モードのゴーストオブジェクト**に適用され、**spec オブジェクトには適用されません**。

型不変条件は、主に*不変条件をカプセル化して隠す*ことを目的としています。

### 型不変条件を宣言する {#declaring-a-type-invariant}

型不変条件は、`#[verifier::type_invariant]` 属性で宣言できます。
最上位の項目として宣言することも、impl ブロックの中で宣言することもできます。

```rust
#[verifier::type_invariant]
spec fn type_inv(x: X) -> bool { ... }
```

```rust
impl X {
    #[verifier::type_invariant]
    spec fn type_inv(self) -> bool { ... }
}
```

`impl` ブロックの中に置いて `self` を取る形にもできますし、最上位の項目として宣言することもできます。
名前は何でもかまいません。

不変条件の述語は、次の条件を満たさなければなりません。

 * 型が `(X) -> bool` または `(&X) -> bool` の spec 関数であること。
   ここで `X` は、不変条件を適用する型です。
 * 次の条件を満たすデータ型（`struct` または `enum`）に適用されていること。
   * 同じクレートの中で宣言されている
   * クレートの外に公開されたフィールドを持たない

型不変条件の関数の*可視性*が、対象の型の可視性と同じでなければならない、という制限はありません。
必要なのは、型不変条件を表明または仮定する必要があるところ（後述）で、その関数が見えていることだけです。
型不変条件はカプセル化を目的としているので、できるだけ非公開にすることをおすすめします。

### 型不変条件が成り立つことを強制する {#enforcing-that-the-type-invariant-holds}

型不変条件を持つどの型 `X` についても、
Verus は、`X` 型のすべての exec オブジェクトと tracked モードのゴーストオブジェクトについて、
その述語が常に成り立つことを強制します。そのため、Verus は、次の箇所に、述語が成り立つという証明義務を追加します。

 * `X` のコンストラクタ式のすべて
 * `X` のフィールドへの代入のあと
 * `X` への可変参照を取る関数呼び出しのあと

現在のところ、型不変条件を「一時的に破る」ことには対応していませんが、この機能は将来追加されるかもしれません。
多くの場合、フィールドへの可変参照を取ることで回避できます。

### 型不変条件を使う {#applying-the-type-invariant}

型不変条件は自動的に強制されますが、利用者に自動的に与えられるわけではありません。
型不変条件を持つどのオブジェクト `x: X` についても、組み込みの疑似補題 `use_type_invariant` を呼び出すと、`x` について型不変条件が成り立つという事実を得られます。

```
use_type_invariant(&x);
```

値 `x` は、tracked または exec の変数でなければなりません。
この文は証明のための機能なので、`exec` 関数の中に現れる場合は、`proof` ブロックの中に置かなければなりません。

### 例 {#example}

```rust
struct X {
    i: u8,
    j: u8,
}

impl X {
    #[verifier::type_invariant]
    spec fn type_inv(self) -> bool {
        self.i <= self.j
    }
}

fn example(x: X) {
    proof {
        use_type_invariant(&x);
    }

    assert(x.i <= x.j); // succeeds
}

fn example_caller() {
    let x = X { i: 20, j: 30 }; // succeeds
    example(x);
}

fn example_caller2() {
    let x = X { i: 30, j: 20 }; // fails
    example(x);
}
```
{% endraw %}

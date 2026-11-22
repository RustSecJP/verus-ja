{% raw %}
# 共用体 {#unions}

Verus は、Rust の共用体（union）に対応しています。

内部的には、Verus は共用体を、列挙型とよく似た形で表します。ただし、共用体にアクセスする Rust の構文は、
列挙型とは違います。Rust では、共用体のフィールドには、フィールドアクセス `u.x` でアクセスします。
Verus は、この操作を exec モードで許し、それが整形式（well-formed）であることを常に検査します。
つまり、`u` が正しい「バリアント」であることを検査します。

spec モードでは、組み込みの spec 演算子 `is_variant` と `get_union_field` を使って、
共用体について推論できます。どちらの演算子も、フィールド名を*文字列リテラル*で指定します。

 * `is_variant(u, "field_name")` は、`u` が整形式のフィールド `"field_name"` を持つとき true を返します。
 * `get_union_field::<U, T>(u, "field_name")` は、`T` 型の値を返します。ここで
    `T` は `"field_name"` の型です。（共用体での型と、演算子のジェネリックパラメータ `T` が
    一致しない場合、Verus はエラーを出します。）

### 例 {#example}

```rust
union U {
    x: u8,
    y: bool,
}

fn union_example() {
    let u = U { x: 3 };

    assert(is_variant(u, "x"));
    assert(get_union_field::<U, u8>(u, "x") == 3); 

    unsafe {
        let j = u.x; // this operation is well-formed
        assert(j == 3); 

        let j = u.y; // Verus rejects this operation
    }   
}   
```

### `unsafe` についての注意 {#note-on-unsafe}

`unsafe` キーワードは、Rust を満足させるために必要です。Rust は、共用体のフィールドアクセスを
unsafe な操作として扱うからです。しかし、Verus では、この操作は安全*です*。
Verus がその事前条件を検査できるからです。[Verus がメモリ安全性をどう扱うかについての詳しい説明](./memory-safety.html)を参照してください。
{% endraw %}

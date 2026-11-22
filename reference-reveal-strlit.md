{% raw %}
# `reveal_strlit` {#reveal_strlit}

### 構文 {#syntax}

```verus-grammar
V@[reveal_strlit_stmt] ::= reveal_strlit ( R@[string_literal] ) ;
```

### 証明上の動作 {#proof-operation}

これは、文字列リテラルの中身を証明器に明かします。中身には、長さと、文字列を構成する文字の列が含まれます。
これを使わない場合、証明器の文脈が多すぎる情報であふれないように、文字列リテラルは不透明なデータとして扱われます。

### 例 {#example}

```rust
fn test() {
    let s = "abc";
    proof { reveal_strlit("abc"); }
    assert(s@[0] == 'a');
}
```
{% endraw %}

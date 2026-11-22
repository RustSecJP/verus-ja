{% raw %}
# 文字列 {#strings}

Verus は、Rust の `String` と `&str` についての推論に対応しています。

```rust
fn get_char() {
    let x = "hello world";
    proof {
        reveal_strlit("hello world");
    }
    assert(x@.len() == 11);
    let val = x.get_char(0);
    assert('h' == val);
}
```

既定では、文字列リテラルは不透明（opaque）なものとして扱われます。

```rust
fn opaque_by_default() {
    let x = "hello world";
    assert(x@.len() == 11); // FAILS
}
```

このコードは、表明の失敗になります。

```
error: assertion failed
  --> ../examples/guide/strings.rs:21:12
   |
21 |     assert(x@.len() == 11); // FAILS
   |            ^^^^^^^^^^^^^^ assertion failed
```

これは、（冒頭の例のように）`reveal_strlit` を使うことで直せます。

ただし、等しいかどうかを比べるだけなら、リテラルを reveal する必要はありません。

```rust
fn literal_eq() {
    let x = "hello world";
    let y = "hello world";
    assert(x@ == y@);
}
```

文字列リテラルは `&str` であり、そのビュー（view）は `Seq<char>` です。

```rust
fn str_view() {
    let x = "hello world";
    let ghost y: Seq<char> = x@;
}
```

その値は、リテラルを reveal するまで分かりません。

もちろん、関数の事前条件（やそのほかの述語）などで、ビューについての情報を指定することはできます。

```rust
fn subrange<'a>(s: &str)
    requires
        s@ =~= "Hello"@,
{
    assert(s@.subrange(0, 1) == "H"@);
}
```

しかしこの場合、それぞれのビューについての情報を得るには、両方のリテラルを reveal する必要があります。

```rust
fn subrange<'a>(s: &str)
    requires s@ =~= "Hello"@,
{
    proof {
        reveal_strlit("Hello");
        reveal_strlit("H");
    }
    assert(s@[..1] =~= "H"@);
}
```

文字列の操作には、現在のところ、`vstd::string` で定義されている操作が必要です。[ドキュメントはこちら](https://verus-lang.github.io/verus/verusdoc/vstd/string/index.html)にあります。

`&str` と `String` にとって重要な述語が `is_ascii` です。これによって、たとえば次のように、効率のよいスライスが可能になります。


```rust
fn test() {
    let a = String::from_str(("ABC"));
    proof {
        reveal_strlit("ABC");
    }
    assert(a.is_ascii());
    let b = a.as_str().substring_ascii(2, 3);
    proof {
        reveal_strlit("C");
    }
    assert(b@ =~= ("C")@);
}

```
{% endraw %}

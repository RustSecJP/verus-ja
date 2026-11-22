{% raw %}
# spec 関数 {#spec-functions}

2 つの整数の最小値を計算する、単純な `spec` 関数から始めましょう。

```rust
spec fn min(x: int, y: int) -> int {
    if x <= y {
        x
    } else {
        y
    }
}

fn test() {
    assert(min(10, 20) == 10); // succeeds
    assert(min(100, 200) == 100); // succeeds
}
```

`exec` 関数とは違い、`spec` 関数の本体は、同じモジュール内のほかの関数から見えます。
そのため、`test` 関数は `min` 関数の中身を見ることができ、`test` の中のアサーションが成功します。

モジュールをまたぐ場合、`spec` 関数の本体は、ほかのモジュールに公開することも、現在のモジュールだけの非公開にしておくこともできます。
関数に `open` という印が付いていれば、本体は公開されます。
この場合、関数の本体についてのアサーションが、ほかのモジュールでも成功します。

```rust
mod M1 {
    use verus_builtin::*;

    pub open spec fn min(x: int, y: int) -> int {
        if x <= y {
            x
        } else {
            y
        }
    }
}

mod M2 {
    use verus_builtin::*;
    use crate::M1::*;

    fn test() {
        assert(min(10, 20) == 10); // succeeds
    }
}
```

これに対して、関数に `closed` という印が付いている場合、ほかのモジュールは、関数の宣言は見えても、関数の本体は見えません。
ただし、同じモジュール内の関数は、`closed spec fn` の本体を見ることが*できます*。
言い換えると、`pub` は宣言を公開にし、`open` と `closed` は本体を公開または非公開にします。
`pub` な `spec` 関数にはすべて、`open` か `closed` のどちらかの印を付けなければなりません。
この注釈が関数に無いと、Verus は文句を言います。

```rust
mod M1 {
    use verus_builtin::*;

    pub closed spec fn min(x: int, y: int) -> int {
        if x <= y {
            x
        } else {
            y
        }
    }

    pub proof fn lemma_min(x: int, y: int)
        ensures
            min(x,y) <= x && min(x,y) <= y,
    {}
}

mod M2 {
    use verus_builtin::*;
    use crate::M1::*;

    fn test() {
        assert(min(10, 20) == min(10, 20)); // succeeds
        assert(min(10, 20) == 10); // FAILS
        proof {
            lemma_min(10,20);
        }
        assert(min(10, 20) <= 10); // succeeds
    }
}
```

`min` が `closed` になっている上の例でも、モジュール `M2` は関数 `min` について語ることはできます。
たとえば、`min(10, 20)` がそれ自身に等しいことを証明できます
（本体に何が書かれていようと、すべてのものはそれ自身に等しいからです）。
一方、`min(10, 20) == 10` というアサーションは失敗します。
`M2` は `min` の本体を見られないので、`min` が 2 つの数の最小値を計算することを知らないからです。

```
error: assertion failed
   |
   |         assert(min(10, 20) == 10); // FAILS
   |                ^^^^^^^^^^^^^^^^^ assertion failed
```

`lemma_min` の呼び出しの後では、`min(10, 20) <= 10` というアサーションが成功します。`lemma_min` が、`min(x,y) <= x` を事後条件として公開しているからです。`lemma_min` がこの事後条件を証明できるのは、`min` が `closed` であっても `min` の本体を見られるからです。`lemma_min` と `min` は同じモジュールにあります。

`pub open spec` 関数は略記を定義するもの、`pub closed spec` 関数は抽象を定義するものと考えることができます。
状況に応じて、どちらも役に立ちます。

`spec` 関数は、ほかの `spec` 関数から呼び出せます。
また、事前条件や事後条件のような、`exec` 関数の中の仕様からも呼び出せます。
たとえば、3 つの数の最小値 `min3` を、2 つの数の最小値を使って定義できます。
そのうえで、`exec` 関数 `compute_min3` を定義できます。
この関数は、変数を書き換える命令型のコードを使って 3 つの数の最小値を計算し、その事後条件を `spec` 関数 `min3` を使って定義します。

```rust
spec fn min(x: int, y: int) -> int {
    if x <= y {
        x
    } else {
        y
    }
}

spec fn min3(x: int, y: int, z: int) -> int {
    min(x, min(y, z))
}

fn compute_min3(x: u64, y: u64, z: u64) -> (m: u64)
    ensures
        m == min3(x as int, y as int, z as int),
{
    let mut m = x;
    if y < m {
        m = y;
    }
    if z < m {
        m = z;
    }
    m
}

fn test() {
    let m = compute_min3(10, 20, 30);
    assert(m == 10);
}

```

`min3` と `compute_min3` の違いは、`spec` コードと `exec` コードの違いのいくつかを際立たせています。
`exec` コードは、変更（mutation）のような命令型言語の機能を使えますが、`spec` コードは、純粋関数型の数学的なコードに限られます。
その一方で、`spec` コードは `int` と `nat` を使えますが、`exec` コードは、`u64` のようなコンパイルできる型に限られます。
{% endraw %}

{% raw %}
## 列挙型 {#enum}

Verus でも Rust と同じように、`enum` を使って、
定義した列挙子（variant）のうちのどれか 1 つになるデータ型を定義できます。
```rust
enum Beverage {
    Coffee { creamers: nat, sugar: bool },
    Soda { flavor: Syrup },
    Water { ice: bool },
}
```

`enum` は、メンバーのフィールドを持たせず、タグだけのために使うこともよくあります。
```rust
enum Syrup {
    Cola,
    RootBeer,
    Orange,
    LemonLime,
}
```

## `is` 演算子で列挙子を見分ける {#identifying-a-variant-with-the-is-operator}

spec の文脈では、`is` 演算子を使って、
変数が列挙型のどの列挙子を保持しているかを問い合わせられます。
```rust
fn make_float(bev: Beverage) -> Dessert
    requires bev is Soda
{
    assert(bev !is Coffee);
    Dessert::new(/*...*/)
}
```

構文 `!is` は `!(.. is ..)` の省略形です。

## 矢印演算子でフィールドにアクセスする {#accessing-fields-with-the-arrow-operator}

`Beverage` の例のように、すべてのフィールドの名前が異なるなら、
矢印演算子 `->` でフィールドを参照できます。
```rust
proof fn sufficiently_creamy(bev: Beverage) -> bool
    requires bev is Coffee
{
   bev->creamers >= 2
}
```

`enum` のフィールドが同じ名前を使い回している場合は、フィールドへのアクセスを修飾できます。
```rust
enum Life {
    Mammal { legs: int, has_pocket: bool },
    Arthropod { legs: int, wings: int },
    Plant { leaves: int },
}

spec fn is_insect(l: Life) -> bool
{
    l is Arthropod && l->Arthropod_legs == 6
}
```

`match` は Rust と同じように働きます。
```rust
enum Shape {
    Circle(int),
    Rect(int, int),
}

spec fn area_2(s: Shape) -> int {
    match s {
        Shape::Circle(radius) => { radius * radius * 3 },
        Shape::Rect(width, height) => { width * height }
    }
}
```

`Shape` のように丸括弧 `()` で宣言された列挙子では、
Verus の `->` によるタプル風の構文を使うと、
match で分解せずに 1 つのフィールドにアクセスできます。
```rust
spec fn rect_height(s: Shape) -> int
    recommends s is Rect
{
    s->1
}
```

## &&、==>、&&& と組み合わせる matches {#matches-with---and-}

`match` は、`enum` のすべての列挙子を調べるのに自然な書き方です。
列挙子のうち 1 つか 2 つだけを考えながらフィールドを束縛したい場合は、
Verus の `matches` 構文を使えます。
```rust
use Life::*;
spec fn cuddly(l: Life) -> bool {
    ||| l matches Mammal { legs, .. } && legs == 4
    ||| l matches Arthropod { legs, wings } && legs == 8 && wings == 0
}
```

`matches` 構文はパターンの中で名前を束縛するので、
列挙子をまたいで使い回されたフィールド名があっても困りません。
そのため、（修飾付きの）矢印構文よりも好ましい場合があります。

`l matches Mammal { legs, .. } && legs == 4` はブール式であり、
`&&` より後ろの式の残りの部分で `legs` が束縛されているという
特別な性質を持つことに注意してください。この便利な束縛は、`==>` や
`&&&` でも働きます。
```rust
spec fn is_kangaroo(l: Life) -> bool {
    &&& l matches Life::Mammal { legs, has_pocket }
    &&& legs == 2
    &&& has_pocket
}

spec fn walks_upright(l: Life) -> bool {
    l matches Life::Mammal { legs, .. } ==> legs == 2
}
```
{% endraw %}

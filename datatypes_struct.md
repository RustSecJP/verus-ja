{% raw %}
## 構造体 {#struct}

Verus でも Rust と同じように、`struct` を使って、
一組のフィールドをひとまとめにしたデータ型を定義できます。
```rust
struct Point {
    x: int,
    y: int,
}
```

spec コードと exec コードは、`struct` のフィールドを参照できます。
```rust
impl Point {
    spec fn len2(&self) -> int {
        self.x * self.x + self.y * self.y
    }
}

fn rotate_90(p: Point) -> (o: Point)
    ensures o.len2() == p.len2()
{
    let o = Point { x: -p.y, y: p.x };
    assert((-p.y) * (-p.y) == p.y * p.y) by(nonlinear_arith);
    o
}
```
{% endraw %}

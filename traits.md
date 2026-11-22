{% raw %}
# トレイト {#traits}

Verus は、`requires` 節と `ensures` 節を含め、トレイトの関数に対する仕様を書くことに対応しています。トレイトの関数に付けた仕様は、すべての実装が満たさなければならない*契約*として働きます。また、呼び出し側は、トレイト境界を通じてその関数を呼び出すときに、この契約に頼ることができます。

外部のクレートで定義されたトレイト（たとえば、Rust の標準ライブラリのもの）については、[外部トレイトの仕様](./external_trait_specifications.html)を参照してください。

## トレイトの関数の仕様 {#trait-function-specifications}

トレイトの関数には、通常の関数とまったく同じように `requires` 節と `ensures` 節を付けられます。

```rust
trait Compressor {
    fn compress(&self, input: u64) -> (output: u64)
        ensures output <= input;
}
```

`Compressor` を実装する型はどれも、本体が `output <= input` を満たす `compress` 関数を用意しなければなりません。

## 実装で仕様を拡張する {#extending-specifications-in-implementations}

実装は、トレイトの宣言にあるすべての `requires` 節と `ensures` 節を自動的に継承します。さらに、次のことが言えます。

 * `impl` は、より強い `ensures` 節を追加**できます**。
 * `impl` は、新しい `requires` 節を追加**できません**。追加できてしまうと健全でなくなります。トレイト境界 `C: Compressor` を使う呼び出し側には、トレイトが言及していない要件を満たす義務が無いからです。

```rust
struct HalfCompressor;

impl Compressor for HalfCompressor {
    // The trait's `ensures output <= input` is automatically inherited.
    // We additionally specify the exact return value.
    fn compress(&self, input: u64) -> (output: u64)
        ensures output == input / 2,
    {
        input / 2
    }
}
```

`HalfCompressor::compress` は、トレイトから `ensures output <= input` を継承します。
Verus は、この関数の本体が、継承した事後条件と、実装が追加した事後条件（`ensures output == input / 2`）の両方を満たすことを検証します。

トレイトの関数には、`requires` と `ensures` の両方を含めることもできます。

```rust
trait Bounded {
    fn clamp(&self, val: u64, lo: u64, hi: u64) -> (result: u64)
        requires lo <= hi,
        ensures lo <= result <= hi;
}
```

```rust
struct Saturate;

impl Bounded for Saturate {
    // Inherits: requires lo <= hi, ensures lo <= result <= hi
    // Adds: the exact formula for result
    fn clamp(&self, val: u64, lo: u64, hi: u64) -> (result: u64)
        ensures result == if val < lo { lo } else if val > hi { hi } else { val },
    {
        if val < lo { lo } else if val > hi { hi } else { val }
    }
}
```

## ジェネリックなディスパッチと具体的なディスパッチ {#generic-vs-concrete-dispatch}

トレイトの関数の呼び出しについて、Verus が具体的な型を静的に決定できるときは、`impl` にある、より強いかもしれない仕様を使います。呼び出しがジェネリックな型パラメータを通じて行われるときは、Verus に分かるのはトレイトの段階の仕様だけです。

```rust
fn compress_generic<C: Compressor>(c: &C, x: u64) {
    let r = c.compress(x);
    assert(r <= x); // OK: trait-level ensures holds for every C
}

fn compress_concrete(c: &HalfCompressor, x: u64) {
    let r = c.compress(x);
    assert(r <= x);      // From the trait ensures
    assert(r == x / 2);  // From HalfCompressor's stronger ensures (statically resolved)
}
```

## spec 関数と proof 関数 {#spec-and-proof-functions}

トレイトには、`spec` 関数と `proof` 関数の宣言を含めることもできます。
たとえば、次のトレイトは、実装に対して、距離の尺度（`dist`）を仕様関数として与えること、そして、その `distance` 関数が実際にその尺度を計算することを証明することを求めます。
さらに実装は、`dist` が妥当な尺度であることも（`valid_distance_metric` の本体で）証明しなければなりません。

```rust
trait Distance {
    spec fn dist(&self, other: &Self) -> nat;

    fn distance(&self, other: &Self) -> (d: u64)
        ensures
            d as nat == self.dist(other),
    ;

    proof fn valid_distance_metric()
        ensures
            forall |x: &Self, y| x.dist(y) == y.dist(x),
            forall |x: &Self, y| x.dist(y) == 0 <==> x == y,
            forall |x: &Self, y, z| x.dist(y) <= x.dist(z) + z.dist(y),
        ;
}
```

この `Distance` トレイトの実装例を次に示します。
```rust
struct P {
    u: u64
}

impl Distance for P {
    spec fn dist(&self, other: &Self) -> nat {
        vstd::math::abs(self.u - other.u)
    }

    fn distance(&self, other: &Self) -> u64 {
        if self.u > other.u {
            self.u - other.u
        } else {
            other.u - self.u
        }
    }

    proof fn valid_distance_metric() 
    {
    }
}
```
この場合、Verus は `valid_distance_metric` の事後条件をすべて自動的に証明できますが、もっと複雑な距離の尺度では、追加の証明注釈が必要になるかもしれません。

実装がトレイトの定義から継承する requires / ensures を、繰り返し書く必要は無いことに注意してください。


## `View` トレイト {#the-view-trait}

`vstd` で最もよく使われるトレイトは `View` です。`View` を使うと、実行可能な型に、`view` 関数を通じてアクセスする数学的な抽象（`type V`）を与えられます。この関数は非常によく呼び出されるので、Verus は短縮記法として [`@` 演算子](reference-at-sign.html)を用意しており、`x.view()` の代わりに `x@` と書けます。

```rust
pub trait View {
    type V;
    spec fn view(&self) -> Self::V;
}
```

`vstd` は、よく使われる型に対する `View` の実装を提供しています。
- `Vec<T>` → `Seq<T>`
- `HashMap<K, V>` → `Map<K, V>`
- `HashSet<K>` → `Set<K>`
- プリミティブ型（`u64`、`bool` など） → それ自身

独自の型に `View` を実装するには、適切な抽象を選び、`type V` と `spec fn view` を定義します。

```rust
struct Stack {
    data: Vec<u64>,
}

impl View for Stack {
    type V = Seq<u64>;

    closed spec fn view(&self) -> Seq<u64> {
        self.data@
    }
}

impl Stack {
    fn push(&mut self, val: u64)
        ensures final(self)@ == old(self)@.push(val),
    {
        self.data.push(val);
    }

    fn is_empty(&self) -> (result: bool)
        ensures result <==> self@.len() == 0,
    {
        self.data.len() == 0
    }
}
```

`Stack` の `data` は非公開のフィールドなので、`view` は `closed` です。呼び出し側はその定義を見ることはできませんが、`push` と `is_empty` の事後条件が示すとおり、各関数がその view に与える効果については推論できます。呼び出し側に `@` をその定義まで展開させたい場合（たとえば、`pub` なフィールドを持つ `pub` な型の場合）は、代わりに `open spec fn view` を使います。

`vstd` は `DeepView` も提供しています。これは、入れ子になった要素に view の抽象を再帰的に適用するものです。ほとんどのコードでは `View` だけで足ります。

## `default_ensures`: 既定の関数実装のための仕様 {#default_ensures-specifications-for-default-function-implementations}

Rust では、トレイトの関数に*既定の実装*を与えることができます。これは、実装がその関数を上書きしない場合に継承する本体です。ここから、仕様に関する微妙な問題が生じます。トレイトの段階の `ensures` 節は、妥当な上書きをどれも許せるくらい弱くなければなりませんが、既定の本体は*より強い*事後条件を満たしているかもしれないのです。

`default_ensures` は、この 2 つの関心事を分けることで、この問題を解決します。

```rust
trait Reducer {
    // Every implementation must satisfy: output <= input.
    // The default implementation additionally satisfies: output == input / 2.
    fn halve(&self, input: u64) -> (output: u64)
        ensures output <= input,
        default_ensures output == input / 2,
    {
        input / 2
    }
}
```

ここで、**トレイトの段階の `ensures`**（`output <= input`）は、どの実装も満たさなければならないものです。**`default_ensures`**（`output == input / 2`）は追加の保証であり、型が既定の実装を上書きせずに使うとき*に限って*成り立ちます。

規則は次のとおりです。

 * `default_ensures` を付けられるのは、トレイトの宣言の中に既定の本体を持つトレイトの関数だけです。
 * `default_ensures` は、通常の `ensures` とまったく同じように、既定の本体に対して検査されます。
 * 型が既定の実装を継承していることを静的に知っている呼び出し側は、`ensures` と `default_ensures` の両方を知ることができます。ジェネリックな境界 `T: Trait` を使う呼び出し側が知ることができるのは、`ensures` だけです。

```rust
struct DefaultReducer;

impl Reducer for DefaultReducer {
    // No override: inherits the default implementation and its default_ensures.
}

struct ThirdReducer;

impl Reducer for ThirdReducer {
    // Overrides with a different strategy.  Only the trait `ensures` (output <= input)
    // applies to callers who don't statically know the type.
    fn halve(&self, input: u64) -> (output: u64)
        ensures output == input / 3,
    {
        input / 3
    }
}
```

```rust
fn call_generic<H: Reducer>(h: &H, x: u64) {
    let r = h.halve(x);
    assert(r <= x);        // From trait ensures — always available
    // assert(r == x / 2); // Would FAIL: not known for arbitrary H
}

fn call_default(h: &DefaultReducer, x: u64) {
    let r = h.halve(x);
    assert(r <= x);     // From trait ensures
    assert(r == x / 2); // From default_ensures (DefaultReducer uses the default impl)
}

fn call_override(h: &ThirdReducer, x: u64) {
    let r = h.halve(x);
    assert(r <= x);        // From trait ensures
    assert(r == x / 3);    // From ThirdReducer's own ensures
    // assert(r == x / 2); // Would FAIL: ThirdReducer overrides, no default_ensures
}
```

独自のトレイトを書くときは、妥当な既定の実装がより強い約束をしていて、独自の実装にはそれと同じ約束を求めない、という関数に `default_ensures` を使うことを検討してください。

## よく使われる `vstd` のトレイトの仕様 {#common-vstd-trait-specifications}

`vstd` は、標準ライブラリの多くのトレイトに仕様を提供しています。知っておくとよいものをいくつか挙げます。

 * **`PartialEq` / `Eq`** — `vstd` はこれらを `obeys_eq_spec()` というガードで包んでいます。これにより、自分から参加を選んだ型だけが、関数としての等価性の契約を満たすと仮定されます。`obeys_*` のパターンについては、[外部トレイトの仕様](./external_trait_specifications.html)を参照してください。
 * **`Iterator`** — `vstd` は、独自のイテレータに仕様を書くために、（`Iterator` を直接ではなく）`IteratorSpecImpl` を提供しています。[イテレータの仕様](./iterator-specs.html)を参照してください。
 * **`PartialOrd` / `Ord`** — `PartialEq` と同様で、`obeys_ord_spec()` というガードが付いています。

## 外部トレイトの仕様 {#external-trait-specifications}

外部のクレート（`std` を含む）のトレイトに仕様を追加する方法については、[外部トレイトの仕様](./external_trait_specifications.html)を参照してください。
{% endraw %}

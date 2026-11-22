{% raw %}
# 内部可変性 {#interior-mutability}

[内部可変性（interior mutability）パターン](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html)は、共有借用 `&` を通じてアクセスした値の中身を、利用者が操作できるという、Rust に特有のパターンです。（`&` は「不変借用」と呼ばれることが多いのですが、ここでは混乱を避けるために「共有借用」と呼びます。）内部可変性を示す Rust の型としてよく知られているのが、[`Cell` と `RefCell`](https://doc.rust-lang.org/std/cell/) の 2 つです。ここでは、Verus でこれらに相当する概念を概観します。

### 変更できないものを変更する {#mutating-stuff-that-cant-mutate}

こうした内部可変性のパターンを検証するうえでの中心的な難しさを理解するために、Verus の SMT エンコーディングについての重要な事実を思い出してください。Verus は、どんな型 `T` についても、`&T` 型の値は決して変化しないと仮定します。しかし私たちは、`&Cell<T>` の中身が変化しうることも知っています。そもそも、それこそが `Cell<T>` 型の存在意義なのです。

そうなると、避けられない結論は、
_Verus の SMT エンコーディングにおいて `Cell<T>` がとる値は、セルの中身に依存してはならない_、ということです。その代わり、`Cell<T>` の SMT 上の「値」は、そのセルの一意な識別子にすぎません。ある面では、`Cell<T>` をポインタ `T*` に似たものと考えると分かりやすいかもしれません。`Cell<T>` の値は、その中身（「ポインタが指している先のもの」）ではなく、識別子（「ポインタのアドレス」）_だけ_ なのです。もちろん、これはポインタでは _ありません_。しかしエンコーディングの観点からは、ポインタだと思っても差し支えありません。

この性質からすぐに導かれる帰結を 1 つ挙げておきます。`Cell` 型に対する [Verus の純粋な等価性 `===`](./equality.html) は、`Cell` 型に対する Rust の標準の `==`（`eq`）と同じ結果になることはありえません。Rust の `==` 関数は、実際にセルの中身を比べます。しかし純粋な等価性 `===` は、SMT エンコーディング上の値に依存しなければならないので、中身に依存することはありえないのです。その代わり、`===` は、2 つのセルが _同じセル_ であるときに限って、それらを等しいと見なします。

では、こうした難しさを踏まえたうえで、Verus では内部可変性を _どのように_ 扱うのでしょうか。

取りうる方法はいくつかあります。

 * Cell のようなデータ構造の内部から値を取り出すとき、これを、その型の値を非決定的に受け取ることとしてモデル化できます。
   一見すると、これでは正しさの性質を検証するための手がかりが少なすぎるように思えるかもしれません。しかし、_データ不変条件_ を指定してありうる値の範囲を絞ることで、追加の構造を課すことができます。

 * `tracked ghost` コードを使って、正確な値を追跡する。

より洗練されたデータ構造、特に並行なデータ構造では、この 2 つの方法を注意深く組み合わせることがしばしば必要になります。ここでは両方を紹介します。

### `InvCell` によるデータ不変条件。 {#data-invariants-with-invcell}

高価な計算があり、その値をメモ化したいとします。値を計算する必要が初めて生じたときに計算を行い、後で必要になったときのためにその値を保存しておきます。これを行うために `Cell` を使います。計算した値を保存するためのもので、その内部は `None` で初期化しておきます。メモ化した計算の関数は、次のように動きます。

 * `Cell` の値を読む。
   * `None` なら、値はまだ計算されていません。
     値を計算し、後のために保存してから、それを返します。
   * `Some(x)` なら、値はすでに計算されているので、
     すぐにそれを返します。

重要なのは、この方法の正しさが、個々の呼び出しがどちらの場合になるかを予測できるかどうかには、実は依存しないということです。（プログラムにかかる時間の上界を証明しようとしているのなら、話は違ってくるかもしれません。）知っておく必要があるのは、これらの場合の _どれか 1 つ_ になる、ということだけです。したがって、データ不変条件を持つセルを使うことで、このコードを検証できます。

 * _不変条件:_ セルの内部に保存されている値は、`None` か `Some(x)` のどちらかである。ここで `x` は、計算の期待される結果である。

具体的には、上の内容は、Verus の標準ライブラリが提供する [`InvCell`](https://verus-lang.github.io/verus/verusdoc/vstd/cell/struct.InvCell.html) を使って Verus で実装できます。`InvCell` は、データ不変条件に基づく仕様を提供します。`InvCell<T, Pred>` は 2 つ目の型パラメータ `Pred` を取り、これは [`Predicate<T>`](https://verus-lang.github.io/verus/verusdoc/vstd/predicate/trait.Predicate.html) トレイトを実装します。これがセルのデータ不変条件を与えます。つまり、型 `T` についての真偽値の述語で、どんな値を保存してよいかをセルに伝えるものです。（`InvCell::new(val, Ghost(pred))` で新しい `InvCell` を構築するとき、利用者は `Pred` 型のゴースト値 `pred` を与え、初期値 `val` がそれを満たすことを示さなければなりません。）そうすれば、`InvCell` が課さなければならない制限は、利用者がセルに書き込むときはいつでも、書き込まれる値 `val` が述語 `cell.inv(val)` を満たさなければならない、ということだけです。その代わりに、利用者がセルから _読む_ ときはいつでも、受け取る値が `cell.inv(val)` を満たすことが分かります。

次は、`InvCell` を使ってメモ化した関数を実装する例です。

```rust
spec fn result_of_computation() -> u64 {
    2
}

fn expensive_computation() -> (res: u64)
    ensures
        res == result_of_computation(),
{
    1 + 1
}

// Implement this trait to specify validity for the cell contents
//
// Validity of the interior contents of the cell means that the value is either
// `None` or `Some(i)` where `i` is the desired value.
struct MemoizedCellInvariant;
impl Predicate<Option<u64>> for MemoizedCellInvariant {
    closed spec fn predicate(&self, v: Option<u64>) -> bool {
        match v {
            Option::Some(i) => i == result_of_computation(),
            Option::None => true,
        }
    }
}

// Memoize the call to `expensive_computation()`.
// The argument here is an InvCell wrapping an Option<u64>,
// which is initially None, but then it is set to the correct
// answer once it's computed.
fn memoized_computation(cell: &InvCell<Option<u64>, MemoizedCellInvariant>) -> (res: u64)
    ensures
        res == result_of_computation(),
{
    let c = cell.get();
    match c {
        Option::Some(i) => {
            // The value has already been computed; return the cached value
            i
        },
        Option::None => {
            // The value hasn't been computed yet. Compute it here
            let i = expensive_computation();
            // Store it for later
            cell.replace(Option::Some(i));
            // And return it now
            i
        },
    }
}
```

### `PCell` による追跡されるゴースト状態。 {#tracked-ghost-state-with-pcell}

（TODO この章を書き上げる）
{% endraw %}

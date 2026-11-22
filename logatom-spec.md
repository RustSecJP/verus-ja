{% raw %}
# アトミックな仕様 {#atomic-specifications}

関数の仕様に `atomically` ブロックを加えることで、その関数が論理的にアトミックであると宣言できます。アトミックな仕様は、一般に次のような形をしています。

```rs
exec fn function(px: PX) -> (py: PY)
    atomically (atomic_update) {
        type PredType,

        (ax: AX) -> (ay: AY),

        requires atomic_pre(px, ax),
        ensures atomic_post(px, ax, ay),

        outer_mask [...],
        inner_mask [...],
    },

    requires private_pre(px),
    ensures private_post(px, ax, ay, py),
```

`atomically` ブロックの中には、`requires` 節と `ensures` 節があります。これらは *アトミックな* 事前条件と事後条件を表し、それぞれ線形化点（LP）の直前と直後のプログラムの状態を記述します。

アトミックな事前条件と事後条件にリソースを結び付けるには、矢印 `(ax: AX) -> (ay: AY)` を使います。以下では、`ax: AX` をアトミック更新の *入力型*、`ay: AY` を *出力型* と呼びます。`ax` と `ay` はどちらも、常に tracked モードです。

このアトミックな仕様は、`AtomicUpdate<AX, AY, PredType>` 型の変数を追加で束縛します。この `tracked` な変数の名前は利用者が決められますが、このガイドでは一貫して `atomic_update` と呼ぶことにします。述語型には、上で見たように、省略可能な `type PredType` 節を使って名前を付けられます。詳しくは、下の[述語型についての節](#the-predicate-type)を参照してください。

最後の 2 つの節、`outer_mask` と `inner_mask` は、アトミック更新の不変条件マスクを指定します。これらが何をするのかは、後の節で見ていきます。

## アボートの場合 {#the-abort-case}

アトミック更新の出力型（上の `AY`）は、`UpdateTry` トレイトを実装していなければなりません。このトレイトによって、ある特定の出力値が、アトミック更新がコミットされたことを示すのか、アボートされたことを示すのかを判定できます。

```rs
pub enum UpdateControlFlow {
    Commit,
    Abort,
}

pub trait UpdateTry {
    spec fn branch(self) -> UpdateControlFlow;
}
```

Verus の標準ライブラリは、次の 2 つの型について、このトレイトの実装を提供しています。

- `AY = Result<T, E>` は、`Ok(t)` を出力することでコミットでき、`Err(e)` を使ってアボートできるアトミック更新を指定します。これは Iris のアトミック更新と等価です。ただし、`AX` と `E` が同じであることは強制しません。

- `AY = Commit<T>` は、アボートできないアトミック更新を指定します。

コミットやアボートの場合が複数あってそれらを区別したいときは、独自の出力型にこのトレイトを実装すると役に立つことがあります。

`Commit<T>` と同じ要領で、アボート専用の型 `Abort<T>` を定義することも可能です。ただし、そのような出力型を使うと、ライブラリの関数もアトミックな関数呼び出しも停止できなくなります。これは通常、論理的にアトミックな関数にとって望ましくありません。

## 述語型 {#the-predicate-type}

上の仕様によって束縛されるアトミック更新オブジェクトは、`AtomicUpdate<AX, AY, PredType>` という型を持ちます。Verus の不変条件の型と同じように、`vstd` が提供する `AtomicUpdate` 型は 1 つだけです。この型は、最後の型引数（これを **述語型** と呼びます）を使い、トレイトの実装によって *設定* されます。

述語型とそのトレイトの実装は、利用者がアトミックな仕様を書くと、Verus によって自動的に生成されます。上の仕様に対しては、（おおよそ）次のものを生成します。

```rs
struct PredType { px: Ghost<PX> }

impl UpdatePredicate<AX, AY> for PredType {
    open spec fn req(self, ax: AX)         -> bool { atomic_pre  }
    open spec fn ens(self, ax: AX, ay: AY) -> bool { atomic_post }

    open spec fn outer_mask(self) -> ISet<int> { [...] }
    open spec fn inner_mask(self) -> ISet<int> { [...] }
}

impl PredType {
    open spec fn args(self, px: PX) -> bool { self.px == px }
}
```

関数の引数を述語型の中に保存しているのは、アトミックな事前条件と事後条件がそれらに依存できるようにするためです。アトミック更新が関数の境界をまたいで移動したり、ループのまわりで移動したりする場合（特に[ループの分離が有効になっている](./reference-attributes.html#verifierloop_isolation)場合）、関数の引数の値をあらためて表明する必要が生じることがあります。これは、上で定義した `PredType::args` 述語を使って、`au.pred().args(...)` で行えます。

現在のところ、独自の述語型や既存の述語型を使って、論理的にアトミックな関数を定義することはできません。これは、`UpdatePredicate` トレイトを手で実装しても役に立たない、ということでもあります。

## 一貫して使う例 {#running-examples}

2 つの例の関数に対するアトミックな仕様は、次のようになるでしょう。

```rs
fn reset(var: &PAtomicU64)
    atomically (atomic_update) {
        (perm: PermissionU64) -> (commit: Commit<PermissionU64>),

        requires
            perm@.patomic == var.id(),

        ensures
            commit@@.patomic == perm@.patomic,
            commit@@.value == 0,

        outer_mask any,
        inner_mask none,
    },
```

ここでは、アトミック更新が、アトミック変数 `var` に対する `PermissionU64` を入力として受け取ることを指定しています。出力は、同じ ID を持つ（つまり、同じ変数に対する）許可オブジェクトで、値は 0 です。アトミック更新の出力は `Commit<...>` で包まれています。これは、この AU がアボートできないことを指定します。

```rs
fn increment(var: &PAtomicU64) -> (out: u64)
    atomically (atomic_update) {
        (perm: PermissionU64)
            -> (res: Result<PermissionU64, (PermissionU64, OpenInvariantCredit)>),

        requires
            perm@.patomic == var.id(),

        ensures match res {
            Err((p, _)) => p@ == perm@,
            Ok(p) => p@.patomic == perm@.patomic
                  && p@.value == perm@.value.wrapping_add(1),
        },

        outer_mask any,
        inner_mask none,
    },

    ensures
        out == perm@.value,
```

こちらの仕様は、もう少し興味深いものです。ここでは出力が `Result<...>` であり、これは、アトミック更新をアボートできることを意味します。アトミックな事後条件は、AU がアボートされたとき（つまり `res` が `Err` のとき）、許可の ID と値が変わらないことを指定します。そうでなく AU がコミットされたときは、値が 1 だけ増えます。AU がアボートされたときは、ライブラリからクライアントへ `OpenInvariantCredit` も渡します。これによってクライアントは、アトミックな関数呼び出しの中で、不変条件を複数回開けるようになります。これについては後で詳しく見ます。最後に、関数の事後条件で指定されているとおり、この関数はアトミック変数の以前の値を返します。
{% endraw %}

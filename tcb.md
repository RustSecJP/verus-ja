{% raw %}
# 仮定と、信頼される構成要素 {#assumptions-and-trusted-components}

コードのすべての行を検証することができず、いくつかの事柄を _仮定_ しなければならないことはよくあります。
そのような場合、コードの最終的な正しさは、検証だけでなく、置かれた仮定にも左右されます。

仮定は、次の仕組みを通じて持ち込まれます。

 * [`assume`](./requires_ensures.html) 文
 * 公理。[`#[verifier::external_body]`](./calling-unverified-from-verified.html) を付けて導入された関数、または `axiom fn` として宣言された関数
 * 公理的な仕様。`#[verifier::external_body]`、`#[verifier::external_fn_specification]`、または `assume_specification` を使って導入された exec 関数
 * `#[verifier::external]`（下記を参照）

コマンドラインフラグ `--no-cheating` は、コードベースの中にあるこうした仮定を見つけ出して管理するのに役立ちます。

型（構造体と列挙型）にも `#[verifier::external_body]` を付けられます。
ただし、細かいことを言えば、これは _それ自体では_ 新しい仮定を持ち込みません。
とはいえ実際には、そうした型は、役に立つものにするための追加の仮定を伴うのが普通です。

これらの仮定が現れてよい場所を管理するために、Verus は [`--no-cheating` モード](#no-cheating-mode)を用意しています。
このモードは、仮定を、明示的に信頼済みとされたコードの中に閉じ込めます。

### `#[verifier::external]` 属性 {#the-verifierexternal-attribute}

`#[verifier::external]` という注釈は、アイテムを完全に無視するよう Verus に伝えます。
これは、関数、トレイト、トレイト実装、型など、どんなアイテムにも付けられます。

多くのアイテム（関数、型、トレイト宣言）については、これは _それだけでは_、そのアイテムについての新しい「仮定」を何も持ち込みません。
たとえば、検証済みの関数から `external` な関数を呼び出そうとすると、Verus はエラーを出します。
実際には、開発者は `external` な関数（`f` とします）を `external_body` な関数（`g` とします）から呼び出すことが多いでしょう。
その場合、`external_body` 属性が `g` についての仮定を持ち込み、それによって `f` についての仮定を _間接的に_ 持ち込むことになります。

さらに、_トレイト実装_ に `#[verifier::external]` を付ける場合は、いっそう慎重な検討が必要です。
Verus は一部の事柄を rustc のトレイト検査に頼っているため、トレイト実装が、どのコードが受け入れられ、どのコードが拒否されるかに影響することがあるからです。

例を示します。

```rust
#[verifier::external]
unsafe impl Send for X { }
```

## no-cheating モード {#no-cheating-mode}

既定では、Verus は、上記の仕組みを通じて検証されていない仮定を持ち込むことを許します。
プロジェクトの確かさを高めるために、コマンドラインフラグ `--no-cheating` を使うことをおすすめします。
このフラグは、そうした「ずる（cheating）」を既定ですべて禁止し、仮定を、明示的に信頼済みとされた、監査できるコードの中に閉じ込めることを強制します。

Verus を `--no-cheating` 付きで実行すると、仮定が明示的に許可されていない場所では、次のものがすべて拒否されます。

 * [`assume`](./requires_ensures.html#assert-and-assume) と `admit`
 * 関数（公理を含みます）に付けた [`#[verifier::external_body]`](./calling-unverified-from-verified.html)
 * [`assume_specification`](./reference-assume-specification.html) と `#[verifier::external_fn_specification]`
 * [`#[verifier::assume_termination]`](./reference-attributes.html#verifierassume_termination)
 * `externals_available_without_declaration` によって可能になる呼び出し

（`#[verifier::external]` は影響を受けません。それだけでは仮定を持ち込まないからです。）

### 信頼済みコードの印付け {#marking-trusted-code}

コードは、既定では信頼されていません。
ファイル、モジュール、またはアイテムの中で仮定を許可するには、`#[verus::trusted]` を使います。
内部属性（inner attribute）は、ファイル単位で適用されます。

```rust
// environment.rs
#![verus::trusted]
```

モジュール単位またはアイテム単位では、外部属性（outer attribute）を使います。
```rust
#[verus::trusted]
mod environment;
```

信頼済みという印は、子のアイテムに継承されます。
子は、`#[verus::untrusted]` を使って、通常の no-cheating の検査の対象に戻ることを選べます。
あるアイテムに明示的に `untrusted` の印を付けると、その子孫はどれも `trusted` と `trusted(spec)` を使えなくなります。

```rust
#[verus::trusted]
mod environment {
    // Trusted through inheritance.
    proof fn platform_axiom() {
        assume(platform_property());
    }

    // Assumptions are prohibited here and in all descendants.
    #[verus::untrusted]
    mod verified_model {
        // ...
    }
}
```

### 信頼済みコードの閉包 {#trusted-closure}

信頼済みコードは、そのクレートの中で推移的に閉じていなければなりません。
つまり、信頼済みコードが参照するローカルのアイテムは、どれも信頼済みでなければなりません。
こうすることで、クレート内の信頼されていないコードを変更しても、信頼済みコードが置いた仮定や定義などに影響が及ばなくなります。

[`vstd`](./vstd.html) やそのほかの外部クレートへの参照は許されます。
クレートのルートにある `use` 宣言は、閉包の起点としては無視されます。
ただし、そうしたインポートを経由した参照は、それが解決される実際のアイテムに対して検査されます。

信頼されていないコードは、信頼済みコードを自由に参照できます。
これにより、信頼される基盤に明確な依存関係の境界が与えられ、監査しなければならないコードが自己完結した状態に保たれます。

### 関数の仕様だけを信頼する {#trusting-only-a-function-specification}

関数には `#[verus::trusted(spec)]` を使えます。これは、関数のインターフェースだけを信頼済みとして印付けます。

 * Rust の型シグネチャ、ジェネリック境界、関連型
 * Verus の仕様。`requires`、`recommends`、`ensures`、`returns`、`decreases`、不変条件マスク、巻き戻し、および関連する節を含みます

実装の本体は、信頼されていないままです。
本体には、`assume`、`external_body`、そのほか `--no-cheating` が禁止する構文を含められません。
実装の本体だけが行う参照は、信頼済みコードの参照の閉包の検査の対象ではないため、本体は、信頼されていない普通のコードを呼び出せます。
`trusted(spec)` は、関数に直接付けることしかできません。

### 例 {#example}

```rust
// lib.rs (crate root)
// Trusted models of the environment and external dependencies live here:
#[verus::trusted]
mod trusted;

// ...the rest of the crate is fully verified, with no assumptions...
```

```rust
// trusted.rs
use vstd::prelude::*;

verus! {

// OK: assumptions are allowed because the module is trusted.
pub proof fn environment_axiom()
    ensures some_property(),
{
    assume(some_property());
}

}
```
{% endraw %}

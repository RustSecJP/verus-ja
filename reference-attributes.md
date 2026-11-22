{% raw %}
# 属性 {#attributes}

 - `accept_recursive_types`
 - [`all_triggers`](#all_triggers)
 - [`allow_complex_invariants`](#verifierallow_complex_invariants)
 - [`allow_in_spec`](#verifierallow_in_spec)
 - [`atomic`](#verifieratomic)
 - [`auto`](#auto)
 - [`custom_err`](#verifiercustom_errtext-and-verifiercustom_errtext)
 - [`external`](#verifierexternal)
 - [`external_body`](#verifierexternal_body)
 - `external_fn_specification`
 - `external_type_specification`
 - [`ext_equal`](#verifierext_equal)
 - [`inline`](#verifierinline)
 - [`loop_isolation`](#verifierloop_isolation)
 - [`memoize`](#verifiermemoize)
 - [`opaque`](#verifieropaque)
 - [`proof_note`](#verifierproof_notetext-and-verifierproof_notetext)
 - [`prophetic`](#verifierprophetic)
 - `reject_recursive_types`
 - `reject_recursive_types_in_ground_variants`
 - [`rlimit`](#verifierrlimitn-and-verifierrlimitinfinity)
 - [`trigger`](#trigger)
 - [`truncate`](#verifiertruncate)
 - [`type_invariant`](#verifiertype_invariant)
 - [`when_used_as_spec`](#verifierwhen_used_as_spec)
 - [`exec_allows_no_decreases_clause`](#verifierexec_allows_no_decreases_clause)
 - [`assume_termination`](#verifierassume_termination)

## `#![all_triggers]` {#all_triggers}

量化子に付ける属性で、その量化子に対するトリガーグループを積極的に選ぶよう Verus に指示します。
詳しくは[トリガーの指定手順](./trigger-annotations.html#selecting-trigger-groups)を参照してください。

Verus のほとんどの属性と違って、これには `verifier::` という接頭辞が要りません。

## `#[verifier::allow_complex_invariants]` {#verifierallow_complex_invariants}

既定では、`invariant_except_break` と `ensures` は、
[`#[verifier::loop_isolation(false)]`](#verifierloop_isolation) と一緒には使えません。必要が無いからです。
ループの分離を無効にすると、最弱事前条件（weakest precondition）の計算が、
break を通ってループのあとのコードへ至るすべての経路を自動的に追跡するので、
こうした複雑な種類の不変条件は不要になります。

ただし、場合によっては（ループの分離の設定を切り替えて試すときや、for ループの脱糖のためなど）、
ループの分離を無効にしていても、これらの種類の不変条件を使えると便利なことがあります。
`allow_complex_invariants` 属性は、不変条件を次のように変換することで、これを可能にします。
 * `invariant_except_break` 節は、通常の `invariant` 節に変換されます
 * `ensures` 節は無視されます（最弱事前条件の計算と重複するため）

**この属性が適用されるのは、`loop_isolation` が false のときだけです。** `loop_isolation(true)`
（または既定の設定）と一緒に使うと、エラーになります。

## `#[verifier::allow_in_spec]` {#verifierallow_in_spec}

[`returns` 節](./reference-returns.html)を持つ実行可能な関数に付けられます。
これにより、その関数を spec モードで使えるようになります。spec モードでは、その関数は、
指定された戻り値と等価なものとして解釈されます。

## `#[verifier::atomic]` {#verifieratomic}

`#[verifier::atomic]` 属性は、任意の *exec モード*の関数に付けられます。これは、
[`open_atomic_invariant!`](https://verus-lang.github.io/verus/verusdoc/vstd/macro.open_atomic_invariant.html)
によるアトミック性の検査において、その関数が「アトミック」であることを表します。

Verus は、本体が本当にアトミックであることを検査します。ただし、関数に `external_body` も付いている場合は別で、
その場合、この性質は、関数のシグネチャの残りの部分と一緒に仮定されます。

この属性は、`vstd` の[信頼されたアトミック型](https://verus-lang.github.io/verus/verusdoc/vstd/atomic/index.html)で使われています。

## `#![auto]` {#auto}

量化子に付ける属性で、Verus に経験則を使ってトリガーを自動的に推論させる、という意図を表します。
技術的には検証に影響しませんが、トリガーの詳細なログ出力に影響することがあります。
詳しくは[トリガーの指定手順](./trigger-annotations.html#selecting-trigger-groups)を参照してください。

Verus のほとんどの属性と違って、これには `verifier::` という接頭辞が要りません。

## `#[verifier::external]` {#verifierexternal}

与えられた項目を無視するよう Verus に伝えます。検証済みのコードがその項目を参照しようとすると、Verus はエラーを出します。

これは、検証済みのクレートの TCB に、無視できない影響を与えることがあります。[こちら](./tcb.html)を参照してください。

この属性は、項目の構文解析のされ方には影響しません。
項目が `verus!` マクロの中にあれば、その項目は、素の Rust の構文ではなく、
引き続き `verus!` の構文で解析されます。
素の Rust の構文を使いたい場合は、単に項目を `verus!` マクロの外で定義すればよく、
その項目は、`#[verifier::external]` を使うまでもなく、
自動的に Verus の外部のものとして扱われます。

## `#[verifier::external_body]` {#verifierexternal_body}

関数の定義だけを考慮し、関数の本体は考慮しないよう Verus に伝えます。
本体がその仕様を正しく満たしていることは、信頼されます。

これは、検証済みのクレートの TCB に、無視できない影響を与えることがあります。[こちら](./tcb.html)を参照してください。

この属性は、関数の本体の構文解析のされ方には影響しません。
関数が `verus!` マクロの中にあれば、関数の本体は、素の Rust の構文ではなく、
引き続き `verus!` の構文で解析されます。
素の Rust の構文を使いたい場合は、`verus!` と `#[verifier::external_body]` の組み合わせの代わりに、
`#[verus_spec]` の[属性構文](./exec_attr.html)と `#[verifier::external_body]` を使って、
関数を `verus!` マクロの外で定義できます。

## `#[verifier::ext_equal]` {#verifierext_equal}

`Seq`、`Set`、`Map`、`Multiset`、`spec_fn` のフィールド、
またはほかの `#[verifier::ext_equal]` なデータ型のフィールドについて、
外延性（extensionality）が必要なデータ型に印を付けるために使います。

詳しくは[外延性による等価性についての説明](./extensional_equality.html#equality-via-extensionality)を参照してください。

## `#[verifier::inline]` {#verifierinline}

`#[verifier::inline]` 属性は、任意の *spec モード*の関数に付けられます。これは、
Verus が SMT-LIB への符号化の中でその定義を自動的に展開すべきであることを表します。

これは関数の意味論には影響しませんが、トリガーに影響することがあります。

## `#[verifier::loop_isolation]` {#verifierloop_isolation}

`#[verifier::loop_isolation(false)]` 属性と `#[verifier::loop_isolation(true)]` 属性は、
モジュール、関数、個々のループに付けられます。どのループについても、
当てはまる属性のうち、もっとも対象が狭いものが優先されます。
この属性は、ループの本体の中で（ループ不変条件が無い場合に）Verus が自動的に行える推論に影響します。

 * `true` に設定した場合: Verus は、ループの本体の中で何も自動的には推論しません。
   関数の事前条件でさえも推論しません。
 * `false` に設定した場合: Verus は、ループの本体の外にある事実の一部を、
   ループの本体の中で自動的に使えるようにします。特に、ループの本体の外にあるアサーションのうち、
   ループの本体が変更しない変数だけに依存するものは、ループの中でも使えるようになります。

## `#[verifier::memoize]` {#verifiermemoize}

`#[verifier::memoize]` 属性は、任意の *spec モード*の関数に付けられます。これは、
[`by(compute)` と `by(compute_only)` の証明器モード](./reference-prover-mode-compute.html)が、
この関数の結果を「メモ化」すべきであることを表します。

## `#[verifier::opaque]` {#verifieropaque}

この関数の定義を自動的には reveal しないよう、ソルバーに指示します。
定義は、[`reveal` ディレクティブと `reveal_with_fuel` ディレクティブ](./reference-reveal-hide.html)を使って、局所的に reveal できます。

## `#[verifier::proof_note("text")]` と `#![verifier::proof_note("text")]` {#verifierproof_notetext-and-verifierproof_notetext}

これらの属性は、`requires` / `ensures` 節、または `assume` / `assert` 文に、文字列のメモを付けます。

- 外側の属性 `#[verifier::proof_note]` は、`assume` / `assert` 文に付けなければなりません。
- 内側の属性 `#![verifier::proof_note]`（`!` に注意）は、`requires` / `ensures` 節に付けなければなりません。

証明義務（`requires` / `ensures` / `assert`）が失敗すると、メモの `"text"` がエラーメッセージに含まれ、JSON 出力の `func-details` というキーの下にも含まれます。[`--no-cheating` モード](./tcb.html#no-cheating-mode)で指摘された `assume` 文も、同じように扱われます。これは、形式的でない仕様上の要件（たとえば、望ましい性質を文章で記述したもの）を、検証済みのコードの中の義務と結び付けるのに役立ちます。

[`custom_err`](#verifiercustom_errtext-and-verifiercustom_errtext) と一緒には使えません。

## `#[verifier::prophetic]` {#verifierprophetic}

この属性は、値が*予言に依存する*（prophecy-dependent）、つまり*予言的*（prophetic）であると印を付けるために使います。
これは、その値が、プログラムの未来の値に依存しうることを意味します。たとえば、`final` 演算子（[可変参照](./mutable-references.html)に使われます）や、
[予言変数の値](https://verus-lang.github.io/verus/verusdoc/vstd/proph/struct.ProphecyGhost.html#method.value)です。
この属性は、どの spec の値が予言に依存するかを Verus が追跡するのを助けます。予言に依存する値は、特定の文脈では制限されます。

 * 予言に依存する値は、[decreases 節](./reference-decreases.html)には書けません。
 * 予言に依存する値は、`Ghost` のオペランドには書けません。
 * 予言に依存する値は、tracked モードのゴースト状態の値に影響を与えられません。

これらの制限の根拠は、[こちらで説明されています](https://verus-lang.github.io/verus/verusdoc/vstd/proph/index.html)。

`#[verifier::prophetic]` 属性は、任意の spec 関数、または任意の ghost モードのローカル変数に付けられます。

### spec 関数に付ける場合 {#on-spec-functions}

既定では、どの spec 関数も、明示的に予言的と印を付けない限り、予言的でないとみなされます。
Verus は、本体が予言的なのに予言的と印が付いていない関数を、不正な形（ill-formed）とみなします。

例:

```rust
// Ill-formed: prophetic value not allowed for body of non-prophetic spec function
spec fn future_value_of_mut_ref(a: &mut u64) -> u64 {
    *final(a)
}
```

```rust
// Ok
#[verifier::prophetic]
spec fn future_value_of_mut_ref2(a: &mut u64) -> u64 {
    *final(a)
}
```

さらに、トレイト実装の中の spec 関数に `#[verifier::prophetic]` が付いている場合、
トレイトの宣言でも `#[verifier::prophetic]` が付いていなければなりません。
逆は成り立ちません。トレイトの関数が予言的であっても、実装の関数は予言的でなくてかまいません。

### ローカル変数に付ける場合 {#on-local-variables}

既定では、Verus は、それぞれのローカル変数が予言的かどうかを、初期値が予言的かどうかに基づいて推論します。
ローカル変数は、初期値にかかわらず、明示的に予言的と印を付けられます。

例:

```rust
proof fn test() {
    let ghost mut local_var = 0;
    let tracked proph_var = vstd::proph::ProphecyGhost::<u64>::new();
    local_var = proph_var.value(); // Ill-formed: prophetic value not allowed for assignment to non-prophetic location
}
```

```rust
proof fn test() {
    #[verifier::prophetic]
    let ghost mut local_var = 0;
    let tracked proph_var = vstd::proph::ProphecyGhost::<u64>::new();
    local_var = proph_var.value(); // Ok, `local_var` was explicitly marked prophetic
}
```

```rust
proof fn test(tracked some_int: &mut u64) {
    let ghost mut local_var = *final(some_int);
    let tracked proph_var = vstd::proph::ProphecyGhost::<u64>::new();
    local_var = proph_var.value(); // Ok, `local_var` was inferred as prophetic
}
```

## `#[verifier::custom_err("text")]` と `#![verifier::custom_err("text")]` {#verifiercustom_errtext-and-verifiercustom_errtext}

これらの属性は、`requires` / `ensures` 節、または `assume` / `assert` 文に、独自のエラーメッセージを付けます。

- 外側の属性 `#[verifier::custom_err]` は、`assume` / `assert` 文に付けなければなりません。
- 内側の属性 `#![verifier::custom_err]`（`!` に注意）は、`requires` / `ensures` 節に付けなければなりません。

対応する証明義務が失敗すると、`"text"` が、主に表示されるエラーメッセージになります。`proof_note` と違って、`custom_err` のラベルは、JSON の `func-details` にある proof-note の集合には含まれません。

[`proof_note`](#verifierproof_notetext-and-verifierproof_notetext) と一緒には使えません。

## `#[verifier::rlimit(n)]` と `#[verifier::rlimit(infinity)]` {#verifierrlimitn-and-verifierrlimitinfinity}

`rlimit` オプションは、任意の関数に付けられます。これは、その関数についてソルバーに適用される計算量の上限を設定します。

既定の `rlimit` は 10 です。rlimit は、ソルバーが諦めるまでにかける時間に、おおよそ比例します。
既定の 10 は、およそ 2 秒になるように決められています。

rlimit を `infinity` に設定すると、上限を無くせます。

rlimit は、コマンドラインオプション `--rlimit` でも設定できます。

## `#[verifier::spinoff_prover]` {#verifierspinoff_prover}

与えられた proof 関数または exec 関数の検証を、そのモジュールの残りの部分に使われるものとは別の、
新しい SMT ソルバーのインスタンスで実行します。
詳しくは[別の SMT ソルバーのインスタンスを切り出す](./spinoff_prover.html)を参照してください。

## `#[trigger]` {#trigger}

量化子に対するトリガーグループを、手動で指定するために使います。
詳しくは[トリガーの指定手順](./trigger-annotations.html#selecting-trigger-groups)を参照してください。

Verus のほとんどの属性と違って、これには `verifier::` という接頭辞が要りません。

## `#[verifier::truncate]` {#verifiertruncate}

`#[verifier::truncate]` 属性を式に付けると、範囲外になる as キャストについての
recommends 検査を黙らせられます。

ある整数型から別の整数型へキャストするとき、Verus はふつう、元の値が変換先の型に収まるかどうかの
recommends 検査を挿入します。たとえば、`x` が `u32` で、それを `x as u8` でキャストすると、
Verus は `0 <= x < 256` という recommends 検査を追加します。
しかし、切り捨てこそが*望んでいる*動作である場合もあります。そのようなとき、
`#[verifier::truncate]` を使ってこの意図を伝えると、recommends 検査を抑制できます。

なお、切り捨ての動作を意図している場合でも、この属性は必須ではありません。
この属性の効果は recommends 検査を黙らせることだけで、その検査は、
囲んでいる関数の本体に本当の検証エラーが無ければ、もともと省かれます。

**余談。** 切り捨てを意図している場合、切り捨てについての証明を書くには、[ビットベクトルのソルバーモード](./reference-prover-mode-bit-vector.html)が役に立つことがよくあります。

## `#[verifier::type_invariant]` {#verifiertype_invariant}

spec 関数が、あるデータ型の型不変条件であることを宣言します。[型不変条件](./reference-type-invariants.html)を参照してください。

## `#[verifier::when_used_as_spec]` {#verifierwhen_used_as_spec}

exec 関数の名前を、仕様の文脈で使えると便利なことがあります。
たとえば、関数が引数として `v: Vec<u64>` を取る場合、`v.len()` は exec 関数ですが、
事前条件や事後条件の中で `v.len()` を使えると便利です。こうした近道を自分のコードに加えるには、
実行可能な関数に `#[verifier::when_used_as_spec(your_spec_fn_name)]` 属性を付けます。
これが働くためには、指定した spec 関数（上の例では `your_spec_fn_name` という名前のもの）が、
exec 関数と同じ数・同じ型の引数を取り、同じ戻り値の型を返さなければなりません。

## `#[verifier::exec_allows_no_decreases_clause]` {#verifierexec_allows_no_decreases_clause}

再帰やループを含む `exec` 関数には decreases 節が必要である、という要件を無効にします。関数、モジュール、クレートに付けられ、その中身のすべてに影響します。

## `#[verifier::assume_termination]` {#verifierassume_termination}

`exec` 関数が、`decreases` 節を持たなくても、必ず停止すると仮定します。
現在のところ、これは必要ありません。`exec` の停止性検査は、呼び出し先も停止するかどうかを検査しないからです。
これは検証されていない仮定を持ち込むので、
[no-cheating モード](./tcb.html#no-cheating-mode)では、その関数が（直接または間接に）[信頼されたものとして注釈されて](./tcb.html)いない限り、拒否されます。
{% endraw %}

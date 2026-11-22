{% raw %}
# 量化子のプロファイリング {#quantifier-profiling}

Verus の関数の検証がタイムアウトすることがあります。これは、証明すべき義務がすべて満たされたかどうかをソルバーが判定できなかったことを意味します。あるいは、検証は成功しても、望むより長い時間がかかることがあります。どちらの現象にもよくある原因の 1 つが[量化子（quantifier）](quants.html)です。量化子（とそれに対応するトリガー）の書き方が緩すぎる（つまり、頻繁にトリガーされすぎる）と、SMT ソルバーが生成する事実が多くなりすぎて、効率よく整理できなくなることがあります。自分の Verus コードがこれに当てはまるかどうかを調べるには、組み込みの量化子プロファイラを使えます。

具体例として、次の 3 つの関数が定義されているとします。

```rust
uninterp spec fn f(x: nat, y: nat) -> bool;

uninterp spec fn g(x: nat) -> bool;

uninterp spec fn h(x: nat, y: nat) -> bool;
```

そして、これらを次の証明コードで使います。

```rust
proof fn trigger_forever2()
    requires
        forall|x: nat| g(x),
        forall|x: nat, y: nat| h(x, y) == f(x, y),
        forall|x: nat, y: nat| f(x + 1, 2 * y) && f(2 * x, y + x) || f(y, x) ==> #[trigger] f(x, y),
    ensures
        forall|x: nat, y: nat| x > 2318 && y < 100 ==> h(x, y),
{
    assert(g(4));
}
```

`requires` 節に量化子が 3 つあることに注目してください。1 つ目は `g(x)` でトリガーされ、`g(4)` についてのアサーションを証明するのに役立ちます。2 つ目の量化子は `f(x, y)` と `h(x, y)` の両方でトリガーされ、この 2 つが等しいことを述べています。最後の量化子は、手動で `f(x, y)` をトリガーに指定していますが、そこから似た形の式をさらに 2 つ、つまり `f(x + 1, 2 * y)` と `f(2 * x, y + x)` を導入します。これらはそれぞれ `f` への新しい引数を持つので、3 つ目の量化子が再びトリガーされ、インスタンス化（instantiation）の無限の循環が生まれます。また、こうしたインスタンス化が起きるたびに、2 つ目の量化子もトリガーされることに注意してください。

この例に対して Verus を実行すると、すぐにタイムアウトします。こうなったときは、`--profile` オプションを付けて Verus を実行すると、プロファイラを起動できます。このオプションは `--rlimit 1` と組み合わせることを強くお勧めします。そうすれば、プロファイリングのデータを生成しすぎずに済みます（生成する量が多いほど、解析に時間がかかります）。`--profile` を付けると、検証がタイムアウトしたときにプロファイラが自動的に起動します。検証には成功しているものの遅い関数をプロファイルしたいときは、`--profile-all` オプションを使えます。このオプションは、`--verify-function` オプションと組み合わせて、調べたい関数に対象を絞るとよいでしょう。

上の例に対してプロファイラを実行すると、おおよそ次のような出力が得られます。

```
error: function body check: Resource limit (rlimit) exceeded
  --> ../examples/trigger_loops.rs:66:7
   |
66 | proof fn trigger_forever2()
   |       ^^^^^^^^^^^^^^^^^^^^^

note: Analyzing prover log for (profile rerun) trigger_loops::trigger_forever2 ...

Z3 4.16.0
note: Log analysis complete for (profile rerun) trigger_loops::trigger_forever2

note: Profile statistics for trigger_loops::trigger_forever2

note: Observed 17,963 total instantiations of user-level quantifiers

note: Cost * Instantiations: 2269911826 (Instantiated 8,981 times - 49% of the total, cost 252746) top 1 of 3 user-level quantifiers.
  --> ../examples/trigger_loops.rs:70:93
   |
70 | ...   forall|x: nat, y: nat| f(x + 1, 2 * y) && f(2 * x, y + x) || f(y, x) ==> #[trigger] f(x, y),
   |       ------------------------------------------------------------------------------------^^^^^^^
   |       |
   |       Triggers selected for this quantifier

note: Cost * Instantiations: 397732566 (Instantiated 8,981 times - 49% of the total, cost 44286) top 2 of 3 user-level quantifiers.
  --> ../examples/trigger_loops.rs:69:32
   |
69 |         forall|x: nat, y: nat| h(x, y) == f(x, y),
   |         -----------------------^^^^^^^----^^^^^^^
   |         |
   |         Triggers selected for this quantifier

note: Cost * Instantiations: 3 (Instantiated 1 times - 0% of the total, cost 3) top 3 of 3 user-level quantifiers.
  --> ../examples/trigger_loops.rs:68:24
   |
68 |         forall|x: nat| g(x),
   |         ---------------^^^^
   |         |
   |         Triggers selected for this quantifier

verification results:: 0 verified, 1 errors
error: aborting due to 1 previous error
```

プロファイラは、量化子の性能を 2 つの面から測ります。まず、各量化子が何回インスタンス化されたかという基本的な回数を集めます。次に、各量化子の「コスト」を計算しようとします。量化子のコストは、そのインスタンス化のコストの合計です。インスタンス化 `i` のコストは、おおよそ `1 + sum_{(i, n) \in edges} cost(n) / in-degree(n)` です。ここで各 `n` は、インスタンス化 `i` によって引き起こされたインスタンス化です。言い換えると、インスタンス化 `i` が生み出した項によって、ソルバーが（同じ量化子または別の量化子の）別のインスタンス化 `n` を作った、ということです。このヒューリスティック（経験則）は、インスタンス化そのものがほかの高価なインスタンス化を引き起こすような量化子に、より大きな重みを置こうとするものです。既定では、プロファイラはこの 2 つの指標の積で並べ替えます。

上の例では、最上位の量化子が Verus コードの 3 つ目の量化子であることが分かります。これがまさに問題の元凶です。コストの指標を使うことで、この量化子は 2 つ目の量化子より上に位置づけられます。2 つ目の量化子はインスタンス化の回数こそ同じですが、この場面では実のところ「巻き添えを食っただけ」です。そして、この 2 つの量化子はどちらも 1 つ目の量化子よりはるかに多くインスタンス化されており、1 つ目の量化子が問題の原因ではないことを示しています。もしすべての量化子のインスタンス化の回数が少ないなら、それは、量化子のインスタンス化がソルバーの性能の悪さの根本原因ではないことの表れかもしれません。
{% endraw %}

{% raw %}
# 論理的にアトミックな関数を呼び出す {#calling-a-logically-atomic-function}

論理的にアトミックな関数を呼び出すには、次の構文を使います。

```rs
let py = function(px) atomically loop |update| -> (au)
    invariant ...,
{
    // ...
    // assert atomic pre
    let ay = update(ax);
    // assume atomic post
    // ...
    break/continue
}
```

アトミックな関数呼び出しは、`update` 関数を束縛します。これは、`try_open_atomic_update` マクロの効果を、クライアントに対して表すものです。このマクロが `update` 関数を定義していると考えると分かりやすいでしょう。両者の入力と出力、そして事前条件と事後条件は、ぴったり一致するからです。利用者はこの関数を好きな名前で呼べますが、このガイドでは一貫して、常に `update` と呼ぶことにします。`update` 関数は `proof` モードなので、そのまわりでアトミックな不変条件を開くことができます。`atomically` ブロックの本体も `proof` モードです。

型ヒントを省略できる矢印の構文 `-> (au: AtomicUpdate<...>)` によって、利用者は、作られている最中のアトミック更新に `spec` モードでアクセスできます。これは証明のデバッグに特に役立ちます。`atomically` ブロックのさまざまな場所で、アトミックな事前条件や事後条件を手で表明するのに使えるからです。また、型推論が失敗する場合に、アトミック更新の入力型と出力型を指定する場所にもなります。

## アボートの場合を扱う {#handling-the-abort-case}

ライブラリの関数は、アボートのためにアトミック更新を複数回開くことがあります。そのためクライアントは、必要なリソースを必要な回数だけ提供できることを証明しなければなりません。つまり、アトミックな関数呼び出しはループでなければなりません。このループは、よく使われるループヘッダの節、`invariant_except_breaks`、`invariant`、`ensures` のすべてに対応しています。

出力型に対する `UpdateTry` の実装が示すとおりに、ライブラリによってアトミック更新がコミットされた場合、クライアントは `break` でループを抜けなければなりません。同様に、ライブラリがアトミック更新をアボートした場合、ループは、明示的にであれ暗黙にであれ、`continue` しなければなりません。Verus は、これらの制御フローの構文が正しく使われていることを、追加の検証条件を使って検査します。つまり、`break` と `continue` には、`ay.branch()` がそれぞれ `Commit` と `Abort` に等しいという事前条件が付きます。

このアボートの仕組みが必要ない場合（たとえば、アトミック更新の出力型が `Commit<T>` である場合）は、**`loop` キーワードを取り除いて**、この機能を無効にできます。これは、`atomically` ブロックの最後に `break` を加えることと等価です。ただし、証明の自動化が少し良くなり、一直線のコードでループ不変条件が要らなくなります。

## 不変条件マスク {#invariant-masks}

アトミック更新の外側のマスクは、`atomically` ブロックの中でどの不変条件を開けるかを制限します。つまり、不変条件は、その名前空間が外側のマスクに含まれている場合にだけ開けます。内側のマスクは、`update` 関数の不変条件マスクを指定します。つまり、`update` そのもののまわりでどの不変条件を開けるかを制限します。

最も汎用的なアトミック更新は、`outer_mask any` と `inner_mask none` を持ち、クライアントがアトミックな関数呼び出しの中でどんな不変条件でも開けるようにします。逆に、内側のマスクが外側のマスクの部分集合でない場合、その関数は呼び出せません。

## 一貫して使う例 {#running-examples}

この節では、2 つの例の関数のそれぞれについて、2 種類のクライアントを見ていきます。許可オブジェクトの完全な所有権を持つ *同期的な* クライアントと、アトミックな不変条件を使って許可オブジェクトを複数の並行プロセスと共有する *非同期的な* クライアントです。なお、以下の例のクライアントではすべて、[ループの分離を無効にしています](./reference-attributes.html#verifierloop_isolation)。

2 つの例の関数を *同期的に*、つまり許可オブジェクトの完全な所有権を持った状態で呼び出すには、次のようにします。

```rs
let (var, Tracked(mut perm)) = PAtomicU64::new(3);

reset(&var) atomically |update| {
    perm = Commit::get(update(perm));
};

assert(perm.is_for(var));
assert(perm.points_to(0));
```

```rs
let (var, Tracked(mut perm)) = PAtomicU64::new(3);

let prev = increment(&var) atomically loop |update|
    invariant
        perm.is_for(var),
        perm.points_to(3),
{
    match update(perm) {
        Err((p, _)) => perm = p,
        Ok(p) => { perm = p; break }
    }
};

assert(prev == 3);
assert(perm.is_for(var));
assert(perm.points_to(4));
```

次は、例の関数に対する 2 つの単純な *非同期的な* クライアントです。ここでは、アトミックな不変条件を使って関数を呼び出し、複数のスレッドがこれらの関数を並行に使えるようにしています。この例で使う不変条件は、アトミック変数の値については何も述べません。単に `perm.is_for(var)` を表明するだけです。

```rs
let tracked mut credit = vstd::invariant::create_open_invariant_credit();

reset(&var) atomically |update| {
    open_atomic_invariant!(credit => &inv => perm => {
        perm = Commit::get(update(perm));
    });
};
```

```rs
let tracked mut credit = vstd::invariant::create_open_invariant_credit();

increment(&var) atomically loop |update| {
    let tracked mut spare = None;
    open_atomic_invariant!(credit => &inv => perm => {
        let tracked res = update(perm);
        match res {
            Ok(p) => perm = p,
            Err((p, c)) => {
                perm = p;
                spare = Some(c);
            }
        }
    });

    match spare {
        None => break,
        Some(c) => credit = c,
    }
};
```

最後の例は、`increment` 関数のシグネチャに不変条件のクレジットが必要な理由を示しています。アトミックな関数呼び出しの本体は `proof` モードなので、どちらの非同期的なクライアントの例でも、アトミックな不変条件を開くために、不変条件のクレジットを 1 つ使わなければなりません。`reset` 関数を呼び出すには、不変条件を 1 回開くだけでよいので、クレジットは 1 つあれば足ります。`increment` 関数では、回数に上限の無いループの中で不変条件を開きたいので、不変条件に使えるクレジットが絶えず供給される必要があります。

私たちはこれを、アトミック更新がアボートされるたびに、ライブラリからクライアントへ不変条件のクレジットを渡すことで解決します。クライアントは、アトミックな関数呼び出しの前に、最初のクレジットを 1 つ取得します。このクレジットは不変条件に使われ、アトミック更新がアボートされた場合には補充されるので、次のループの反復で使えます。

ここで `match` 式が 2 つ必要なことに注意してください。`open_atomic_invariant` マクロの中に `break` と `continue` を書くことは許されていないからです。1 つ目の `match` が不変条件を閉じ、2 つ目の `match` がアトミックな関数呼び出しの制御フローを扱います。それでも Verus は十分に賢く、`update` 関数の結果がアトミック更新のコミットを知らせているとき、かつそのときに限って `break` が実行されることを理解します。
{% endraw %}

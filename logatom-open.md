{% raw %}
# 線形化点を発動する {#invoking-the-linearization-point}

論理的にアトミックな関数の定義の中で線形化点を発動するには、アトミックな仕様によって与えられたアトミック更新オブジェクトを「開く」必要があります。そのためには、`try_open_atomic_update` マクロを次のように使います。

```rs
let res_au = try_open_atomic_update!(au, mut? ax => {
    // assume atomic pre
    // ...
    // assert atomic post
    Tracked(ay)
})
```

アトミック更新を開くと、マクロはそのアトミック更新を消費します。マクロ呼び出しの本体の中では、入力 `ax: AX` が与えられ、アトミックな事前条件が分かります。そしてマクロ呼び出しの最後で、出力 `ay: AY` を返し、アトミックな事後条件を証明します。

このマクロは、`Tracked<Result<(), AtomicUpdate<AX, AY, PredType>>>` 型の値を出力します。`UpdateTry` トレイトが示すとおりにアトミック更新がコミットされた場合、アトミック更新は消費され、`Ok(())` が得られます。そうでなく、アトミック更新がアボートされた場合は、渡したのと同じアトミック更新が `Err(au)` として戻ってくるので、後でもう一度開くことができます。

## resolves の検査 {#the-resolves-check}

Verus は、論理的にアトミックな関数の本体に対して、関数が戻るまでにアトミック更新が開かれてコミットされていることの証明を要求します。この要求は、予言変数 `au.resolves()` を使って強制します。最初は、`au.resolves()` の値は分かりません。アトミック更新が `try_open_atomic_update!` マクロによって解決されると、`au.resolves()` の値は `true` になります。そのうえで利用者は、関数が戻るときに `au.resolves()` であることを証明しなければなりません。

`au.resolves()` のほかにも、アトミック更新がコミットされたときに設定する予言が 2 つあります。`au.input()` と `au.output()` です。これらは、それぞれ `ax` と `ay` の最終的な値を保存します。これらは内部で使われ、関数の事後条件が、アトミック更新の入力と出力のリソースにアクセスできるようにします。

アトミック更新の解決が自明でない、より複雑なデータ構造（[「ヘルピング」を示す](logatom.html#notes-on-helping)ものなど）では、AU がコミットされた後で、これらの予言変数の値を手で表明する必要が生じることがあります。

## 不変条件マスク {#invariant-masks}

アトミック更新を開くには、その外側のマスクが現在のスコープで有効でなければなりません。つまり、現在のプログラム地点で、集合 `au.outer_mask()` に含まれるすべての不変条件の名前空間を開けなければなりません。`try_open_atomic_update` マクロの本体の中では、集合 `au.inner_mask()` に含まれる不変条件の名前空間しか開けません。外側のマスクと内側のマスクの差が、ちょうど、クライアントが[アトミックな関数呼び出しの中で](./logatom-call.html)開くことを許される不変条件の集合です。

## 一貫して使う例 {#running-examples}

2 つの例の関数の完全な定義は、次のとおりです。

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
{
    // open atomic update and commit
    try_open_atomic_update!(atomic_update, mut perm => {
        var.store(Tracked(&mut perm), 0);
        Tracked(Commit(perm))
    });

    // verify that the atomic update had been resolved
    assert(atomic_update.resolves());
}
```

ここでは、アトミック更新を開き、アトミック変数の許可オブジェクトを受け取って `mut perm` として束縛し、それを使って store の操作を行います。その後、`try_open_atomic_update` マクロ呼び出しの本体から `Commit(perm)` を返すことで、アトミック更新をコミットします。アトミック更新が無事に解決されたことは、関数の本体の最後にある `assert` で確かめられます。

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
{
    let tracked credit = vstd::invariant::create_open_invariant_credit();
    let tracked mut au = atomic_update;
    let mut curr;

    // open atomic update and abort for inital load
    let wrapped_au = try_open_atomic_update!(au, perm => {
        curr = var.load(Tracked(&perm));
        Tracked(Err((perm, credit)))
    });

    // recover `au` from the returned `Tracked(Err(au))`
    proof { au = wrapped_au.get().tracked_unwrap_err() };

    // compare exchange loop
    loop invariant au == atomic_update {
        let tracked credit = vstd::invariant::create_open_invariant_credit();
        let next = curr.wrapping_add(1);
        let res;

        // open atomic update again
        let maybe_au = try_open_atomic_update!(au, mut perm => {
            res = var.compare_exchange_weak(Tracked(&mut perm), curr, next);

            // dynamically commit or abort atomic update
            // depending on `compare_exchange_weak`
            Tracked(match res {
                Ok(_) => Ok(perm),
                Err(_) => Err((perm, credit)),
            })
        });

        match res {
            Ok(_) => return curr,
            Err(new) => {
                proof { au = maybe_au.get().tracked_unwrap_err() };
                curr = new;
            },
        }
    }
}
```

この例では、アトミック更新のアボートの場合を使って、許可オブジェクトを変更することなく、一時的にアクセスできるようにしています。これによって、最初の `load` と、失敗したすべての `compare_exchange_weak` の操作を行えます。これらを実行するには、許可へのアクセスが必要だからです。最初の `load` では、アトミック更新を開いて、無条件にアボートします。一方 `compare_exchange_weak` では、操作が成功したかどうかを調べて、アトミック更新をコミットすべきかアボートすべきかを決めます。

なお、Verus がこの関数の定義を受け入れるには、関数またはモジュールに、次の 2 つの verifier 属性を設定する必要があります。
- `#[verifier::exec_allows_no_decreases_clause]` は、`decreases` 節で停止性を証明しなくても、exec モードで回数に上限の無いループを使えるようにします。
- `#[verifier::loop_isolation(false)]` は、ループのまわりでの証明の自動化を少し良くし、ループ不変条件をかなり単純にできるようにします。
{% endraw %}

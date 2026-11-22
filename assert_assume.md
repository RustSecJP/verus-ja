{% raw %}
# assert と assume を使って証明を作り上げる {#using-assert-and-assume-to-develop-proofs}

[前の章](./spec_lib.html)では、次のような証明の骨組みから始めました。

```rust
pub proof fn lemma_len_intersect<A>(s1: Set<A>, s2: Set<A>)
    ensures
        s1.intersect(s2).len() <= s1.len(),
    decreases
        s1.len(),
{
    if s1.is_empty() {

    } else {
        let a = s1.choose();

        lemma_len_intersect(s1.remove(a), s2);
    }
}
```

そして、欠けていた重要な手順を埋めて、証明を完成させました。
しかしそこでは、どの重要な手順が欠けているのかを、どうやって見つければよいのかは述べていませんでした。
実際には、この種の証明を埋めるには、いくらか試行錯誤が必要です。

この節では、上の証明の骨組みを出発点にして、証明を作り上げていく典型的な過程をたどります。
この過程は、Verus と SMT ソルバーへの一連の問い合わせからなります。`assert` と `assume` を使って質問をし、その答えをもとに、検証が失敗する原因を絞り込んでいきます。

上の証明を実行すると、Verus はエラーを報告します。

```
error: postcondition not satisfied
   |
   |           s1.intersect(s2).len() <= s1.len(),
   |           ---------------------------------- failed this postcondition
```

ここから、2 つの疑問が浮かびます。

- なぜこの事後条件（postcondition）は失敗するのでしょうか。
- この事後条件が成功したら、関数全体の検証は成功するのでしょうか。

まず 2 つ目の疑問を確かめましょう。単に事後条件を仮定して、どうなるかを見ればよいのです。

```rust
pub proof fn lemma_len_intersect<A>(s1: Set<A>, s2: Set<A>)
    ...
{
    if s1.is_empty() {
    } else {
        let a = s1.choose();
        lemma_len_intersect::<A>(s1.remove(a), s2);
    }
    assume(s1.intersect(s2).len() <= s1.len());
}
```

この場合、検証は成功します。

```
verification results:: verified: 1 errors: 0
```

このコードには 2 つの経路があります。`s1.is_empty()` のときの経路と、`!s1.is_empty()` のときの経路です。
失敗は、どちらか一方の経路にあるかもしれませんし、両方にあるかもしれません。
`assume` の写しを分岐ごとに 1 つずつ移して、`if`/`else` の各分岐に別々に取り組む準備をしましょう。

```rust
{
    if s1.is_empty() {
        assume(s1.intersect(s2).len() <= s1.len());
    } else {
        let a = s1.choose();
        lemma_len_intersect::<A>(s1.remove(a), s2);
        assume(s1.intersect(s2).len() <= s1.len());
    }
}
```
```
verification results:: verified: 1 errors: 0
```

次に、1 つ目の `assume` を `assert` に変えて、`if` の分岐で成功するかどうかを見てみましょう。

```rust
{
    if s1.is_empty() {
        assert(s1.intersect(s2).len() <= s1.len());
    } else {
        let a = s1.choose();
        lemma_len_intersect::<A>(s1.remove(a), s2);
        assume(s1.intersect(s2).len() <= s1.len());
    }
}
```

```
error: assertion failed
   |
   |         assert(s1.intersect(s2).len() <= s1.len());
   |                ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ assertion failed
```

`s1.is_empty()` の場合、`s1.len() == 0` のはずです（空集合の濃度は 0 です）。
これは、簡単な表明で念のため確かめられます。

```rust
{
    if s1.is_empty() {
        assert(s1.len() == 0);
        assume(s1.intersect(s2).len() <= s1.len());
    } else {
        ...
    }
}
```
```
verification results:: verified: 1 errors: 0
```

すると、必要なのは `s1.intersect(s2).len() <= 0` です。
これが真であれば、ここで事後条件を満たせます。

```rust
{
    if s1.is_empty() {
        assume(s1.intersect(s2).len() <= 0);
        assert(s1.intersect(s2).len() <= s1.len());
    } else {
        ...
    }
}
```
```
verification results:: verified: 1 errors: 0
```

集合の濃度は `nat` なので、それが `<= 0` になり得るのは、`0` に等しいときだけです。

```rust
{
    if s1.is_empty() {
        assume(s1.intersect(s2).len() == 0);
        assert(s1.intersect(s2).len() <= s1.len());
    } else {
        ...
    }
}
```
```
verification results:: verified: 1 errors: 0
```

そして、それが `0` になり得るのは、その集合が空集合のときだけです。

```rust
{
    if s1.is_empty() {
        assume(s1.intersect(s2) == Set::<A>::empty());
        assert(s1.intersect(s2).len() == 0);
        assert(s1.intersect(s2).len() <= s1.len());
    } else {
        ...
    }
}
```
```
verification results:: verified: 1 errors: 0
```

これで問題を絞り込めました。
空集合 `s1` と別の集合との共通部分は空集合に等しいはずですが、検証器にはそれが自動的には分からないのです。
その理由は、前の節の[等価性についての議論](spec_lib.html#proving-properties-of-seq-set-iset-map-and-imap)から推測できます。
SMT ソルバーは、コレクション同士の等価性をいつも自動的に証明してくれるわけではなく、外延性（extensionality）を使って等価性を表明するよう求めてくるのです。
そこで、外延性の表明を追加します。

```rust
{
    if s1.is_empty() {
        assert(s1.intersect(s2) =~= Set::<A>::empty());
        assert(s1.intersect(s2).len() == 0);
        assert(s1.intersect(s2).len() <= s1.len());
    } else {
        ...
    }
}
```
```
verification results:: verified: 1 errors: 0
```

うまくいきました。これで `s1.is_empty()` の場合を検証できたので、`!s1.is_empty()` の場合に目を向けましょう。

```rust
{
    if s1.is_empty() {
        ...
    } else {
        let a = s1.choose();
        lemma_len_intersect::<A>(s1.remove(a), s2);
        assume(s1.intersect(s2).len() <= s1.len());
    }
}
```

この `assume` を `assert` に変えると失敗するので、こちらの場合にもやるべきことがあります。

```rust
{
    if s1.is_empty() {
        ...
    } else {
        let a = s1.choose();
        lemma_len_intersect::<A>(s1.remove(a), s2);
        assert(s1.intersect(s2).len() <= s1.len());
    }
}
```
```
error: assertion failed
   |
   |         assert(s1.intersect(s2).len() <= s1.len());
   |                ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ assertion failed
```

幸い、再帰呼び出し `lemma_len_intersect::<A>(s1.remove(a), s2)` は成功しているので、この呼び出しの事後条件からいくらか情報が得られます。
もっと詳しく調べられるように、`s1` を `s1.remove(a)` に置き換えて、これを明示的に書き出してみましょう。

```rust
{
    if s1.is_empty() {
        ...
    } else {
        let a = s1.choose();
        lemma_len_intersect::<A>(s1.remove(a), s2);
        assert(s1.remove(a).intersect(s2).len() <= s1.remove(a).len());

        assume(s1.intersect(s2).len() <= s1.len());
    }
}
```
```
verification results:: verified: 1 errors: 0
```

`s1.remove(a)` について分かっていることと、`s1` について証明しようとしていることを比べてみましょう。

```rust
        assert(s1.remove(a).intersect(s2).len() <= s1.remove(a).len()); // WE KNOW THIS

        assume(s1          .intersect(s2).len() <= s1          .len()); // WE WANT THIS
```

分かっていることを、求めているものにもっと近い形にする方法はないでしょうか。
たとえば、`s1.remove(a).len()` は `s1.len()` とどう関係しているでしょうか。
値 `a` は `s1` の要素なので、`s1` から取り除けば、濃度は 1 だけ減るはずです。

```rust
{
    if s1.is_empty() {
        ...
    } else {
        let a = s1.choose();
        lemma_len_intersect::<A>(s1.remove(a), s2);
        assert(s1.remove(a).intersect(s2).len() <= s1.remove(a).len());
        assert(s1.remove(a).len() == s1.len() - 1);

        assume(s1.intersect(s2).len() <= s1.len());
    }
}
```
```
verification results:: verified: 1 errors: 0
```

これで、少し簡単にできます。

```rust
{
    if s1.is_empty() {
        ...
    } else {
        let a = s1.choose();
        lemma_len_intersect::<A>(s1.remove(a), s2);
        assert(s1.remove(a).intersect(s2).len() <= s1.remove(a).len());
        assert(s1.remove(a).intersect(s2).len() <= s1.len() - 1);
        assert(s1.remove(a).intersect(s2).len() + 1 <= s1.len());

        assume(s1.intersect(s2).len() <= s1.len());
    }
}
```
```
verification results:: verified: 1 errors: 0
```

残る欠けた部分は、`s1.remove(a).intersect(s2).len() + 1` と `s1.intersect(s2).len()` の関係です。

```rust
{
    if s1.is_empty() {
        ...
    } else {
        let a = s1.choose();
        lemma_len_intersect::<A>(s1.remove(a), s2);
        assert(s1.remove(a).intersect(s2).len() <= s1.remove(a).len());
        assert(s1.remove(a).intersect(s2).len() <= s1.len() - 1);
        assert(s1.remove(a).intersect(s2).len() + 1 <= s1.len());

        assume(s1.intersect(s2).len() <= s1.remove(a).intersect(s2).len() + 1);

        assert(s1.intersect(s2).len() <= s1.len());
    }
}
```
```
verification results:: verified: 1 errors: 0
```

仮定 `s1.intersect(s2).len() <= s1.remove(a).intersect(s2).len() + 1` を証明できれば、完成です。

```rust
        assume(s1          .intersect(s2).len()
            <= s1.remove(a).intersect(s2).len() + 1);
```

この不等式を証明しやすくするために、`s1.remove(a).intersect(s2)` を `s1.intersect(s2)` にもっと近い形にする方法はないでしょうか。
順序を `s1.remove(a).intersect(s2)` から `s1.intersect(s2).remove(a)` に入れ替えれば、部分式 `s1.intersect(s2)` が一致します。

```rust
        assume(s1.intersect(s2)          .len()
            <= s1.intersect(s2).remove(a).len() + 1);
```

では、それを試してみましょう。

```rust
{
    if s1.is_empty() {
        ...
    } else {
        let a = s1.choose();
        lemma_len_intersect::<A>(s1.remove(a), s2);
        assert(s1.remove(a).intersect(s2).len() <= s1.remove(a).len());
        assert(s1.remove(a).intersect(s2).len() <= s1.len() - 1);
        assert(s1.remove(a).intersect(s2).len() + 1 <= s1.len());

        assert(s1.intersect(s2).len() <= s1.intersect(s2).remove(a).len() + 1);
        assert(s1.intersect(s2).len() <= s1.remove(a).intersect(s2).len() + 1);

        assert(s1.intersect(s2).len() <= s1.len());
    }
}
```
```
error: assertion failed
   |
   |         assert(s1.intersect(s2).len() <= s1.remove(a).intersect(s2).len() + 1);
   |                ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ assertion failed
```

これらの表明の一方は成功し、もう一方は失敗します。
成功した表明と失敗した表明の違いは、`s1.intersect(s2).remove(a)` と `s1.remove(a).intersect(s2)` における `intersect` と `remove` の順序だけです。
したがって、証明を完成させるのに必要なのは、`s1.intersect(s2).remove(a)` が `s1.remove(a).intersect(s2)` に等しいことだけです。

```rust
{
    if s1.is_empty() {
        ...
    } else {
        let a = s1.choose();
        lemma_len_intersect::<A>(s1.remove(a), s2);
        assert(s1.remove(a).intersect(s2).len() <= s1.remove(a).len());
        assert(s1.remove(a).intersect(s2).len() <= s1.len() - 1);
        assert(s1.remove(a).intersect(s2).len() + 1 <= s1.len());

        assert(s1.intersect(s2).len() <= s1.intersect(s2).remove(a).len() + 1);
        assume(s1.intersect(s2).remove(a) == s1.remove(a).intersect(s2));
        assert(s1.intersect(s2).len() <= s1.remove(a).intersect(s2).len() + 1);

        assert(s1.intersect(s2).len() <= s1.len());
    }
}
```
```
verification results:: verified: 1 errors: 0
```

またしても、2 つのコレクションが等しいことを知る必要が出てきました。
そしてまたしても、最初に試すべきことは、外延的等価性（extensional equality）を表明することです。

```rust
{
    if s1.is_empty() {
        ...
    } else {
        let a = s1.choose();
        lemma_len_intersect::<A>(s1.remove(a), s2);
        assert(s1.remove(a).intersect(s2).len() <= s1.remove(a).len());
        assert(s1.remove(a).intersect(s2).len() <= s1.len() - 1);
        assert(s1.remove(a).intersect(s2).len() + 1 <= s1.len());

        assert(s1.intersect(s2).len() <= s1.intersect(s2).remove(a).len() + 1);
        assert(s1.intersect(s2).remove(a) =~= s1.remove(a).intersect(s2));
        assert(s1.intersect(s2).len() <= s1.remove(a).intersect(s2).len() + 1);

        assert(s1.intersect(s2).len() <= s1.len());
    }
}
```
```
verification results:: verified: 1 errors: 0
```

うまくいきました。
これですべての `assume` を取り除けたので、検証は完了です。
紛れ込んだ `assume` が残っていないことを保証するために、Verus にはそれを強制する[no-cheating モード](./tcb.html#no-cheating-mode)があります。

完成した証明は次のとおりです。

```rust
pub proof fn lemma_len_intersect<A>(s1: Set<A>, s2: Set<A>)
    requires
        s1.finite(),
    ensures
        s1.intersect(s2).len() <= s1.len(),
    decreases
        s1.len(),
{
    if s1.is_empty() {
        assert(s1.intersect(s2) =~= Set::empty());
        assert(s1.intersect(s2).len() == 0);
        assert(s1.intersect(s2).len() <= s1.len());
    } else {
        let a = s1.choose();
        lemma_len_intersect::<A>(s1.remove(a), s2);
        assert(s1.remove(a).intersect(s2).len() <= s1.remove(a).len());
        assert(s1.remove(a).intersect(s2).len() <= s1.len() - 1);
        assert(s1.remove(a).intersect(s2).len() + 1 <= s1.len());

        assert(s1.intersect(s2).len() <= s1.intersect(s2).remove(a).len() + 1);
        assert(s1.intersect(s2).remove(a) =~= s1.remove(a).intersect(s2));
        assert(s1.intersect(s2).len() <= s1.remove(a).intersect(s2).len() + 1);

        assert(s1.intersect(s2).len() <= s1.len());
    }
}
```
```
verification results:: verified: 1 errors: 0
```

ただし、上のコードには不要な `assert` がたくさん含まれています。
将来このコードを保守しなければならない人のために、数分かけてコードを整理する価値があります。
保守するコードを減らすために不要なコードは取り除きたいのですが、コードを保守する人が理解できるだけの情報は残しておきたいところです。
適切な情報の量は好みの問題ですが、簡潔さと分かりやすさのあいだで、ほどよい釣り合いを取るよう努めることはできます。

```rust
pub proof fn lemma_len_intersect<A>(s1: Set<A>, s2: Set<A>)
    ensures
        s1.intersect(s2).len() <= s1.len(),
    decreases s1.len(),
{
    if s1.is_empty() {
        assert(s1.intersect(s2).len() == 0) by {
            assert(s1.intersect(s2) =~= s1);
        }
    } else {
        let a = s1.choose();
        lemma_len_intersect(s1.remove(a), s2);
        // by induction: s1.remove(a).intersect(s2).len() <= s1.remove(a).len()
        assert(s1.intersect(s2).remove(a).len() <= s1.remove(a).len()) by {
            assert(s1.intersect(s2).remove(a) =~= s1.remove(a).intersect(s2));
        }
        // simplifying ".remove(a).len()" yields s1.intersect(s2).len() <= s1.len())

    }
}
```
{% endraw %}

{% raw %}
# 再帰とループ {#recursion-and-loops}

n 番目の[三角数](https://en.wikipedia.org/wiki/Triangular_number)を計算したいとします。

```
triangle(n) = 0 + 1 + 2 + ... + (n - 1) + n
```

これは、単純な再帰関数として表せます。

```rust
spec fn triangle(n: nat) -> nat
    decreases n,
{
    if n == 0 {
        0
    } else {
        n + triangle((n - 1) as nat)
    }
}

```

この章では、再帰関数の定義のしかたと使い方を説明します。
`decreases` 節の書き方や、燃料（fuel）の使い方も含みます。
続いて、検証された `triangle` の実装をいくつか順に見ていきます。
基本的な再帰による実装から始めて、while ループによる実装で終わります。
{% endraw %}

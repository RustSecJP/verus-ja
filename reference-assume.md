{% raw %}
# `assume` {#assume}

<!-- TODO: assume があるとき、その仮定が成り立つプログラム実行の部分集合については保証が残るのかどうかを決める必要がある。 -->

> [!CAUTION]
> `assume` 文は検査されないため、Verus の保証を簡単にすり抜けられてしまいます。
> 特に、プログラムに `assume` 文が 1 つでも含まれている場合、Verus による「検証」が成功しても、
> そのプログラムについては**何の保証もありません**。ただし、それらの `assume` 文が、
> 原理的には成功する `assert` 文に置き換えられる場合は別です。
>
> `assume` 文がもっとも役に立つのは、開発の*途中*の段階です。
> たとえば、[assert / assume を使って証明を組み立てていく進め方](./assert_assume.html)の中で使います。

> [!TIP]
> `--no-cheating` フラグを使うと、`assume` 文を禁止できます。

### 構文 {#syntax}

```verus-grammar
V@[assume] ::= assume (V@[spec_expr]) ;
```

### 証明上の動作 {#proof-operation}

与えられた述語を、証明なしで仮定します。
{% endraw %}

{% raw %}
# Rust のサブセット {#rust-subset}

spec 言語の大部分は Rust 言語のサブセットのように見えますが、微妙な違いがいくつかあります。

### 関数呼び出し {#function-calls}

**構文:**

```verus-grammar
V@[fn_call_expr] ::= R@[path] ( V@[spec_expr],* )
```

許されるのは、純粋な関数の呼び出しだけです（つまり、ほかの `spec` 関数の呼び出しか、`when_used_as_spec` ディレクティブが付いた関数の呼び出しです）。

### let による代入 {#let-assignment}

**構文:**

```verus-grammar
V@[spec_block_expr] ::= { V@[spec_let_stmt]* V@[spec_expr] }
V@[spec_let_stmt]   ::= let R@[pattern] = V@[spec_expr];
```

spec 式は `let` 束縛に対応していますが、`let mut` 束縛には対応していません。

### if / if let / match 式 {#if--if-let--match-expressions}

**構文:**

```verus-grammar
V@[if_expr]     ::= if V@[spec_expr] { V@[spec_expr] } else { V@[spec_expr] }
V@[if_let_expr] ::= if let R@[pattern] = V@[spec_expr] { V@[spec_expr] } else { V@[spec_expr] }
V@[match_expr]  ::= match R@[expr] { R@[match_arms] }
```

これらは通常どおりに動作します。

### `&&` と `||`
{: id="-and-"}

**構文:**

```verus-grammar
V@[and_expr] ::= V@[spec_expr] && V@[spec_expr]
V@[or_expr]  ::= V@[spec_expr] || V@[spec_expr]
V@[not_expr] ::= ! V@[spec_expr]
```

これらは通常どおりに動作します。ただし、すべての spec 式は純粋で副作用が無いので、「短絡評価」という概念はありません。

### 等価性（==） {#equality-}

**構文:**

```verus-grammar
V@[equality_expr] ::= V@[spec_expr] == V@[spec_expr]
                | V@[spec_expr] != V@[spec_expr]
```

これは exec モードの `==` と同じものではありません。[`==` についての詳細](./spec-equality.html)を参照してください。

### 算術 {#arithmetic}

**構文:**

```verus-grammar
V@[arith_expr] ::= V@[spec_expr] \+ V@[spec_expr]
             | V@[spec_expr] - V@[spec_expr]
             | - V@[spec_expr]
             | V@[spec_expr] \* V@[spec_expr]
             | V@[spec_expr] / V@[spec_expr]
             | V@[spec_expr] % V@[spec_expr]
```

算術は、Verus の `int` 型と `nat` 型を扱えるように、少し違った動作をします。[算術についての詳細](./spec-arithmetic.html)を参照してください。

### 参照（&T） {#references-t}

**構文:**

```verus-grammar
V@[ref_expr]   ::= & V@[spec_expr]
V@[deref_expr] ::= \* V@[spec_expr]
```

Verus は、spec モードでは `Box` と参照をできるかぎり無視しようとします。
それでも Rust の型検査器を満足させる必要はあるので、検査を通すために参照（`&`）や参照外し（`*`）を挿入しなければならないことがあります。
ただし、Verus はこれらの操作を無視します。

### Box {#box}

**構文:**

```verus-grammar
V@[box_new_expr] ::= Box::new( V@[spec_expr] )
```

Verus は `Box<T>` を、`Box::new(x)` や `*box` のようなボックスの操作とともに特別扱いしているので、これらは spec モードで使えます。
参照と同じく、これらの操作は無視されますが、役に立つことはよくあります。
たとえば、再帰的な型を作るには Rust の基本的な検査を通す必要があり、そのためにはしばしば `Box` を使うことになります。
{% endraw %}

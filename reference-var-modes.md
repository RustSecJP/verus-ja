{% raw %}
# 変数のモード {#variable-modes}

Verus には、3 つの関数のモードに加えて、3 つの _変数の_ モードがあります。`exec`、`tracked`、`ghost` です。
コンパイル後のコードに存在するのは `exec` 変数だけで、`ghost` 変数と `tracked` 変数は、コンパイル後のコードから「消去」されます。

モードという概念の入門については、[このチュートリアルのページ](./modes.html)を参照してください。
tracked モードは高度な機能で、[並行性のガイド](https://verus-lang.github.io/verus/state_machines/intro.html)で詳しく説明されています。

## 変数のモードと関数のモード {#variable-modes-and-function-modes}

どの変数が許されるかは、式のモードによって決まります。次の表のとおりです。

|            | 既定の変数のモード    | `ghost` 変数      | `tracked` 変数      | `exec` 変数      |
|------------|-----------------------|-------------------|---------------------|------------------|
| spec コード  | `ghost`               | 可                |                     |                  |
| proof コード | `ghost`               | 可                | 可                  |                  |
| exec コード  | `exec`                | 可                | 可                  | 可               |

`exec` コードではどのモードの変数も許されますが、いくつか制限があります。下記を参照してください。

## `proof` 関数から `tracked` 変数と `ghost` 変数を使う {#using-tracked-and-ghost-variables-from-a-proof-function}

既定では、proof 関数の中の変数はどれも `ghost` モードです。
引数、変数、戻り値には tracked の印を付けられます。例を示します。

```rust
proof fn some_proof_fn(tracked param: Foo) -> (tracked ret: RetType) {
    let tracked x = ...;
}
```

戻り値については、`tracked` キーワードは戻り値の型全体にしか適用できません。
たとえば、タプルの個々の要素に `tracked` を選んで適用することはできません。

tracked なデータと ghost なデータを組み合わせて使うには、いくつかの方法があります。
1 つ目は、`tracked` の印を付けた構造体を作り、個々のフィールドに `ghost` か `tracked` の印を付ける方法です。

2 つ目は、Verus の組み込みライブラリにある `Tracked` 型と `Ghost` 型を使って、`(Tracked<X>, Ghost<Y>)` のようなタプルを作る方法です。
これらの型はパターンマッチに対応しています。

```rust
proof fn some_call() -> (tracked ret: (Tracked<X>, Ghost<Y>)) { ... }

proof fn example() {
    // The lower-case `tracked` keyword is used to indicate the right-hand side
    // has `proof` mode, in order to allow the `tracked` call.
    // The upper-case `Tracked` and `Ghost` are used in the pattern matching to unwrap
    // the `X` and `Y` objects.
    let tracked (Tracked(x), Ghost(y)) = some_call();
}
```

## `exec` 関数から `tracked` 変数と `ghost` 変数を使う {#using-tracked-and-ghost-variables-from-an-exec-function}

`exec` コードの中の変数には、`tracked` か `ghost` の印を付けられます。
これらの変数は、コードがコンパイルされるときに消去されます。
ただし、いくつか制限があります。
具体的には、`tracked` や `ghost` の印を付けた変数は、`exec` ブロックの中のどこでも宣言できます。
しかし、そうした変数への _代入_ は、`proof { ... }` ブロックの中からしか行えません。

```rust
fn some_exec_fn() {
    let ghost mut x = 5; // this is allowed

    proof {
        x = 7; // this is allowed
    }

    x = 9; // this is not allowed
}
```

さらに、次の制限があります。

 * `exec` 関数の引数と戻り値は、`exec` モードでなければなりません。

 * `exec` 構造体のフィールドは、`exec` モードでなければなりません。

これらを回避するために、プログラムでは `Tracked` 型と `Ghost` 型を使えます。
proof コードの場合と同じく、Verus はこれらの型のパターンマッチに対応しています。

```rust
exec fn example() {
    // Because of the keyword `tracked`, Verus interprets the right-hand side
    // as if it were in a `proof` block.
    let tracked (Tracked(x), Ghost(y)) = some_call();
}
```

`Tracked` 型や `Ghost` 型を通じて渡さなければならない引数を扱うには、パターンマッチで中身を取り出せます。

```rust
exec fn example(Tracked(x): Tracked<X>, Ghost(y): Ghost<Y>) {
    // Use `x` as if it were declared `let tracked x`
    // Use `y` as if it were declared `let ghost y`
}
```

## 早見表 {#cheat-sheet}

### proof 関数で、tracked または ghost の引数を取る {#proof-function-take-tracked-or-ghost-param}

```rust
proof fn example(tracked x: X, ghost y: Y)
```

この関数を proof コードから呼び出すには、次のようにします。

```rust
proof fn test(tracked x: X, ghost y: Y) {
    example(x, y);
}
```

この関数を exec コードから呼び出すには、次のようにします。

```rust
fn test() {
    let tracked x = ...;
    let ghost y = ...;

    // From a proof block:
    proof { example(x, y); }
}
```

### proof 関数で、ghost の値を返す {#proof-function-return-ghost-param}

```rust
proof fn example() -> (ret: Y)
```

この関数を proof コードから呼び出すには、次のようにします。

```rust
proof fn test() {
    let y = example();
}
```

この関数を exec コードから呼び出すには、次のようにします。

```rust
fn test() {
    let ghost y = example();
}
```

### proof 関数で、tracked の値を返す {#proof-function-return-tracked-param}

```rust
proof fn example() -> (tracked ret: X)
```

この関数を proof コードから呼び出すには、次のようにします。

```rust
proof fn test() {
    let tracked y = example();
}
```

この関数を exec コードから呼び出すには、次のようにします。

```rust
fn test() {
    // In a proof block:
    proof { let tracked y = example(); }

    // Or outside a proof block:
    let tracked y = example();
}
```

### proof 関数で、ghost の値と tracked の値の両方を返す {#proof-function-return-both-a-ghost-param-and-tracked-param}

```rust
proof fn example() -> (tracked ret: (Tracked<X>, Ghost<Y>))
```

この関数を proof コードから呼び出すには、次のようにします。

```rust
proof fn test() {
    let tracked (Tracked(x), Ghost(y)) = example();
}
```

この関数を exec コードから呼び出すには、次のようにします。

```rust
fn test() {
    // In a proof block:
    proof { let tracked (Tracked(x), Ghost(y)) = example(); }

    // or outside a proof block:
    let tracked (Tracked(x), Ghost(y)) = example();
}
```

### exec 関数で、tracked の引数と ghost の引数を取る {#exec-function-take-a-tracked-and-ghost-parameter}

```rust
fn example(Tracked(x): Tracked<X>, Ghost(y): Ghost<Y>)
```

この関数を exec コードから呼び出すには、次のようにします。

```rust
fn test() {
    let tracked x = ...;
    let ghost y = ...;

    example(Tracked(x), Ghost(y));
}
```

### exec 関数で、tracked の値と ghost の値を返す {#exec-function-return-a-tracked-and-ghost-value}

```rust
fn example() -> (Tracked<X>, Ghost<Y>)
```

この関数を exec コードから呼び出すには、次のようにします。

```rust
fn test() {
    let (Tracked(x), Ghost(y)) = example();
}
```

### exec 関数で、可変参照である tracked の引数を取る {#exec-function-take-a-tracked-parameter-that-is-a-mutable-reference}

```rust
fn example(Tracked(x): Tracked<&mut X>)
```

この関数を exec コードから呼び出すには、次のようにします。

```rust
fn test() {
    let tracked mut x = ...;

    example(Tracked(&mut x));
}
```
{% endraw %}

{% raw %}
# exec 関数のシグネチャ {#exec-fn-signature}

`exec` 関数のシグネチャの一般形は、次のとおりです。

```verus-grammar
V@[exec_fn_item] ::=
    R@[visibility]? exec? fn R@[function_name] R@[generics]?(R@[args...]) ( -> V@[exec_return_type] )?
        R@[where_clause]?
        V@[requires_clause]?
        V@[ensures_clause]?
        V@[returns_clause]?
        V@[invariants_clause]?
        V@[unwind_clause]?

V@[exec_return_type]       ::= V@[exec_return_type_named] | V@[exec_return_type_anon]
V@[exec_return_type_named] ::= ( R@[pattern] : R@[type] )
V@[exec_return_type_anon]  ::= R@[type]
```

## 関数の仕様 {#function-specification}

関数の仕様を構成する要素は、シグネチャの各節で与えられます。

**事前条件。**
V@[requires_clause] が事前条件（precondition）です。

**事後条件。**
V@[ensures_clause] と V@[returns_clause] が、合わせて事後条件（postcondition）になります。

**不変条件。**
V@[invariants_clause] は、その関数がどの不変条件を開けるかを指定します。
exec 関数では、既定は `opens_invariants any` です。
詳しくは[このページ](./reference-opens-invariants.html)を参照してください。

**巻き戻し。**
V@[unwind_clause] は、その関数が巻き戻し（unwind）によって「異常に」終了することがあるかどうか、
また、それがどんな条件で起こりうるかを指定します。
詳しくは[このページ](./reference-unwind-sig.html)を参照してください。

## 関数の引数 {#function-arguments}

すべての引数と戻り値は、`exec` モードでなければなりません。ghost / tracked の変数をシグネチャに埋め込むには、`Tracked` 型と `Ghost` 型を使います。

詳しくは[こちら](./reference-var-modes.html#cheat-sheet)を参照してください。
{% endraw %}

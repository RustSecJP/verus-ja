{% raw %}
# proof 関数のシグネチャ {#proof-fn-signature}

`proof` 関数のシグネチャの一般形は、次のとおりです。

```verus-grammar
V@[proof_fn_item] ::= V@[proof_fn_proved] | V@[proof_fn_axiom]

V@[proof_fn_proved] ::=
    R@[visibility]? broadcast? proof fn R@[function_name] R@[generics]?(R@[args...]) ( by(function_prover_mode) )? ( -> V@[proof_return_type] )?
        R@[where_clause]?
        V@[requires_clause]?
        V@[ensures_clause]?
        V@[returns_clause]?
        V@[invariants_clause]?
        V@[decreases_clause]?
    { V@[proof_stmt]* }

V@[proof_fn_axiom] ::=
    R@[visibility]? broadcast? axiom fn R@[function_name] R@[generics]?(R@[args...]) ( -> V@[proof_return_type] )?
        R@[where_clause]?
        V@[requires_clause]?
        V@[ensures_clause]?
        V@[returns_clause]?
        V@[invariants_clause]?
        V@[decreases_clause]?
        ;

V@[function_prover_mode]    ::= integer_ring | bit_vector | nonlinear_arith

V@[proof_return_type]       ::= V@[proof_return_type_named] | V@[proof_return_type_anon]
V@[proof_return_type_named] ::= ( tracked? R@[pattern] : R@[type] )
V@[proof_return_type_anon]  ::= R@[type]
```

## 関数の仕様 {#function-specification}

関数の仕様を構成する要素は、シグネチャの各節で与えられます。

**事前条件。**
V@[requires_clause] が事前条件（precondition）です。

**事後条件。**
V@[ensures_clause]
と
V@[returns_clause]
が、合わせて事後条件（postcondition）になります。

**不変条件。**
V@[invariants_clause] は、その関数がどの不変条件を開けるかを指定します。
proof 関数では、既定は `opens_invariants none` です。
詳しくは[このページ](./reference-opens-invariants.html)を参照してください。

## 関数の引数 {#function-arguments}

すべての引数と戻り値は、`ghost` モードまたは `tracked` モードでなければなりません。
引数は既定では `ghost` で、`tracked` キーワードを付けると `tracked` として宣言できます。

詳しくは[こちら](./reference-var-modes.html#cheat-sheet)を参照してください。

## 関数の証明器モード {#function-prover-mode}

V@[function_prover_mode] が与えられている場合、証明は指定されたソルバー
（[`integer_ring`](./reference-prover-mode-integer-ring.html)、
[`bit_vector`](./reference-prover-mode-bit-vector.html)、
または [`nonlinear_arith`](./reference-prover-mode-nonlinear.html)）
に回されます。
{% endraw %}

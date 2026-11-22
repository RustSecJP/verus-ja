{% raw %}
# returns {#returns}

`returns` 節は、ある決まった形の `ensures` 節に対する糖衣構文（syntactic sugar）です。
`returns` 節は、`ensures` 節の代わりに書くことも、`ensures` 節に加えて書くこともできます。

次のコードは、

```rust
fn example() -> return_type
    returns $expr
{
  ...
}
```

次のコードと等価です。

```rust
fn example() -> (return_name: return_type)
    ensures return_name == $expr
{
  ...
}
```

## `#[verifier::allow_in_spec]` 属性と組み合わせる {#with-the-verifierallow_in_spec-attribute}

[`#[verifier::allow_in_spec]` 属性](./reference-attributes.html#verifierallow_in_spec)は、[`returns` 節](./reference-returns.html)を持つ実行可能な関数に付けられます。これにより、その関数を spec モードで使えるようになります。spec モードでは、その関数は、指定された戻り値と等価なものとして解釈されます。
{% endraw %}

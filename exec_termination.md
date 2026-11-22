{% raw %}
# 軽量な停止性検査 {#lightweight-termination-checking}

再帰的な `spec` 関数と `proof` 関数は必ず停止しなければならず、したがって必ず decreases 節を持たなければなりません。一方、exec 関数では停止しないことが許されています。それでも Verus は既定で、再帰的な `exec` 関数と `exec` モードのループに `decreases` 節があることを要求します。これが保証するのは、呼び出される側の関数もすべて停止するという仮定のもとで、その関数自身が停止することだけです。そのため、これは停止性の完全な保証ではなく、リント（lint）として扱うべきものです。

属性 [`#![verifier::exec_allows_no_decreases_clause]`](reference-attributes.html#verifierexec_allows_no_decreases_clause) を使うと、関数・モジュール・クレートの単位でこの検査を無効にできます。
{% endraw %}

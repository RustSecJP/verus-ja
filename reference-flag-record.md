{% raw %}
# record フラグ {#record-flag}

Verus の実行の記録を、その実行を再現するのに必要な依存物すべてと一緒に、共有するために残したいことがあります。
これは、検証済みのプロジェクトをひとまとめにするときや、Verus のバグを[イシュートラッカー](https://github.com/verus-lang/verus/issues)に報告するときに役立つでしょう。

`--record` フラグは、まさにこれを行います。具体的には、Verus の実行（たとえば `verus foo --bar --baz`）を記録するには、単に `--record` フラグを付け足します（たとえば `verus foo --bar --baz --record`）。すると、Verus がもう一度実行され、関係するすべてのソースファイルが、実行の出力とバージョン情報と一緒に、現在のディレクトリの zip ファイル（`yyyy-mm-dd-hh-mm-ss.zip`）にまとめられます。
{% endraw %}

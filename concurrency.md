{% raw %}
# 並行性 {#concurrency}

Verus は、自明でない所有権の規律を必要とするプログラムを検証するために、_VerusSync フレームワーク_ を提供しています。これには *マルチスレッドの並行コード* が含まれます。また、unsafe な機能（ポインタや unsafe なセルなど）を自明でない形で使う場合にも、しばしば必要になります。

この話題はかなり複雑なので、[別のチュートリアル兼リファレンスの本](https://verus-lang.github.io/verus/state_machines/intro.html)で扱っています。
{% endraw %}

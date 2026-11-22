{% raw %}
# 「global」ディレクティブ {#the-global-directive}

ほとんどの場合、Verus は、構造体の大きさ
（[`std::mem::size_of::<T>()`](https://doc.rust-lang.org/std/mem/fn.size_of.html)）
やアラインメント（[`std::mem::align_of::<T>()`](https://doc.rust-lang.org/std/mem/fn.align_of.html)）
といった[レイアウト情報](https://doc.rust-lang.org/reference/type-layout.html)を参照できません。
こうした情報は、不安定である（つまり、Rust のバージョンによって変わりうる）ことや、
プラットフォームに依存する（`usize` の大きさなど）ことがよくあります。
vstd は、[安定でプラットフォームに依存しないレイアウト情報についての公理](https://verus-lang.github.io/verus/verusdoc/vstd/layout/index.html)を
（たとえば、プリミティブ型やポインタ型について）いくつか提供していますが、ほとんどの場合は、レイアウトが不安定な型を扱う必要があります。

この情報は、`global` ディレクティブを使って、必要に応じて Verus に与えられます。

型 `T` と、整数リテラル `n` または `m` について、`global` ディレクティブは、次の形をとる Verus の項目です。

```verus-grammar
V@[global_item] ::= global layout R@[T:type] is size == R@[n:int_literal], align == R@[m:int_literal]
              | global layout R@[T:type] is size == R@[n:int_literal]
              | global layout R@[T:type] is align == R@[m:int_literal]
```

global ディレクティブは、次の両方を行います。

 * 公理 `size_of::<T>() == n` と `align_of::<T>() == m` の一方または両方を、Verus の証明で使えるように公開します
 * コンパイルしたときに、与えられた値が実際に正しいことを確かめる「静的な」検査を作ります。

2 つ目の検査は、コード生成を実行したとき*だけ*行われることに注意してください。「通常の」検証の実行では、
この検査は行われません。これにより、検査が必ず正しいプラットフォームの上で行われることが保証されますが、
コード生成を実行せずに検証に時間をかけていると、あとで驚くことになるかもしれません。

レイアウトを安定に保つために、
[`#[repr(C)]`](https://doc.rust-lang.org/reference/type-layout.html#reprc-structs) のような Rust の属性を使うことをおすすめします。
ただし、Verus の検証器は、これらの属性からは何の情報も得ないことを覚えておいてください。
レイアウト情報を Verus に与えられるのは、`global` ディレクティブを通じてだけです。

## `usize` と `isize` の場合 {#with-usize-and-isize}

整数型 `usize` と `isize` については、`global` ディレクティブに追加の動作があります。
具体的には、`usize` 型と `isize` 型を符号化するときに使われる*整数の範囲*に影響します。

整数リテラル `n` について、次のディレクティブは、

```
global layout usize is size == n;
```

Verus に次のことを伝えます。
  * `usize::BITS == 8 * n` 
  * `isize::BITS == 8 * n` 
  * `usize` の整数の範囲（`usize::MIN ..= usize::MAX`）は <code>0 ..= 2<sup>8*n</sup> - 1</code> である
  * `isize` の整数の範囲（`isize::MIN ..= isize::MAX`）は <code>-2<sup>8&#42;n-1</sup> ..= 2<sup>8*n-1</sup> - 1</code> である

既定では（つまり、`usize` や `isize` についての `global` ディレクティブが無い場合）、
Verus は、大きさが 4 または 8 のどちらかである、つまり、整数の範囲が
32 ビットまたは 64 ビットのどちらかであると仮定します。

### 例 {#example}

```
global layout usize is size == 4;

fn test(x: usize) {
    // This passes because Verus assumes x is a 32-bit integer:
    assert(x <= 0xffffffff);
    assert(usize::BITS == 32);
}
```
{% endraw %}

{% raw %}
# Rustdoc による文書化 {#documentation-with-rustdoc}

Verus は、Verus の仕様を rustdoc で見栄えよく表示するためのツールを提供しています。
これを使うには、まずクレートに対して `rustdoc` を実行し、その後で Verusdoc という HTML の後処理ツールを実行します。

まず、`verus/source` ディレクトリで `cargo build -p verusdoc` を実行して、`verusdoc` がビルドされていることを確かめます。

残念ながら、適切な依存とフラグを付けて `rustdoc` を実行するための便利な道具は、今のところありません。
そのため、手作業で設定する必要があります。
例を示します。

```bash
VERUS=/path/to/verus/source

if [ `uname` == "Darwin" ]; then
    DYN_LIB_EXT=dylib
elif [ `uname` == "Linux" ]; then
    DYN_LIB_EXT=so
fi

# Run rustdoc.
# Note the VERUSDOC environment variable.

RUSTC_BOOTSTRAP=1 VERUSDOC=1 rustdoc \
  --extern verus_builtin=$VERUS/target-verus/debug/libverus_builtin.rlib \
  --extern verus_builtin_macros=$VERUS/target-verus/debug/libverus_builtin_macros.$DYN_LIB_EXT \
  --extern verus_state_machines_macros=$VERUS/target-verus/debug/libverus_state_machines_macros.$DYN_LIB_EXT \
  --extern vstd=$VERUS/target-verus/debug/libvstd.rlib \
  --edition=2021 \
  --cfg verus_keep_ghost \
  --cfg verus_keep_ghost_body \
  --cfg 'feature="std"' \
  --cfg 'feature="alloc"' \
  '-Zcrate-attr=feature(register_tool)' \
  '-Zcrate-attr=register_tool(verus)' \
  '-Zcrate-attr=register_tool(verifier)' \
  '-Zcrate-attr=register_tool(verusfmt)' \
  --crate-type=lib \
  ./lib.rs

# Run the post-processor.

$VERUS/target/debug/verusdoc
```

これを、次のような `lib.rs` ファイルに対して実行したとします。

```
#![allow(unused_imports)]

use verus_builtin::*;
use verus_builtin_macros::*;
use vstd::prelude::*;

verus!{

/// Computes the max
pub fn compute_max(x: u32, y: u32) -> (max: u32)
    ensures max == (if x > y { x } else { y }),
{
    if x < y {
        y
    } else {
        x
    }
}

}
```

すると、次のような見た目の rustdoc が生成されます。

![`ensures` 節が含まれていることを示す、verusdoc の例のスクリーンショット](graphics/verusdoc-example.png)
{% endraw %}

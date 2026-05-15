# `'static` の強力な力

`#[init]`、`#[idle]`、および発散するソフトウェアタスクでは、`local` リソースは `'static` ライフタイムを持ちます。

これは、リソースを事前に割り当てたり、タスク、ドライバー、またはその他のオブジェクト間でリソースを分割したりする際に便利です。これは、USB ドライバーのようにメモリを割り当てる必要があるドライバーや、[`heapless::spsc::Queue`] のような分割可能なデータ構造を使用する場合に役立ちます。

次の例では、2 つの異なるタスクが [`heapless::spsc::Queue`] を共有し、共有キューにロックフリーでアクセスします。

[`heapless::spsc::Queue`]: https://docs.rs/heapless/0.7.5/heapless/spsc/struct.Queue.html

```rust,noplayground
{{#include ../../../../../examples/lm3s6965/examples/static-resources-in-init.rs}}
```

このプログラムを実行すると、期待どおりの出力が得られます。

```console
$ cargo xtask qemu --verbose --example static-resources-in-init
```

```console
{{#include ../../../../../ci/expected/lm3s6965/static-resources-in-init.run}}
```
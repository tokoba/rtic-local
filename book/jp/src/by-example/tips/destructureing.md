# リソースの分解

タスクが複数のリソースを受け取る場合、タスクのリソースを分割代入すると可読性が向上することがあります。以下に、リソース構造体を分割する 2 つの例を示します。

```rust,noplayground
{{#include ../../../../../examples/lm3s6965/examples/destructure.rs}}
```

```console
$ cargo xtask qemu --verbose --example destructure
```

```console
{{#include ../../../../../ci/expected/lm3s6965/destructure.run}}
```
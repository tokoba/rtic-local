# 最小のアプリケーション

これは、可能な限り最小の RTIC アプリケーションです。

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/smallest.rs}}
```

RTIC は、リソース効率を念頭に置いて設計されています。RTIC 自体はいかなる動的メモリ割り当てにも依存しないため、必要な RAM 量はアプリケーションのみに依存します。割り込みベクタテーブルを含めても、フラッシュメモリの使用量は 1kB 未満です。

最小の例では、次のようなものが想定されます。

```console
$ cargo xtask size --example smallest --backend thumbv7
```

```console
{{#include ../../../../ci/expected/lm3s6965/smallest.size}}
```

<!-- ---

厳密には、RTIC は各 *ソフトウェア* タスクごとに、静的に割り当てられたフューチャー（`Context` 構造体やスタックに割り当てられた変数を含む実行コンテキストを保持するもの）を生成します。同じ静的優先度に関連付けられたフューチャーは、実行中に非同期スタックを共有します。  -->
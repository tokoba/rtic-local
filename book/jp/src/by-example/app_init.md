# アプリケーションの初期化と `#[init]` タスク

RTIC アプリケーションでは、システムをセットアップする `init` タスクが必要です。対応する `init` 関数は
シグネチャ `fn(init::Context) -> (Shared, Local)` を持つ必要があります。ここで、`Shared` と `Local` はユーザー定義のリソース構造体です。

`init` タスクは、システムリセット後、[任意で定義された `pre-init` コードセクション][^pre-init] と、常に行われる RTIC の内部初期化の後に実行されます。

`init` タスクとオプションの `pre-init` タスクは、_割り込みを無効化した状態で_ 実行され、Cortex-M への排他的アクセス権を持ちます（`critical_section::CriticalSection` トークンは `cs` として利用できます）。

デバイス固有のペリフェラルは、`init::Context` の `core` フィールドと `device` フィールドを通じて利用できます。

[^pre-init]: [https://docs.rs/cortex-m-rt/latest/cortex_m_rt/attr.pre_init.html](https://docs.rs/cortex-m-rt/latest/cortex_m_rt/attr.pre_init.html)

## 例

以下の例は、`core`、`device`、`cs` フィールドの型を示すとともに、`'static` ライフタイムを持つ `local` 変数の使用例を示しています。このような変数は、RTIC アプリケーションの `init` タスクから他のタスクへ委譲できます。

`device` フィールドは、`peripherals` 引数がデフォルト値 `true` に設定されている場合にのみ利用できます。
ごくまれに超軽量なアプリケーションを実装したい場合は、`peripherals` を明示的に `false` に設定できます。

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/init.rs}}
```

この例を実行すると、コンソールに `init` が出力され、その後 QEMU プロセスが終了します。

```console
$ cargo xtask qemu --verbose --example init
```

```console
{{#include ../../../../ci/expected/lm3s6965/init.run}}
```
# v1.0.x から v2.0.0 への移行

RTIC `v1.0.x` から `v2.0.0` へプロジェクトを移行するには、次の手順が必要です。

1. `v2.1.0` は Rust Stable 1.75 以降で動作します（**推奨**）。一方、古いバージョンでは [`#![type_alias_impl_trait]`](https://github.com/rust-lang/rust/issues/63063) を使用するため、`nightly` コンパイラが必要です。
2. `v1.0.x` に含まれている monotonic を `rtic-time` および `rtic-monotonics` へ移行し、`spawn_after`、`spawn_at` を置き換えます。
3. ソフトウェアタスクは `async` であることが必須になったため、それらを正しく使用します。
4. `rtic-sync` が提供するデータ型を理解し、使用します。

変更点の詳細については、各小節を参照してください。

必要な変更のコード例を見たい場合は、[完全な移行例のページ](./migration_v1_v2/complete_example.md) を参照してください。

#### TL;DR（長いので要点だけ）

1. `spawn_after` と `spawn_at` の代わりに、`rtic-monotonics` が実装を提供する `async` 関数 `delay`、`delay_until`（および関連する関数）を使用するようになりました。
2. ソフトウェアタスクは теперь _必ず_ `async fn` でなければなりません。タスク内に `await` がある限り、タスクから戻らないことも許可されます。共有リソースは引き続き `lock` できます。
3. 新しいタスクを `spawn` する代わりに、共有リソースへのアクセスを `await` するには `rtic_sync::arbiter::Arbiter` を使用し、タスク間で通信するには `rtic_sync::channel::Channel` を使用します。
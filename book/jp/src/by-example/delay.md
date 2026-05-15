# Monotonic を使った遅延とタイムアウト

最小限のタイミング要件を表現する便利な方法の 1 つは、進行を遅延させることです。

これは monotonic タイマーをインスタンス化することで実現できます（実装については [`rtic-monotonics`] を参照してください）:

[`rtic-monotonics`]: https://github.com/rtic-rs/rtic/tree/master/rtic-monotonics
[`rtic-time`]: https://github.com/rtic-rs/rtic/tree/master/rtic-time
[`Monotonic`]: https://docs.rs/rtic-time/latest/rtic_time/trait.Monotonic.html
[Implementing a `Monotonic`]: ../monotonic_impl.md

```rust,noplayground
...
{{#include ../../../../examples/lm3s6965/examples/async-timeout.rs:init}}
        ...
```

_ソフトウェア_タスクは、遅延が満了するのを `await` できます:

```rust,noplayground
#[task]
async fn foo(_cx: foo::Context) {
    ...
    Mono::delay(100.millis()).await;
    ...
}

```

<details>
<summary>完全な例</summary>

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/async-delay.rs}}
```

```console
$ cargo xtask qemu --verbose --example async-delay --features test-critical-section
```

```console
{{#include ../../../../ci/expected/lm3s6965/async-delay.run}}
```

</details>

> [`Monotonic`] の新しい実装への貢献や、monotonic の内部動作に関する詳しい情報に興味がありますか？
> [Implementing a `Monotonic`] の章を確認してください！

## タイムアウト

Rust の [`Future`]（Rust の `async`/`await` を支える仕組み）は合成可能です。これにより、完了した `Future` の間で `select` することが可能になります。

[`Future`]: https://doc.rust-lang.org/std/future/trait.Future.html

一般的なユースケースは、タイムアウトを伴うトランザクションです。以下に示す例では、`hal_get(n).await` を呼び出すと何らかの想定上のトランザクションを実行する、ダミーの HAL デバイスを導入しています。所要時間は入力パラメーター（`n`）に基づいて `350ms + n * 100ms` としてモデル化しています。

`futures` クレートの `select_biased` マクロを使うと、次のようになります:

```rust,noplayground,noplayground
{{#include ../../../../examples/lm3s6965/examples/async-timeout.rs:select_biased}}
```

`hal_get` の完了に 450ms かかるとすると、200ms の短いタイムアウトは `hal_get` が完了する前に満了します。

タイムアウトを 1000ms に延ばすと、`hal_get` のほうが先に完了します。

`select_biased` を使えば任意の数の futures を組み合わせられるため、非常に強力です。しかし、タイムアウトのパターンは頻繁に使われるため、[`rtic-monotonics`] および [`rtic-time`] クレートによって提供される、より扱いやすいサポートが RTIC に組み込まれています。以下は別の例で、`Mono::delay_until` と `Mono::timeout_after` を使用します:

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/async-timeout.rs:timeout_at_basic}}
```

ドリフトなしで時間を正確に制御したい場合は、時刻の正確な一点を表す `Instant` と、時間の区間を表す `Duration` を使えます。`Instant` 型と `Duration` 型に対する操作は [`fugit`] クレートによって提供されます。

[`fugit`]: https://crates.io/crates/fugit

`let mut instant = Mono::now()` は、実行の開始時刻を設定します。

この開始時刻を基準に、1000ms ごとに `hal_get` を呼び出したいとします。これを実現するには、`instant` を 1000 ms ずつ増やし、その後で `Mono::delay_until(instant).await` を使います。このループを繰り返す中で発生する追加の遅延は、'now + 1000' ではなく 'previous + 1000' まで待機することで補償されます（'now + 1000' だとループのタイミングがドリフトします）。

上記の `select!` を使った async タイムアウトの例に対する別の方法として、将来の時点を `timeout` として定義し、`Mono::timeout_at(timeout, hal_get(n)).await` を呼び出します。

ループの 1 回目の反復では `n == 0` なので、`hal_get` は 350ms かかり（前述のとおり）、タイムアウト前に完了します。2 回目の反復では遅延は 450ms で、これも依然としてタイムアウト前に完了します。3 回目の反復では `n == 2` となり、`hal_get` の完了には 550ms かかるため、この場合はタイムアウトに達します。

<details>
<summary>完全な例</summary>

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/async-timeout.rs}}
```

```console
$ cargo xtask qemu --verbose --example async-timeout --features test-critical-section
```

```console
{{#include ../../../../ci/expected/lm3s6965/async-timeout.run}}
```

</details>
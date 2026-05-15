# タスクの優先度

## 優先度

`priority` 引数は、各 `task` の静的優先度を宣言します。

Cortex-M では、タスクは `0..=(1 << NVIC_PRIO_BITS)` の範囲の優先度を持つことができます。ここで `NVIC_PRIO_BITS` は `device` クレートで定義される定数です。

`priority` 引数を省略すると、タスクの優先度はデフォルトで `0` になります。`idle` タスクは変更不可能な静的優先度 `0` を持ち、これは最も低い優先度です。

> RTIC では、数値が大きいほど優先度が高く、これは NVIC ペリフェラルにおける
> Cortex-M の動作とは逆です。
> 明示的に言うと、数値 `10` は数値 `9` より **高い** 優先度を持ちます。

複数のタスクが実行可能な状態にある場合、最も高い静的優先度を持つタスクが優先されます。

次のシナリオは、タスクの優先順位付けを示しています:
より低い優先度のタスク B の実行中に、より高い優先度のタスク A を spawn すると、タスク B は一時停止します。タスク A はより高い優先度を持つため、タスク B をプリエンプトし、タスク B はタスク A の実行が完了するまで一時停止したままになります。したがって、タスク A が完了すると、タスク B は実行を再開します。

```text
Task Priority
  ┌────────────────────────────────────────────────────────┐
  │                                                        │
  │                                                        │
3 │                      Preempts                          │
2 │                    A─────────►                         │
1 │          B─────────► - - - - B────────►                │
0 │Idle┌─────►                   Resumes  ┌──────────►     │
  ├────┴──────────────────────────────────┴────────────────┤
  │                                                        │
  └────────────────────────────────────────────────────────┘Time
```

次の例は、優先度に基づくタスクスケジューリングを示しています。

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/preempt.rs}}
```

```console
$ cargo xtask qemu --verbose --example preempt
{{#include ../../../../ci/expected/lm3s6965/preempt.run}}
```

タスク `bar` の優先度は `baz` と _同じ_ であるため、タスク `bar` はタスク `baz` を _プリエンプトしない_ ことに注意してください。`baz` が戻ると、より高い優先度のタスク `bar` が `foo` より先に実行されます。`bar` が戻ると、`foo` は再開できます。

優先度についてもう 1 点注意があります。デバイスがサポートする値より高い優先度を選ぶと、コンパイルエラーになります。Rust 言語の制約により、このエラーは分かりにくいものになります。`example/common.rs` のタスク `uart0_interrupt` に対して `priority = 9` とすると、次のようになります。

Rust 言語の制約により、このエラーは分かりにくいものになります。`example/common.rs` のタスク `uart0_interrupt` に対して `priority = 9` とすると、次のようになります。

```text
   error[E0080]: evaluation of constant value failed
  --> examples/common.rs:10:1
   |
10 | #[rtic::app(device = lm3s6965, dispatchers = [SSI0, QEI0])]
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ attempt to compute `8_usize - 9_usize`, which would overflow
   |
   = note: this error originates in the attribute macro `rtic::app` (in Nightly builds, run with -Z macro-backtrace for more info)

```

エラーメッセージは誤ってマクロの開始位置を指していますが、少なくとも減算された値（この場合は 9）から、どのタスクがエラーの原因かを推測できます。
# バックグラウンドタスク `#[idle]`

`idle` 属性でマークされた関数は、モジュール内に任意で記述できます。これは特別な _idle タスク_ となり、シグネチャは `fn(idle::Context) -> !` でなければなりません。

存在する場合、ランタイムは `init` の後に `idle` タスクを実行します。`init` とは異なり、`idle` は _割り込みが有効な状態で_ 実行され、`-> !` の関数シグネチャが示すとおり、決してリターンしてはいけません。
[Rust の型 `!` は「never」を意味します][nevertype]。

[nevertype]: https://doc.rust-lang.org/core/primitive.never.html

`init` と同様に、ローカルに宣言されたリソースは安全にアクセスできる `'static` ライフタイムを持ちます。

以下の例は、`idle` が `init` の後に実行されることを示しています。

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/idle.rs}}
```

```console
$ cargo xtask qemu --verbose --example idle
```

```console
{{#include ../../../../ci/expected/lm3s6965/idle.run}}
```

デフォルトでは、RTIC の `idle` タスクは特定のターゲット向けの最適化を行いません。

一般的で有用な最適化の 1 つは、[SLEEPONEXIT] を有効にして、`idle` に到達したときに MCU がスリープに入れるようにすることです。

> **注意**: 一部のハードウェアでは、設定しない限り、スリープモード中にデバッグユニットが無効になります。
>
> これは RTIC の範囲外であるため、ハードウェア固有のドキュメントを参照してください。

次の例は、
[`SLEEPONEXIT`][SLEEPONEXIT] を設定し、デフォルトの [`nop()`][NOP] を [`wfi()`][WFI] に置き換えるカスタム `idle` タスクを提供することで、スリープを有効にする方法を示しています。

[SLEEPONEXIT]: https://developer.arm.com/documentation/100737/0100/Power-management/Sleep-mode/Sleep-on-exit-bit
[WFI]: https://developer.arm.com/documentation/dui0662/b/The-Cortex-M0--Instruction-Set/Miscellaneous-instructions/WFI
[NOP]: https://developer.arm.com/documentation/dui0662/b/The-Cortex-M0--Instruction-Set/Miscellaneous-instructions/NOP

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/idle-wfi.rs}}
```

```console
$ cargo xtask qemu --verbose --example idle-wfi
```

```console
{{#include ../../../../ci/expected/lm3s6965/idle-wfi.run}}
```

> **注意**: `idle` タスクは、優先度 0 で動作する _ソフトウェア_ タスクと一緒には使用できません。理由は、`idle` が優先度 0 でリターンしない Rust 関数として実行されるためです。そのため、優先度 0 のエグゼキュータが同じ優先度の _ソフトウェア_ タスクに制御を渡す方法がありません。
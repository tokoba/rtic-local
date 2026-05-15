# ハードウェアタスク

RTIC はその中核で、タスクのスケジュールと実行開始にハードウェア割り込みコントローラー（[cortex-m 上の ARM NVIC][NVIC]）を使用します。`pre-init`（隠れた「タスク」）、`#[init]`、`#[idle]` を除くすべてのタスクは、割り込みハンドラとして実行されます。

タスクを割り込みにバインドするには、`#[task]` 属性引数 `binds = InterruptName` を使用します。すると、このタスクはこのハードウェア割り込みベクターの割り込みハンドラになります。

明示的な割り込みにバインドされたタスクはすべて、ハードウェアイベントに反応して実行を開始するため、_ハードウェアタスク_ と呼ばれます。

存在しない割り込み名を指定すると、コンパイルエラーになります。割り込み名は通常、[PAC または HAL][pacorhal] クレートで定義されています。

利用可能な割り込みベクターであれば、どれでも動作するはずです。特定のデバイスでは、ユーザーコードからは制御できない形で、特定の割り込み優先度が特定の割り込みベクターに結び付けられている場合があります。例として [nRF “softdevice”](https://github.com/rtic-rs/rtic/issues/434) を参照してください。

ハードウェア機能によって内部的に使用されている割り込みベクターを使用する際は注意してください。RTIC は、そのようなハードウェア固有の詳細を認識しません。

[pacorhal]: https://docs.rust-embedded.org/book/start/registers.html
[NVIC]: https://developer.arm.com/documentation/100166/0001/Nested-Vectored-Interrupt-Controller/NVIC-functional-description/NVIC-interrupts

## 例

以下の例は、割り込みハンドラにバインドされたハードウェアタスクを宣言するための `#[task(binds = InterruptName)]` 属性の使い方を示しています。

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/hardware.rs}}
```

```console
$ cargo xtask qemu --verbose --example hardware
```

```console
{{#include ../../../../ci/expected/lm3s6965/hardware.run}}
```
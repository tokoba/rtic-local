# チャネルを介した通信。

チャネルは、実行中のタスク間でデータをやり取りするために使用できます。チャネルは本質的には待機キューであり、複数のプロデューサーと単一のレシーバーを持つタスク間の通信を可能にします。チャネルは `init` タスクで構築され、静的に割り当てられたメモリを基盤としています。送信エンドポイントと受信エンドポイントは _ソフトウェア_ タスクに配布されます。

```rust,noplayground
...
const CAPACITY: usize = 5;
#[init]
    fn init(_: init::Context) -> (Shared, Local) {
        let (s, r) = make_channel!(u32, CAPACITY);
        receiver::spawn(r).unwrap();
        sender1::spawn(s.clone()).unwrap();
        sender2::spawn(s.clone()).unwrap();
        ...
```

この場合、チャネルは `u32` 型のデータを保持し、容量は 5 要素です。

チャネルは _ハードウェア_ タスクからも使用できますが、[Try API](#try-api) を使用した非 `async` の方法でのみ可能です。

## データの送信

`send` メソッドは、以下のようにチャネルにメッセージを送信します。

```rust,noplayground
#[task]
async fn sender1(_c: sender1::Context, mut sender: Sender<'static, u32, CAPACITY>) {
    hprintln!("Sender 1 sending: 1");
    sender.send(1).await.unwrap();
}
```

## データの受信

受信側は到着するメッセージを `await` できます。

```rust,noplayground
#[task]
async fn receiver(_c: receiver::Context, mut receiver: Receiver<'static, u32, CAPACITY>) {
    while let Ok(val) = receiver.recv().await {
        hprintln!("Receiver got: {}", val);
        ...
    }
}
```

チャネルは、競合状態を防ぐために、小さな（グローバルな）_クリティカルセクション_（CS）を使って実装されています。ユーザーは CS 実装を提供しなければなりません。例を `--features test-critical-section` 付きでコンパイルすると、可能な実装の 1 つが得られます。

完全な例:

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/async-channel.rs}}
```

```console
$ cargo xtask qemu --verbose --example async-channel --features test-critical-section
```

```console
{{#include ../../../../ci/expected/lm3s6965/async-channel.run}}
```

また、送信エンドポイントも `await` できます。チャネル容量がまだ上限に達していない場合、送信側を `await` した処理は直ちに進行できます。一方、容量に達している場合、キューに空きができるまで送信側はブロックされます。このようにしてデータが失われることはありません。

以下の例では、`CAPACITY` が 1 に減らされており、送信タスクはチャネル内のデータが受信されるまで待機することになります。

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/async-channel-done.rs}}
```

出力を見ると、`Sender 2` は `Sender 1` によって送信されたデータが受信されるまで待機することがわかります。

> **注意** 同じ優先度の _ソフトウェア_ タスクは互いに非同期に実行されるため、厳密な順序を **一切** 仮定できません。（ここで示している順序は現在の実装にのみ当てはまるものであり、RTIC フレームワークのリリース間で変わる可能性があります。）

```console
$ cargo xtask qemu --verbose --example async-channel-done --features test-critical-section
{{#include ../../../../ci/expected/lm3s6965/async-channel-done.run}}
```

## エラーハンドリング

すべての送信側がドロップされている場合、空の受信チャネルを `await` するとエラーになります。これにより、さまざまな種類のシャットダウン処理を適切に実装できます。

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/async-channel-no-sender.rs}}
```

```console
$ cargo xtask qemu --verbose --example async-channel-no-sender --features test-critical-section
```

```console
{{#include ../../../../ci/expected/lm3s6965/async-channel-no-sender.run}}
```

同様に、レシーバーがドロップされている場合、送信チャネルを `await` するとエラーになります。これにより、アプリケーションレベルのエラーハンドリングを適切に実装できます。

その際のエラーはデータを送信側に返すため、送信側は適切な対応（たとえば、後で再送するためにデータを保存するなど）を取ることができます。

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/async-channel-no-receiver.rs}}
```

```console
$ cargo xtask qemu --verbose --example async-channel-no-receiver --features test-critical-section
```

```console
{{#include ../../../../ci/expected/lm3s6965/async-channel-no-receiver.run}}
```

## Try API

Try API を使うと、操作の成功を前提とせずに、また非 `async` コンテキストでも、チャネルへの送信およびチャネルからの受信を行えます。

この API は `Receiver::try_recv` と `Sender::try_send` を通じて提供されています。

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/async-channel-try.rs}}
```

```console
$ cargo xtask qemu --verbose --example async-channel-try --features test-critical-section
```

```console
{{#include ../../../../ci/expected/lm3s6965/async-channel-try.run}}
```
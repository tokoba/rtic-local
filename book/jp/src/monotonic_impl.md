# Monotonic の仕組み

内部では、すべての Monotonic 実装が [Timer Queue](#the-timer-queue) を使用します。これは、対応する `Future` が完了すべき時刻を記述したエントリを持つ優先度キューです。

## スケジューリング用 `Monotonic` タイマーの実装

[`rtic-time`] フレームワークは、コンペアマッチを備え、さらに必要に応じてスケジューリング用のオーバーフロー割り込みもサポートする任意のタイマーを利用できるため、柔軟です。タイマーを RTIC で利用可能にするための唯一の要件は、[`rtic-time::Monotonic`] トレイトを実装することです。

RTIC 2.0 では、[`Monotonic`] を実装する際のすべての時間ベースの操作の基盤として、ユーザーが [`fugit`] などの時間ライブラリを利用していることを前提としています。これらのライブラリを使うことで、[`Monotonic`] トレイトを正しく実装することが大幅に容易になり、その結果、システム内のほぼ任意のタイマーをスケジューリングに使用できるようになります。

このトレイトには、各メソッドに対する要件が記載されています。参考にできるリファレンス実装が [`rtic-monotonics`] に用意されています。

- [`Systick based`] は固定の割り込み（tick）レートで動作します。多少のオーバーヘッドはありますが、シンプルで、長い時間範囲をサポートします
- [`RP2040 Timer`] は、割り込みなしで長時間待機できる「本格的な」実装です。スケジューリングを処理するために [`TimerQueue`] をどのように使うかを明確に示しています。
- [`nRF52 timers`] は、nRF52 の RTC と通常タイマー向けに Monotonic とタイマーキューを実装しています

## 貢献

`Monotonic` の新しい実装への貢献は、複数の方法で行えます。
* [`rtic-monotonics`] に feature flag の下でこのトレイトを実装し、それらをメインの RTIC リポジトリに含めるための PR を作成してください。この方法なら、実装は in-tree となるため、RTIC はその正しさを保証でき、新しいリリース時に更新できます。
* 外部リポジトリで変更を実装してください。この方法では [`rtic-monotonics`] に含まれませんが、将来的にそうしやすくなる可能性があります。

[`rtic-monotonics`]: https://github.com/rtic-rs/rtic/tree/master/rtic-monotonics/
[`fugit`]: https://docs.rs/fugit/
[`Systick based`]: https://github.com/rtic-rs/rtic/blob/master/rtic-monotonics/src/systick.rs
[`rtic-monotonics`]:  https://github.com/rtic-rs/rtic/blob/master/rtic-monotonics
[`RP2040 Timer`]: https://github.com/rtic-rs/rtic/blob/master/rtic-monotonics/src/rp2040.rs
[`nRF52 timers`]: https://github.com/rtic-rs/rtic/blob/master/rtic-monotonics/src/nrf.rs
[`rtic-time`]: https://docs.rs/rtic-time/latest/rtic_time
[`rtic-time::Monotonic`]: https://docs.rs/rtic-time/latest/rtic_time/trait.Monotonic.html
[`Monotonic`]: https://docs.rs/rtic-time/latest/rtic_time/trait.Monotonic.html
[`TimerQueue`]: https://docs.rs/rtic-time/latest/rtic_time/timer_queue/struct.TimerQueue.html

## The timer queue

タイマーキューはリストベースの優先度キューとして実装されており、リストノードは、monotonic を待機する際に作成される Future を `await` したときに生成される  `Future` の一部として静的に割り当てられます。したがって、タイマーキューは実行時に失敗しません（そのサイズと割り当てはコンパイル時に決定されます）。

チャネル実装と同様に、タイマーキュー実装は競合を防ぐためにグローバルな *クリティカルセクション* (CS) に依存しています。例では、ビルドオプションに `--features test-critical-section` を追加することで、CS 実装が提供されます。
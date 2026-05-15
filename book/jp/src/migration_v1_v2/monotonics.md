# `rtic-monotonics` への移行

以前の `rtic` のバージョンでは、モノトニックは `#[rtic::app]` の不可欠で密結合な一部でした。この新しいバージョンでは、[`rtic-monotonics`] によって、より疎結合な方法で提供されます。

`#[monotonic]` 属性はもう使用しません。代わりに、[`rtic-monotonics`] の `create_X_token` を使用します。このマクロを呼び出すと割り込み登録トークンが返され、これを使って目的のモノトニックのインスタンスを構築できます。

`spawn_after` と `spawn_at` はもう利用できません。代わりに、[`rtic-monotonics`] を通じて利用できる `rtic_time::Monotonic` トレイトの実装が提供する async 関数 `delay` と `delay_until` を使用します。

必要な変更の概要については、[コード例](./complete_example.md) を参照してください。

現在のモノトニック実装の詳細については、[`rtic-monotonics` のドキュメント](https://docs.rs/rtic-monotonics) と、[サンプル](https://github.com/rtic-rs/rtic/tree/master/examples) を参照してください。

[`rtic-monotonics`]: https://github.com/rtic-rs/rtic
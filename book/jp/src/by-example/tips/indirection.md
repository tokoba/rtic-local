# メッセージ受け渡しを高速化するための間接参照の利用

メッセージ受け渡しでは、常に送信側から静的変数へ、さらに静的変数から受信側へと、ペイロードのコピーが発生します。そのため、`[u8; 128]` のような大きなバッファをメッセージとして送信すると、高コストな
`memcpy` が 2 回発生します。

間接参照を使うと、メッセージ受け渡しのオーバーヘッドを最小化できます。つまり、バッファを値として送る代わりに、バッファへの所有権を持つポインタを送ることができます。

間接参照を実現するには、グローバルメモリアロケータ（`alloc::Box`、`alloc::Rc` など）を使う方法がありますが、Rust v1.37.0 時点では nightly チャネルの使用が必要です。あるいは、[`heapless::Pool`] のような静的に確保されたメモリプールを使うこともできます。

[`heapless::Pool`]: https://docs.rs/heapless/latest/heapless/pool/index.html

このアプローチの例は、shared と local を持つ RTIC のリソースモデルを完全に外れるため、プログラムは、この場合 `heapless::pool` であるメモリアロケータの正しさに依存することになります。

以下は、`heapless::Pool` を使って 128 バイトのバッファを「box 化」する例です。

```rust,noplayground
{{#include ../../../../../examples/lm3s6965/examples/pool.rs}}
```

```console
$ cargo xtask qemu --verbose --example pool
```

```console
{{#include ../../../../../ci/expected/lm3s6965/pool.run}}
```
# RAM からタスクを実行する

RTIC v0.4.0 で RTIC アプリケーションの指定を属性へ移した主な目的は、ほかの属性と相互運用できるようにすることでした。たとえば、`link_section` 属性をタスクに適用してそれらを RAM に配置できます。これにより、場合によっては性能が向上することがあります。

> **重要**: 一般に、`link_section`、`export_name`、`no_mangle` 属性は強力ですが、誤用もしやすいものです。これらの属性のいずれかを誤って使用すると未定義動作を引き起こす可能性があります。そのため、常に `cortex-m-rt` の `interrupt` 属性や `exception` 属性のような、安全でより高水準な属性を優先して使用すべきです。
>
> RAM 上の関数という個別のケースでは、`cortex-m-rt` v0.6.5 にはそのための安全な抽象化はありませんが、将来のリリースで `ramfunc` 属性を追加するための [RFC] があります。

[RFC]: https://github.com/rust-embedded/cortex-m-rt/pull/100

以下の例は、より高い優先度を持つタスク `bar` を RAM に配置する方法を示しています。

```rust,noplayground
{{#include ../../../../../examples/lm3s6965/examples/ramfunc.rs}}
```

このプログラムを実行すると、期待どおりの出力が得られます。

```console
$ cargo xtask qemu --verbose --example ramfunc
```

```console
{{#include ../../../../../ci/expected/lm3s6965/ramfunc.run}}
```

`cargo-nm` の出力を見ると、`bar` が RAM
(`0x2000_0000`) に配置され、一方で `foo` は Flash (`0x0000_0000`) に配置されたことを確認できます。

```console
$ cargo nm --example ramfunc --release | grep ' foo::'
```

```console
{{#include ../../../../../ci/expected/lm3s6965/ramfunc.run.grep.foo}}
```

```console
$ cargo nm --example ramfunc  --target thumbv7m-none-eabi --release | grep '*bar::'
```

```console
{{#include ../../../../../ci/expected/lm3s6965/ramfunc.run.grep.bar}}
```
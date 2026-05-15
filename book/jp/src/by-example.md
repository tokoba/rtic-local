# RTIC を例で学ぶ

この本のこのパートでは、複雑さが段階的に増していく例を通して、
RTIC フレームワークを新しいユーザーに紹介します。

この本のこのパートにあるすべての例は、
`examples` ディレクトリにある [RTICリポジトリ][repoexamples] の一部です。
これらの例は QEMU 上で実行可能であり（Cortex M3 ターゲットをエミュレート）、
そのため、読み進めるのに特別なハードウェアは必要ありません。

[repoexamples]: https://github.com/rtic-rs/rtic/tree/master/rtic/examples

## 例を実行する

QEMU で例を実行するには、`qemu-system-arm` プログラムが必要です。
QEMU を含む組み込み開発環境のセットアップ方法については、
[the embedded Rust book] の手順を参照してください。

[the embedded Rust book]: https://rust-embedded.github.io/book/intro/install.html

QEMU を使って `examples/` にある例をローカルで実行するには:

```
cargo xtask qemu
```

これにより、デフォルトの `thumbv7m-none-eabi` デバイス `lm3s6965` に対して、すべての例が実行されます。

実行する例を絞り込むには、`--example <example name>` フラグを使用します。ここで名前は例のファイル名です。

依存関係が整っているとすると、次を実行すると:

```console
$ cargo xtask qemu --example locals
```

次の出力が得られます:

```console
   Finished dev [unoptimized + debuginfo] target(s) in 0.07s
    Running `target/debug/xtask qemu --example locals`
INFO  xtask > Testing for platform: Lm3s6965, backend: Thumbv7
INFO  xtask::run > 👟 Build example locals (thumbv7m-none-eabi, release, "test-critical-section,thumbv7-backend", in examples/lm3s6965)
INFO  xtask::run > ✅ Success.
INFO  xtask::run > 👟 Run example locals in QEMU (thumbv7m-none-eabi, release, "test-critical-section,thumbv7-backend", in examples/lm3s6965)
INFO  xtask::run > ✅ Success.
INFO  xtask::results > ✅ Success: Build example locals (thumbv7m-none-eabi, release, "test-critical-section,thumbv7-backend", in examples/lm3s6965)
INFO  xtask::results > ✅ Success: Run example locals in QEMU (thumbv7m-none-eabi, release, "test-critical-section,thumbv7-backend", in examples/lm3s6965)
INFO  xtask::results > 🚀🚀🚀 All tasks succeeded 🚀🚀🚀
```

例が正常に通っており、これが RTIC の CI セットアップの一部でもあるのは素晴らしいことですが、この本の目的上、実際のプログラム出力を見るには `--verbose` フラグ、短くは `-v` を追加する必要があります:

```console
❯ cargo xtask qemu --verbose --example locals
    Finished dev [unoptimized + debuginfo] target(s) in 0.03s
     Running `target/debug/xtask qemu --example locals --verbose`
 DEBUG xtask > Stderr of child processes is inherited: false
 DEBUG xtask > Partial features: false
 INFO  xtask > Testing for platform: Lm3s6965, backend: Thumbv7
 INFO  xtask::run > 👟 Build example locals (thumbv7m-none-eabi, release, "test-critical-section,thumbv7-backend", in examples/lm3s6965)
 INFO  xtask::run > ✅ Success.
 INFO  xtask::run > 👟 Run example locals in QEMU (thumbv7m-none-eabi, release, "test-critical-section,thumbv7-backend", in examples/lm3s6965)
 INFO  xtask::run > ✅ Success.
 INFO  xtask::results > ✅ Success: Build example locals (thumbv7m-none-eabi, release, "test-critical-section,thumbv7-backend", in examples/lm3s6965)
    cd examples/lm3s6965 && cargo build --target thumbv7m-none-eabi --features test-critical-section,thumbv7-backend --release --example locals
 DEBUG xtask::results >
cd examples/lm3s6965 && cargo build --target thumbv7m-none-eabi --features test-critical-section,thumbv7-backend --release --example locals
Stderr:
    Finished release [optimized] target(s) in 0.02s
 INFO  xtask::results > ✅ Success: Run example locals in QEMU (thumbv7m-none-eabi, release, "test-critical-section,thumbv7-backend", in examples/lm3s6965)
    cd examples/lm3s6965 && cargo run --target thumbv7m-none-eabi --features test-critical-section,thumbv7-backend --release --example locals
 DEBUG xtask::results >
cd examples/lm3s6965 && cargo run --target thumbv7m-none-eabi --features test-critical-section,thumbv7-backend --release --example locals
Stdout:
bar: local_to_bar = 1
foo: local_to_foo = 1
idle: local_to_idle = 1

Stderr:
    Finished release [optimized] target(s) in 0.02s
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel target/thumbv7m-none-eabi/release/examples/locals`
Timer with period zero, disabling

 INFO  xtask::results > 🚀🚀🚀 All tasks succeeded 🚀🚀🚀
```

出力の末尾付近にある `Stdout:` の後の内容を確認してください。プログラム出力には次の行が含まれているはずです:

```console
{{#include ../../../ci/expected/lm3s6965/locals.run}}
```

> **注記**: 
> `cargo xtask` のその他の便利なオプションについては、次を参照してください:
> ```
> cargo xtask qemu --help
> ```
> 
> `--platform` フラグを使うと、どのデバイス上で例を実行するかを変更できます。
> 現在は `lm3s6965` が最もよくサポートされており、ARM と RISC-V の両方を含む
> 他のデバイスのサポート拡充も進められています
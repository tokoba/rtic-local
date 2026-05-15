# 生成されたコードの確認

`#[rtic::app]` は補助コードを生成するプロシージャルマクロです。
何らかの理由でこのマクロによって生成されたコードを確認する必要がある場合、方法は 2 つあります。

* `target` ディレクトリ内の `rtic-expansion.rs` ファイルを確認する
* [`cargo-expand`] サブコマンドを使う

## 生成された `rtic-expansion.rs` を使う

このファイルの場所は、ビルドの実行方法によって異なります。

たとえばメインの RTIC リポジトリ内で `cargo xtask build-example` を使うと、このファイルは使用した "platform" に応じて配置されます。

```
$ cargo xtask example-build --example smallest
$ cargo xtask example-build --example monotonic --platform esp32-c3

$ fd -u rtic-expansion.rs
examples/esp32c3/target/rtic-expansion.rs
examples/lm3s6965/target/rtic-expansion.rs
```

通常の cargo プロジェクトの場合は、`target` フォルダー直下に置かれます。

このファイルには、*最後にビルドされた*（`cargo build` または `cargo check` による）RTIC アプリケーションの `#[rtic::app]` アイテムの展開結果（プログラム全体ではありません！）が含まれます。
展開されたコードはデフォルトでは整形されていないため、読む前に `rustfmt` を実行したくなるでしょう。

``` console
$ cargo build --example smallest --target thumbv7m-none-eabi
```

``` console
$ rustfmt target/rtic-expansion.rs
```

``` console
$ tail target/rtic-expansion.rs
```

``` rust,noplayground
#[doc = r" Implementation details"]
mod app {
    #[doc = r" Always include the device crate which contains the vector table"]
    use lm3s6965 as _;
    #[no_mangle]
    unsafe extern "C" fn main() -> ! {
        rtic::export::interrupt::disable();
        let mut core: rtic::export::Peripherals = core::mem::transmute(());
        core.SCB.scr.modify(|r| r | 1 << 1);
        rtic::export::interrupt::enable();
        loop {
            rtic::export::wfi()
        }
    }
}
```

## `cargo-expand` ツールを使う

利用できない場合は、インストールしてください。

```
$ cargo install cargo-expand
```

このサブコマンドは、`#[rtic::app]` 属性やクレート内のモジュールを含む *すべて* のマクロを展開し、その出力をコンソールに表示します。


[`cargo-expand`]: https://crates.io/crates/cargo-expand

``` console
# 前と同じ出力を生成します
```

``` console
cargo expand --example smallest | tail
```
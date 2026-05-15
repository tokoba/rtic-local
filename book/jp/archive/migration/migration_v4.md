# v0.4.x から v0.5.0 への移行

このセクションでは、RTFM v0.4.x 向けに書かれたアプリケーションを
フレームワークの v0.5.0 にアップグレードする方法について説明します。

## プロジェクト名の変更 RTFM -> RTIC

[v0.5.2][rtic0.5.2] のリリースで、名前は Real-Time Interrupt-driven Concurrency に変更されました

`RTFM` の出現箇所はすべて `RTIC` に変更する必要があります。

[RTFM から RTIC への移行ガイド](./migration_rtic.md) を参照してください

[rtic0.5.2]: https://crates.io/crates/cortex-m-rtic/0.5.2

## `Cargo.toml`

`cortex-m-rtfm` のバージョンを
`"0.5.0"` に変更し、`rtfm` を `rtic` に変更してください。
`timer-queue` フィーチャーを削除してください。

``` toml
[dependencies.cortex-m-rtfm]
# これを変更
version = "0.4.3"

# これに変更
[dependencies.cortex-m-rtic]
version = "0.5.0"

# この Cargo フィーチャーを削除
features = ["timer-queue"]
#           ^^^^^^^^^^^^^
```

## `Context` 引数

`#[rtfm::app]` アイテム内のすべての関数は、最初の引数として
`Context` 構造体を受け取る必要があります。この `Context` 型には、
フレームワークの v0.4.x が関数のスコープに暗黙的に注入していた
変数 `resources`、`spawn`、`schedule` が含まれます。これらの変数は
`Context` 構造体のフィールドになります。`#[rtfm::app]` アイテム内の各関数は
それぞれ異なる `Context` 型を受け取ります。

``` rust,noplayground
#[rtfm::app(/* .. */)]
const APP: () = {
    // これを変更
    #[task(resources = [x], spawn = [a], schedule = [b])]
    fn foo() {
        resources.x.lock(|x| /* .. */);
        spawn.a(message);
        schedule.b(baseline);
    }

    // これに変更
    #[task(resources = [x], spawn = [a], schedule = [b])]
    fn foo(mut cx: foo::Context) {
        // ^^^^^^^^^^^^^^^^^^^^

        cx.resources.x.lock(|x| /* .. */);
    //  ^^^

        cx.spawn.a(message);
    //  ^^^

        cx.schedule.b(message, baseline);
    //  ^^^
    }

    // これを変更
    #[init]
    fn init() {
        // ..
    }

    // これに変更
    #[init]
    fn init(cx: init::Context) {
        //  ^^^^^^^^^^^^^^^^^
        // ..
    }

    // ..
};
```

## リソース

リソースを宣言するための構文は、`static mut`
変数から `struct Resources` に変更されました。

``` rust,noplayground
#[rtfm::app(/* .. */)]
const APP: () = {
    // これを変更
    static mut X: u32 = 0;
    static mut Y: u32 = (); // 遅延リソース

    // これに変更
    struct Resources {
        #[init(0)] // <- 初期値
        X: u32, // 注: 命名スタイルを `snake_case` に変更することを推奨します

        Y: u32, // 遅延リソース
    }

    // ..
};
```

## デバイスペリフェラル

アプリケーションが `#[init]` 内で `device` 変数を通じてデバイスペリフェラルに
アクセスしていた場合、`init::Context` 構造体の `device` フィールドを通じて
引き続きデバイスペリフェラルにアクセスするには、`#[rtfm::app]` 属性に
`peripherals = true` を追加する必要があります。

これを変更します:

``` rust,noplayground
#[rtfm::app(/* .. */)]
const APP: () = {
    #[init]
    fn init() {
        device.SOME_PERIPHERAL.write(something);
    }

    // ..
};
```

これに変更します:

``` rust,noplayground
#[rtfm::app(/* .. */, peripherals = true)]
//                    ^^^^^^^^^^^^^^^^^^
const APP: () = {
    #[init]
    fn init(cx: init::Context) {
        //  ^^^^^^^^^^^^^^^^^
        cx.device.SOME_PERIPHERAL.write(something);
    //  ^^^
    }

    // ..
};
```

## `#[interrupt]` と `#[exception]`

`#[interrupt]` 属性と `#[exception]` 属性を削除してください。
v0.5.x でハードウェアタスクを宣言するには、代わりに `binds` 引数を指定した `#[task]`
属性を使用してください。

これを変更します:

``` rust,noplayground
#[rtfm::app(/* .. */)]
const APP: () = {
    // ハードウェアタスク
    #[exception]
    fn SVCall() { /* .. */ }

    #[interrupt]
    fn UART0() { /* .. */ }

    // ソフトウェアタスク
    #[task]
    fn foo() { /* .. */ }

    // ..
};
```

これに変更します:

``` rust,noplayground
#[rtfm::app(/* .. */)]
const APP: () = {
    #[task(binds = SVCall)]
    //     ^^^^^^^^^^^^^^
    fn svcall(cx: svcall::Context) { /* .. */ }
    // ^^^^^^ ここでは `snake_case` 名を使用することを推奨します

    #[task(binds = UART0)]
    //     ^^^^^^^^^^^^^
    fn uart0(cx: uart0::Context) { /* .. */ }

    #[task]
    fn foo(cx: foo::Context) { /* .. */ }

    // ..
};
```

## `schedule`

`schedule` API では、`timer-queue` Cargo フィーチャーは不要になりました。
`schedule` API を使用するには、まず `#[rtfm::app]` 属性の `monotonic` 引数を使って、
ランタイムが使用するモノトニックタイマーを定義する必要があります。
モノトニックタイマーとしてサイクルカウンター (CYCCNT) を引き続き使用し、
v0.4.x と同じ動作にするには、`#[rtfm::app]` 属性に `monotonic = rtfm::cyccnt::CYCCNT`
引数を追加してください。

また、`Duration` 型と `Instant` 型、および `U32Ext` トレイトは
`rtfm::cyccnt` モジュールに移動しました。
このモジュールは ARMv7-M+ デバイスでのみ利用できます。
`timer-queue` が削除されたことで、`DWT` ペリフェラルは
コアペリフェラル構造体内に再び含まれるようになりました。`DWT` が必要な場合は、
アプリケーションが `init` 内でこれを有効化するようにしてください。

これを変更します:

``` rust,noplayground
use rtfm::{Duration, Instant, U32Ext};

#[rtfm::app(/* .. */)]
const APP: () = {
    #[task(schedule = [b])]
    fn a() {
        // ..
    }
};
```

これに変更します:

``` rust,noplayground
use rtfm::cyccnt::{Duration, Instant, U32Ext};
//        ^^^^^^^^

#[rtfm::app(/* .. */, monotonic = rtfm::cyccnt::CYCCNT)]
//                    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
const APP: () = {
    #[init]
    fn init(cx: init::Context) {
        cx.core.DWT.enable_cycle_counter();
        // オプション: デバッガーが接続されていなくても DWT が動作するよう設定します
        cx.core.DCB.enable_trace();
    }
    #[task(schedule = [b])]
    fn a(cx: a::Context) {
        // ..
    }
};
```
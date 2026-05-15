# 割り込みの設定

割り込みは、RTIC アプリケーションの動作の中核です。割り込み優先度を正しく
設定し、それらが実行時にも固定されたままであることを保証することは、
アプリケーションのメモリ安全性にとって必須です。

RTIC フレームワークは、割り込み優先度をコンパイル時に宣言されるものとして
公開しています。しかし、この静的な設定は、アプリケーションの初期化時に
関連するレジスタへ書き込まなければなりません。割り込みの設定は、`init`
関数が実行される前に行われます。

この例は、RTIC フレームワークが実行するコードのイメージを示しています:

``` rust,noplayground
#[rtic::app(device = lm3s6965)]
mod app {
    #[init]
    fn init(c: init::Context) {
        // .. ユーザーコード ..
    }

    #[idle]
    fn idle(c: idle::Context) -> ! {
        // .. ユーザーコード ..
    }

    #[interrupt(binds = UART0, priority = 2)]
    fn foo(c: foo::Context) {
        // .. ユーザーコード ..
    }
}
```

フレームワークは、次のようなエントリポイントを生成します:

``` rust,noplayground
// プログラムの実際のエントリポイント
#[no_mangle]
unsafe fn main() -> ! {
    // 論理優先度をハードウェア / NVIC の優先度に変換する
    fn logical2hw(priority: u8) -> u8 {
        use lm3s6965::NVIC_PRIO_BITS;

        // NVIC は優先度をビットの上位ビットにエンコードする
        // また、数値が大きいほど優先度は低い
        ((1 << NVIC_PRIORITY_BITS) - priority) << (8 - NVIC_PRIO_BITS)
    }

    cortex_m::interrupt::disable();

    let mut core = cortex_m::Peripheral::steal();

    core.NVIC.enable(Interrupt::UART0);

    // ユーザーが指定した値
    let uart0_prio = 2;

    // 指定した優先度がサポートされる範囲内にあることをコンパイル時にチェックする
    let _ = [(); (1 << NVIC_PRIORITY_BITS) - (uart0_prio as usize)];

    core.NVIC.set_priority(Interrupt::UART0, logical2hw(uart0_prio));

    // ユーザーコードを呼び出す
    init(/* .. */);

    // ..

    cortex_m::interrupt::enable();

    // ユーザーコードを呼び出す
    idle(/* .. */)
}
```
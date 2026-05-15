# 移行の完全な例

以下に、v1.0.x および v2.0.0 における `stm32f3_blinky` サンプルの実装コードを示します。さらに下に、diff を示します。

# v1.0.X

```rust
#![deny(unsafe_code)]
#![deny(warnings)]
#![no_main]
#![no_std]

use panic_rtt_target as _;
use rtic::app;
use rtt_target::{rprintln, rtt_init_print};
use stm32f3xx_hal::gpio::{Output, PushPull, PA5};
use stm32f3xx_hal::prelude::*;
use systick_monotonic::{fugit::Duration, Systick};

#[app(device = stm32f3xx_hal::pac, peripherals = true, dispatchers = [SPI1])]
mod app {
    use super::*;

    #[shared]
    struct Shared {}

    #[local]
    struct Local {
        led: PA5<Output<PushPull>>,
        state: bool,
    }

    #[monotonic(binds = SysTick, default = true)]
    type MonoTimer = Systick<1000>;

    #[init]
    fn init(cx: init::Context) -> (Shared, Local, init::Monotonics) {
        // クロックを設定
        let mut flash = cx.device.FLASH.constrain();
        let mut rcc = cx.device.RCC.constrain();

        let mono = Systick::new(cx.core.SYST, 36_000_000);

        rtt_init_print!();
        rprintln!("init");

        let _clocks = rcc
            .cfgr
            .use_hse(8.MHz())
            .sysclk(36.MHz())
            .pclk1(36.MHz())
            .freeze(&mut flash.acr);

        // LED を設定
        let mut gpioa = cx.device.GPIOA.split(&mut rcc.ahb);
        let mut led = gpioa
            .pa5
            .into_push_pull_output(&mut gpioa.moder, &mut gpioa.otyper);
        led.set_high().unwrap();

        // 点滅タスクをスケジュール
        blink::spawn_after(Duration::<u64, 1, 1000>::from_ticks(1000)).unwrap();

        (
            Shared {},
            Local { led, state: false },
            init::Monotonics(mono),
        )
    }

    #[task(local = [led, state])]
    fn blink(cx: blink::Context) {
        rprintln!("blink");
        if *cx.local.state {
            cx.local.led.set_high().unwrap();
            *cx.local.state = false;
        } else {
            cx.local.led.set_low().unwrap();
            *cx.local.state = true;
        }
        blink::spawn_after(Duration::<u64, 1, 1000>::from_ticks(1000)).unwrap();
    }
}

```

# V2.0.0

``` rust,noplayground
{{ #include ../../../../examples/stm32f3_blinky/src/main.rs }}
```

## 2 つのプロジェクト間の diff

_注_: この diff は 100% 正確ではない可能性がありますが、重要な変更点を示しています。

``` diff
#![no_main]
 #![no_std]
 
 use panic_rtt_target as _;
 use rtic::app;
 use stm32f3xx_hal::gpio::{Output, PushPull, PA5};
 use stm32f3xx_hal::prelude::*;
-use systick_monotonic::{fugit::Duration, Systick};
+use rtic_monotonics::Systick;
 
 #[app(device = stm32f3xx_hal::pac, peripherals = true, dispatchers = [SPI1])]
 mod app {
@@ -20,16 +21,14 @@ mod app {
         state: bool,
     }
 
-    #[monotonic(binds = SysTick, default = true)]
-    type MonoTimer = Systick<1000>;
-
     #[init]
     fn init(cx: init::Context) -> (Shared, Local, init::Monotonics) {
-        // Setup clocks
+        // クロックを設定
         let mut flash = cx.device.FLASH.constrain();
         let mut rcc = cx.device.RCC.constrain();
 
-        let mono = Systick::new(cx.core.SYST, 36_000_000);
+        let mono_token = rtic_monotonics::create_systick_token!();
+        let mono = Systick::start(cx.core.SYST, 36_000_000, mono_token);
 
         let _clocks = rcc
             .cfgr
@@ -46,7 +45,7 @@ mod app {
         led.set_high().unwrap();
 
-        // Schedule the blinking task
+        // 点滅タスクをスケジュール
-        blink::spawn_after(Duration::<u64, 1, 1000>::from_ticks(1000)).unwrap();
+        blink::spawn().unwrap();
 
         (
             Shared {},
@@ -56,14 +55,18 @@ mod app {
     }
 
     #[task(local = [led, state])]
-    fn blink(cx: blink::Context) {
-        rprintln!("blink");
-        if *cx.local.state {
-            cx.local.led.set_high().unwrap();
-            *cx.local.state = false;
-        } else {
-            cx.local.led.set_low().unwrap();
-            *cx.local.state = true;
-        blink::spawn_after(Duration::<u64, 1, 1000>::from_ticks(1000)).unwrap();
-    }
+    async fn blink(cx: blink::Context) {
+        loop {
+            // タスクは、ループ内のどこかに
+            // `await` がある限り、無限に実行できるようになりました。
+            SysTick::delay(1000.millis()).await;
+            rprintln!("blink");
+            if *cx.local.state {
+                cx.local.led.set_high().unwrap();
+                *cx.local.state = false;
+            } else {
+                cx.local.led.set_low().unwrap();
+                *cx.local.state = true;
+            }
+        }
+    }
 }
```
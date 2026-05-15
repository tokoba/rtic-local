# 遅延リソース

一部のリソースは、`init` 関数が返った後に実行時に初期化されます。
これらのリソース（静的変数）は、タスクの実行が許可される前に完全に初期化
されていることが重要です。つまり、割り込みが無効化されている間に初期化
されなければなりません。

以下の例は、遅延リソースを初期化するためにフレームワークが生成するコードの
種類を示しています。

``` rust,noplayground
#[rtic::app(device = ..)]
mod app {
    struct Resources {
        x: Thing,
    }

    #[init]
    fn init() -> init::LateResources {
        // ..

        init::LateResources {
            x: Thing::new(..),
        }
    }

    #[task(binds = UART0, resources = [x])]
    fn foo(c: foo::Context) {
        let x: &mut Thing = c.resources.x;

        x.frob();

        // ..
    }

    // ..
}
```

フレームワークによって生成されるコードは次のようになります。

``` rust,noplayground
fn init(c: init::Context) -> init::LateResources {
    // .. ユーザーコード ..
}

fn foo(c: foo::Context) {
    // .. ユーザーコード ..
}

// 公開API
pub mod init {
    pub struct LateResources {
        pub x: Thing,
    }

    // ..
}

pub mod foo {
    pub struct Resources<'a> {
        pub x: &'a mut Thing,
    }

    pub struct Context<'a> {
        pub resources: Resources<'a>,
        // ..
    }
}

/// 実装の詳細
mod app {
    // 未初期化のstatic変数
    static mut x: MaybeUninit<Thing> = MaybeUninit::uninit();

    #[no_mangle]
    unsafe fn main() -> ! {
        cortex_m::interrupt::disable();

        // ..

        let late = init(..);

        // 遅延リソースの初期化
        x.as_mut_ptr().write(late.x);

        cortex_m::interrupt::enable(); //~ コンパイラフェンス

        // この時点で、例外、割り込み、タスクは `main` をプリエンプトできる

        idle(..)
    }

    #[no_mangle]
    unsafe fn UART0() {
        foo(foo::Context {
            resources: foo::Resources {
                // この時点で `x` は初期化されている
                x: &mut *x.as_mut_ptr(),
            },
            // ..
        })
    }
}
```

ここで重要な詳細は、`interrupt::enable` が *compiler
fence* のように振る舞うことです。これにより、コンパイラは `X` への書き込みを
`interrupt::enable` の *後* に並べ替えられなくなります。もしコンパイラがそのような
並べ替えを行うと、その書き込みと `foo` が `X` に対して実行する何らかの操作との間で
データ競合が発生します。

より複雑な命令パイプラインを持つアーキテクチャでは、割り込みを再度有効化する前に
書き込み操作を完全にフラッシュするために、compiler fence ではなくメモリバリア
（`atomic::fence`）が必要になる場合があります。ARM Cortex-M アーキテクチャでは、
シングルコアのコンテキストではメモリバリアは不要です。
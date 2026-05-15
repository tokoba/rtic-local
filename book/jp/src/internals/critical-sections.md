# クリティカルセクション

あるリソース（静的変数）が、異なる優先度で動作する 2 つ以上のタスクの
間で共有される場合、メモリをデータ競合のない形で変更するには、何らかの
相互排他が必要です。RTIC では、相互排他を保証するために優先度ベースの
クリティカルセクションを使用します（[即時天井優先度プロトコル][icpp] を
参照）。

[icpp]: https://en.wikipedia.org/wiki/Priority_ceiling_protocol

クリティカルセクションは、タスクの *動的* 優先度を一時的に引き上げる
ことで構成されます。タスクがこのクリティカルセクション内にいる間は、
そのリソースを要求しうるほかのすべてのタスクは *開始してはなりません*。

特定のリソースに対する相互排他を確保するには、動的優先度をどこまで
高くしなければならないでしょうか。その問いに答えるのは
[上限解析](ceilings.html) であり、次の節で説明します。この節では
クリティカルセクションの実装に焦点を当てます。

## リソースプロキシ

単純化のため、異なる優先度で動作する 2 つのタスクで共有される
リソースを見てみましょう。明らかに、一方のタスクはもう一方を
プリエンプトできます。データ競合を防ぐため、*低優先度* のタスクは
共有メモリを変更する必要があるときにクリティカルセクションを使用
しなければなりません。一方で、高優先度タスクは低優先度タスクに
プリエンプトされないため、共有メモリを直接変更できます。低優先度
タスクにクリティカルセクションの使用を強制するため、そのタスクには
*リソースプロキシ* を渡し、高優先度タスクには一意参照
(`&mut-`) を渡します。

次の例は、各タスクに渡される異なる型を示しています:

``` rust,noplayground
#[rtic::app(device = ..)]
mut app {
    struct Resources {
        #[init(0)]
        x: u64,
    }

    #[interrupt(binds = UART0, priority = 1, resources = [x])]
    fn foo(c: foo::Context) {
        // リソースプロキシ
        let mut x: resources::x = c.resources.x;

        x.lock(|x: &mut u64| {
            // クリティカルセクション
            *x += 1
        });
    }

    #[interrupt(binds = UART1, priority = 2, resources = [x])]
    fn bar(c: bar::Context) {
        let mut x: &mut u64 = c.resources.x;

        *x += 1;
    }

    // ..
}
```

では、これらの型がフレームワークによってどのように生成されるのかを
見てみましょう。

``` rust,noplayground
fn foo(c: foo::Context) {
    // .. ユーザーコード ..
}

fn bar(c: bar::Context) {
    // .. ユーザーコード ..
}

pub mod resources {
    pub struct x {
        // ..
    }
}

pub mod foo {
    pub struct Resources {
        pub x: resources::x,
    }

    pub struct Context {
        pub resources: Resources,
        // ..
    }
}

pub mod bar {
    pub struct Resources<'a> {
        pub x: &'a mut u64,
    }

    pub struct Context {
        pub resources: Resources,
        // ..
    }
}

mod app {
    static mut x: u64 = 0;

    impl rtic::Mutex for resources::x {
        type T = u64;

        fn lock<R>(&mut self, f: impl FnOnce(&mut u64) -> R) -> R {
            // これについては後で詳しく見ていきます
        }
    }

    #[no_mangle]
    unsafe fn UART0() {
        foo(foo::Context {
            resources: foo::Resources {
                x: resources::x::new(/* .. */),
            },
            // ..
        })
    }

    #[no_mangle]
    unsafe fn UART1() {
        bar(bar::Context {
            resources: bar::Resources {
                x: &mut x,
            },
            // ..
        })
    }
}
```

## `lock`

では次に、クリティカルセクションそのものを詳しく見ていきましょう。
この例では、データ競合を防ぐために動的優先度を少なくとも `2` まで
引き上げる必要があります。Cortex-M アーキテクチャでは、動的優先度は
`BASEPRI` レジスタに書き込むことで変更できます。

`BASEPRI` レジスタのセマンティクスは次のとおりです:

- `0` を `BASEPRI` に書き込むと、その機能は無効になります。
- 0 以外の値を `BASEPRI` に書き込むと、割り込みプリエンプションに
  必要な優先度レベルが変更されます。ただし、これは書き込まれた値が
  現在の実行コンテキストの優先度レベルよりも *低い* 場合にのみ
  効果があります。なお、ハードウェア上での優先度レベルが低いほど、
  論理的な優先度は高くなります

したがって、任意の時点における動的優先度は次のように計算できます

``` rust,noplayground
dynamic_priority = max(hw2logical(BASEPRI), hw2logical(static_priority))
```

ここで、`static_priority` は現在の割り込みに対して NVIC に
プログラムされている優先度であり、現在のコンテキストが `idle` の
場合は論理的な `0` です。

この具体例では、クリティカルセクションを次のように実装できます:

> **注:** これは簡略化した実装です

``` rust,noplayground
impl rtic::Mutex for resources::x {
    type T = u64;

    fn lock<R, F>(&mut self, f: F) -> R
    where
        F: FnOnce(&mut u64) -> R,
    {
        unsafe {
            // クリティカルセクションの開始: 動的優先度を `2` まで引き上げる
            asm!("msr BASEPRI, 192" : : : "memory" : "volatile");

            // クリティカルセクション内でユーザーコードを実行する
            let r = f(&mut x);

            // クリティカルセクションの終了: 動的優先度を静的な値 (`1`) に戻す
            asm!("msr BASEPRI, 0" : : : "memory" : "volatile");

            r
        }
    }
}
```

ここでは、`asm!` ブロックで `"memory"` clobber を使用することが
重要です。これにより、コンパイラがこれをまたいでメモリ操作を
並べ替えるのを防げます。これは、クリティカルセクションの外側で
変数 `x` にアクセスするとデータ競合が発生するため重要です。

`lock` メソッドのシグネチャにより、その呼び出しをネストできないことに
注意することが重要です。これはメモリ安全性のために必要です。ネストした
呼び出しは `x` への複数の一意参照 (`&mut-`) を生成し、Rust の
エイリアシング規則を破ってしまうためです。以下を見てください:

``` rust,noplayground
#[interrupt(binds = UART0, priority = 1, resources = [x])]
fn foo(c: foo::Context) {
    // リソースプロキシ
    let mut res: resources::x = c.resources.x;

    res.lock(|x: &mut u64| {
        res.lock(|alias: &mut u64| {
            //~^ エラー: `res` はすでに一意に借用されています (`&mut-`)
            // ..
        });
    });
}
```

## ネスト

*同じ* リソースに対する `lock` 呼び出しのネストは、メモリ安全性のために
コンパイラによって拒否されなければなりませんが、*異なる* リソースに
対する `lock` 呼び出しのネストは有効な操作です。その場合、クリティカル
セクションのネストによって動的優先度が下がることが決してないようにしたい
です。そうなると不健全だからです。また、`BASEPRI` レジスタへの書き込み回数
とコンパイラフェンスの数も最適化したいと考えます。そのために、スタック
変数を使ってタスクの動的優先度を追跡し、それに基づいて `BASEPRI` に
書き込むかどうかを決定します。実際には、このスタック変数はコンパイラに
よって最適化で取り除かれますが、それでもコンパイラに追加情報を提供します。

次のプログラムを考えてみましょう:
``` rust,noplayground
#[rtic::app(device = ..)]
mod app {
    struct Resources {
        #[init(0)]
        x: u64,
        #[init(0)]
        y: u64,
    }

    #[init]
    fn init() {
        rtic::pend(Interrupt::UART0);
    }

    #[interrupt(binds = UART0, priority = 1, resources = [x, y])]
    fn foo(c: foo::Context) {
        let mut x = c.resources.x;
        let mut y = c.resources.y;

        y.lock(|y| {
            *y += 1;

            *x.lock(|x| {
                x += 1;
            });

            *y += 1;
        });

        // 中間点

        x.lock(|x| {
            *x += 1;

            y.lock(|y| {
                *y += 1;
            });

            *x += 1;
        })
    }

    #[interrupt(binds = UART1, priority = 2, resources = [x])]
    fn bar(c: foo::Context) {
        // ..
    }

    #[interrupt(binds = UART2, priority = 3, resources = [y])]
    fn baz(c: foo::Context) {
        // ..
    }

    // ..
}
```

フレームワークによって生成されるコードは次のようになります:

``` rust,noplayground
// 省略: ユーザーコード

pub mod resources {
    pub struct x<'a> {
        priority: &'a Cell<u8>,
    }

    impl<'a> x<'a> {
        pub unsafe fn new(priority: &'a Cell<u8>) -> Self {
            x { priority }
        }

        pub unsafe fn priority(&self) -> &Cell<u8> {
            self.priority
        }
    }

    // `y` についても同様
}

pub mod foo {
    pub struct Context {
        pub resources: Resources,
        // ..
    }

    pub struct Resources<'a> {
        pub x: resources::x<'a>,
        pub y: resources::y<'a>,
    }
}

mod app {
    use cortex_m::register::basepri;

    #[no_mangle]
    unsafe fn UART1() {
        // ユーザーが指定したこの割り込みの静的優先度
        const PRIORITY: u8 = 2;

        // BASEPRI のスナップショットを取得
        let initial = basepri::read();

        let priority = Cell::new(PRIORITY);
        bar(bar::Context {
            resources: bar::Resources::new(&priority),
            // ..
        });

        // 先ほど取得したスナップショット値に BASEPRI を戻す
        basepri::write(initial); // 前に見た `asm!` ブロックと同じ
    }

    // `UART0` / `foo` と `UART2` / `baz` についても同様

    impl<'a> rtic::Mutex for resources::x<'a> {
        type T = u64;

        fn lock<R>(&mut self, f: impl FnOnce(&mut u64) -> R) -> R {
            unsafe {
                // このリソースの優先度上限
                const CEILING: u8 = 2;

                let current = self.priority().get();
                if current < CEILING {
                    // 動的優先度を引き上げる
                    self.priority().set(CEILING);
                    basepri::write(logical2hw(CEILING));

                    let r = f(&mut y);

                    // 動的優先度を復元する
                    basepri::write(logical2hw(current));
                    self.priority().set(current);

                    r
                } else {
                    // 動的優先度は十分に高い
                    f(&mut y)
                }
            }
        }
    }

    // リソース `y` についても同様
}
```

最終的にコンパイラは関数 `foo` を次のような形に
最適化します:

``` rust,noplayground
fn foo(c: foo::Context) {
    // 注: この時点で BASEPRI には値 `0`（リセット値）が入っている

    // 動的優先度を `3` に引き上げる
    unsafe { basepri::write(160) }

    // `y` に対する 2 つの操作は 1 つにまとめられる
    y += 2;

    // 動的優先度が十分に高いため、`x` にアクセスするために BASEPRI は変更されない
    x += 1;

    // 動的優先度を `1` に下げる（復元する）
    unsafe { basepri::write(224) }

    // 中間点

    // 動的優先度を `2` に引き上げる
    unsafe { basepri::write(192) }

    x += 1;

    // 動的優先度を `3` に引き上げる
    unsafe { basepri::write(160) }

    y += 1;

    // 動的優先度を `2` に下げる（復元する）
    unsafe { basepri::write(192) }

    // 注: `x` に対するこの操作を前のものとまとめても健全だが、
    // コンパイラフェンスは粒度が粗く、そのような最適化を妨げる
    x += 1;

    // 動的優先度を `1` に下げる（復元する）
    unsafe { basepri::write(224) }

    // 注: この時点で BASEPRI には値 `224` が入っている
    // UART0 ハンドラは戻る前にこの値を `0` に復元する
}
```

## BASEPRI の不変条件

RTIC フレームワークが保持しなければならない不変条件の 1 つは、
*割り込み* ハンドラの開始時点における BASEPRI の値が、
割り込みハンドラの終了時点の値と同じでなければならないということです。BASEPRI は
割り込みハンドラの実行中に変化してもかまいませんが、割り込みハンドラを最初から最後まで
実行しても、BASEPRI に観測可能な変化が生じてはなりません。

この不変条件は、プリエンプションによってハンドラの動的優先度が
引き上げられるのを避けるために保持される必要があります。これは次の例を見るとよくわかります:

``` rust,noplayground
#[rtic::app(device = ..)]
mod app {
    struct Resources {
        #[init(0)]
        x: u64,
    }

    #[init]
    fn init() {
        // `init` が戻った直後に `foo` が実行される
        rtic::pend(Interrupt::UART0);
    }

    #[task(binds = UART0, priority = 1)]
    fn foo() {
        // この時点で BASEPRI は `0`、現在の動的優先度は `1`

        // この時点で `bar` が `foo` をプリエンプトする
        rtic::pend(Interrupt::UART1);

        // この時点で BASEPRI は `192`（バグのため）、動的優先度は `2` になっている
        // この関数は `idle` に戻る
    }

    #[task(binds = UART1, priority = 2, resources = [x])]
    fn bar() {
        // BASEPRI は `0`（動的優先度 = 2）

        x.lock(|x| {
            // BASEPRI は `160` に引き上げられる（動的優先度 = 3）

            // ..
        });

        // BASEPRI は `192` に復元される（動的優先度 = 2）
    }

    #[idle]
    fn idle() -> ! {
        // BASEPRI は `192`（バグのため）、動的優先度 = 2

        // BASEPRI の値のため、これは効果がない
        // タスク `foo` は二度と実行されない
        rtic::pend(Interrupt::UART0);

        loop {
            // ..
        }
    }

    #[task(binds = UART2, priority = 3, resources = [x])]
    fn baz() {
        // ..
    }

}
```

重要: `UART1` で `BASEPRI` をロールバックするのを *忘れた* としましょう -- これは
RTIC コードジェネレータのバグになります。

``` rust,noplayground
// RTIC によって生成されたコード

mod app {
    // ..

    #[no_mangle]
    unsafe fn UART1() {
        // ユーザーが指定したこの割り込みの静的優先度
        const PRIORITY: u8 = 2;

        // BASEPRI のスナップショットを取得
        let initial = basepri::read();

        let priority = Cell::new(PRIORITY);
        bar(bar::Context {
            resources: bar::Resources::new(&priority),
            // ..
        });

        // バグ: 先ほど取得したスナップショット値に BASEPRI を戻すのを忘れた
        basepri::write(initial);
    }
}
```
その結果、`idle` は動的優先度 `2` で実行されることになり、実際には
システムが再び `2` 未満の動的優先度で実行されることはなくなります。これは
プログラムのメモリ安全性を損なうものではありませんが、タスクスケジューリングに影響します:
この特定のケースでは、優先度 `1` のタスクは決して実行の機会を得られなく
なります。
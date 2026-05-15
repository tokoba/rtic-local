# ソフトウェアタスク

RTIC はソフトウェアタスクとハードウェアタスクをサポートしています。各ハードウェアタスクは
異なる割り込みハンドラに結び付けられます。一方、複数のソフトウェアタスクが
同じ割り込みハンドラによってディスパッチされる場合があります。これは、
フレームワークが使用する割り込みハンドラの数を最小限に抑えるためです。

フレームワークは `spawn` 可能なタスクを優先度レベルごとにグループ化し、
優先度レベルごとに 1 つの*タスクディスパッチャ*を生成します。各タスクディスパッチャは
異なる割り込みハンドラ上で動作し、その割り込みハンドラの優先度は、
ディスパッチャが管理するタスクの優先度レベルに一致するよう設定されます。

各タスクディスパッチャは、実行可能な*ready*状態のタスクの*キュー*を保持します。この
キューは*レディキュー*と呼ばれます。ソフトウェアタスクを `spawn` することは、
このキューにエントリを追加し、対応するタスクディスパッチャを実行する
割り込みをペンディングすることから成ります。このキューの各エントリには、
実行するタスクを識別するタグ（`enum`）と、そのタスクに渡されるメッセージへの
*ポインタ*が含まれます。

レディキューは、SPSC（Single Producer Single Consumer）のロックフリーキューです。タスク
ディスパッチャはこのキューのコンシューマーエンドポイントを所有します。一方、
プロデューサーエンドポイントは、他のタスクを `spawn` できるタスクによって競合する
リソースとして扱われます。

## タスクディスパッチャ

まず、タスクをディスパッチするためにフレームワークが生成するコードを見てみましょう。
次の例を考えてみます。

``` rust,noplayground
#[rtic::app(device = ..)]
mod app {
    // ..

    #[interrupt(binds = UART0, priority = 2, spawn = [bar, baz])]
    fn foo(c: foo::Context) {
        foo.spawn.bar().ok();

        foo.spawn.baz(42).ok();
    }

    #[task(capacity = 2, priority = 1)]
    fn bar(c: bar::Context) {
        // ..
    }

    #[task(capacity = 2, priority = 1, resources = [X])]
    fn baz(c: baz::Context, input: i32) {
        // ..
    }

    extern "C" {
        fn UART1();
    }
}
```

フレームワークは、割り込みハンドラとレディキューから構成される次の
タスクディスパッチャを生成します。

``` rust,noplayground
fn bar(c: bar::Context) {
    // .. ユーザーコード ..
}

mod app {
    use heapless::spsc::Queue;
    use cortex_m::register::basepri;

    struct Ready<T> {
        task: T,
        // ..
    }

    /// 優先度レベル `1` で実行される `spawn` 可能なタスク
    enum T1 {
        bar,
        baz,
    }

    // タスクディスパッチャのレディキュー
    // `5-1=4` はこのキューの容量を表す
    static mut RQ1: Queue<Ready<T1>, 5> = Queue::new();

    // 優先度 `1` のタスクをディスパッチするために選ばれた割り込みハンドラ
    #[no_mangle]
    unsafe UART1() {
        // この割り込みハンドラの優先度
        const PRIORITY: u8 = 1;

        let snapshot = basepri::read();

        while let Some(ready) = RQ1.split().1.dequeue() {
            match ready.task {
                T1::bar => {
                    // **NOTE** 簡略化した実装

                    // 動的優先度を追跡するために使用
                    let priority = Cell::new(PRIORITY);

                    // ユーザーコードを呼び出す
                    bar(bar::Context::new(&priority));
                }

                T1::baz => {
                    // `baz` は後で見ます
                }
            }
        }

        // BASEPRI 不変条件
        basepri::write(snapshot);
    }
}
```

## タスクを `spawn` する

`spawn` API は、`Spawn` 構造体のメソッドとしてユーザーに公開されます。
`Spawn` 構造体はタスクごとに 1 つあります。

前の例に対してフレームワークが生成する `Spawn` コードは、次のようになります。

``` rust,noplayground
mod foo {
    // ..

    pub struct Context<'a> {
        pub spawn: Spawn<'a>,
        // ..
    }

    pub struct Spawn<'a> {
        // タスクの動的優先度を追跡する
        priority: &'a Cell<u8>,
    }

    impl<'a> Spawn<'a> {
        // ユーザーにこれを改変してほしくないため、`unsafe` にして隠している
        #[doc(hidden)]
        pub unsafe fn priority(&self) -> &Cell<u8> {
            self.priority
        }
    }
}

mod app {
    // ..

    // `RQ1` のプロデューサーエンドポイントの優先度上限
    const RQ1_CEILING: u8 = 2;

    // さらにいくつの `bar` メッセージをエンキューできるかを追跡するために使用
    // `3-1=2` はこのキューの容量を表す
    // このキューは `init` が実行される前にフレームワークによって埋められる
    static mut bar_FQ: Queue<(), 3> = Queue::new();

    // `bar_FQ` のコンシューマーエンドポイントの優先度上限
    const bar_FQ_CEILING: u8 = 2;

    // 優先度ベースのクリティカルセクション
    //
    // これは、与えられたクロージャ `f` を少なくとも
    // `ceiling` の動的優先度で実行する
    fn lock(priority: &Cell<u8>, ceiling: u8, f: impl FnOnce()) {
        // ..
    }

    impl<'a> foo::Spawn<'a> {
        /// `bar` タスクを `spawn` する
        pub fn bar(&self) -> Result<(), ()> {
            unsafe {
                match lock(self.priority(), bar_FQ_CEILING, || {
                    bar_FQ.split().1.dequeue()
                }) {
                    Some(()) => {
                        lock(self.priority(), RQ1_CEILING, || {
                            // タスクをレディキューに入れる
                            RQ1.split().1.enqueue_unchecked(Ready {
                                task: T1::bar,
                                // ..
                            })
                        });

                        // タスクディスパッチャを実行する割り込みをペンディングする
                        rtic::pend(Interrupt::UART0);
                    }

                    None => {
                        // 最大容量に達したため、spawn に失敗した
                        Err(())
                    }
                }
            }
        }
    }
}
```

`spawn` できる `bar` タスクの数を制限するために `bar_FQ` を使うのは、
人為的な制限に見えるかもしれませんが、タスクの容量について説明するときに
より意味が分かるようになります。

## メッセージ

メッセージパッシングが実際にどのように動作するかは省いていたので、ここで `spawn`
実装をもう一度見てみましょう。ただし今回は、`u64` メッセージを受け取る
タスク `baz` を対象にします。
``` rust,noplayground
fn baz(c: baz::Context, input: u64) {
    // .. ユーザーコード ..
}

mod app {
    // ..

    // ここで `Ready` 構造体の全内容を示します
    struct Ready {
        task: Task,
        // メッセージのインデックス。`INPUTS` バッファの添字として使用される
        index: u8,
    }

    // `baz` に渡されるメッセージを保持するために予約されたメモリ
    static mut baz_INPUTS: [MaybeUninit<u64>; 2] =
        [MaybeUninit::uninit(), MaybeUninit::uninit()];

    // フリーキュー: `baz_INPUTS` 配列内の空きスロットを追跡するために使われる
    // このキューは `init` が実行される前に値 `0` と `1` で初期化される
    static mut baz_FQ: Queue<u8, 3> = Queue::new();

    // `baz_FQ` のコンシューマエンドポイントの優先度上限
    const baz_FQ_CEILING: u8 = 2;

    impl<'a> foo::Spawn<'a> {
        /// `baz` タスクを spawn する
        pub fn baz(&self, message: u64) -> Result<(), u64> {
            unsafe {
                match lock(self.priority(), baz_FQ_CEILING, || {
                    baz_FQ.split().1.dequeue()
                }) {
                    Some(index) => {
                        // 注: `index` はこのバッファへの所有ポインタである
                        baz_INPUTS[index as usize].write(message);

                        lock(self.priority(), RQ1_CEILING, || {
                            // タスクをレディキューに入れる
                            RQ1.split().1.enqueue_unchecked(Ready {
                                task: T1::baz,
                                index,
                            });
                        });

                        // タスクディスパッチャを実行する割り込みをペンドする
                        rtic::pend(Interrupt::UART0);
                    }

                    None => {
                        // 最大容量に達した。spawn は失敗する
                        Err(message)
                    }
                }
            }
        }
    }
}
```

では次に、タスクディスパッチャの実際の実装を見てみましょう:

``` rust,noplayground
mod app {
    // ..

    #[no_mangle]
    unsafe UART1() {
        const PRIORITY: u8 = 1;

        let snapshot = basepri::read();

        while let Some(ready) = RQ1.split().1.dequeue() {
            match ready.task {
                Task::baz => {
                    // 注: `index` はこのバッファへの所有ポインタである
                    let input = baz_INPUTS[ready.index as usize].read();

                    // メッセージは読み出されたので、このスロットを
                    // フリーキューに戻せる
                    // （タスクディスパッチャはこのキューのプロデューサ
                    // エンドポイントに排他的にアクセスできる）
                    baz_FQ.split().0.enqueue_unchecked(ready.index);

                    let priority = Cell::new(PRIORITY);
                    baz(baz::Context::new(&priority), input)
                }

                Task::bar => {
                    // `baz` 分岐とまったく同じ
                }

            }
        }

        // BASEPRI の不変条件
        basepri::write(snapshot);
    }
}
```

`INPUTS` とフリーキューである `FQ` を合わせると、実質的にはメモリプールです。しかし、
`INPUTS` バッファ内の空きスロットを追跡するために通常の *フリーリスト*（リンクリスト）を使う
代わりに、ここでは SPSC キューを使います。これにより、クリティカルセクションの数を減らせます。
実際、この選択のおかげでタスクディスパッチのコードはロックフリーになっています。

## キュー容量

RTIC フレームワークは、レディキューやフリーキューのようないくつかのキューを使用します。フリーキュー
が空のときにタスクを `spawn` しようとするとエラーになり、この条件は実行時にチェックされます。
これらのキューに対してフレームワークが行うすべての操作が、キューが空 / 満杯かどうかをチェックする
わけではありません。たとえば、フリーキューにスロットを戻す操作（タスクディスパッチャを参照）は、
フリーキューの容量と等しい固定数のスロットがシステム内を循環しているため、チェックなしです。
同様に、レディキューへのエントリ追加（`Spawn` を参照）も、フレームワークが選んだキュー容量により
チェックなしです。

ユーザーはソフトウェアタスクの容量を指定できます。この容量は、`spawn` がエラーを返すまでに、
より高い優先度のタスクからそのタスクへポストできるメッセージの最大数です。ユーザーが指定したこの
容量は、そのタスクのフリーキュー（たとえば `foo_FQ`）の容量であり、同時にそのタスクへの入力を保持
する配列（たとえば `foo_INPUTS`）のサイズでもあります。

レディキュー（たとえば `RQ1`）の容量は、ディスパッチャが管理するすべての異なるタスクの容量の
*合計* になるように選ばれます。この合計は、タスクディスパッチャが実行の機会を得る前に、あり得る
すべてのメッセージがポストされるという最悪のシナリオにおいて、そのキューが保持するメッセージ数
でもあります。このため、どの `spawn` 操作でもフリーキューからスロットを取得できたということは、
レディキューがまだ満杯ではないことを意味するので、レディキューへのエントリ挿入では
「満杯か？」のチェックを省略できます。

ここまでの例では、タスク `bar` は入力を取らないので、`bar_INPUTS` と `bar_FQ` の両方を省略し、
このタスクに対してユーザーが無制限の数のメッセージをポストできるようにすることもできました。
しかし、そうすると、`baz` タスクを spawn するときに「満杯か？」のチェックを省略できるような
`RQ1` の容量を選ぶことができなくなります。[タイマーキュー](timer-queue.html) の節では、
入力を持たないタスクでフリーキューがどのように使われるかを見ていきます。

## 優先度上限解析

`spawn` API が内部で使用するキューは通常のリソースと同様に扱われ、優先度上限解析に含まれます。
重要なのは、これらが SPSC キューであり、片方のエンドポイントだけがリソースの背後にあり、もう
片方のエンドポイントはタスクディスパッチャが所有しているという点です。

次の例を考えてみましょう:

``` rust,noplayground
#[rtic::app(device = ..)]
mod app {
    #[idle(spawn = [foo, bar])]
    fn idle(c: idle::Context) -> ! {
        // ..
    }

    #[task]
    fn foo(c: foo::Context) {
        // ..
    }

    #[task]
    fn bar(c: bar::Context) {
        // ..
    }

    #[task(priority = 2, spawn = [foo])]
    fn baz(c: baz::Context) {
        // ..
    }

    #[task(priority = 3, spawn = [bar])]
    fn quux(c: quux::Context) {
        // ..
    }
}
```

優先度上限解析は次のようになります:

- `idle`（prio = 0）と `baz`（prio = 2）は `foo_FQ` のコンシューマエンドポイントに競合します。
  この結果、優先度上限は `2` になります。

- `idle`（prio = 0）と `quux`（prio = 3）は `bar_FQ` のコンシューマエンドポイントに競合します。
  この結果、優先度上限は `3` になります。

- `idle`（prio = 0）、`baz`（prio = 2）、`quux`（prio = 3）はすべて `RQ1` のプロデューサ
  エンドポイントに競合します。この結果、優先度上限は `3` になります。
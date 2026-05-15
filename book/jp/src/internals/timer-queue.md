# タイマーキュー

タイマーキュー機能を使うと、ユーザーは将来のある時点で実行されるタスクをスケジュールできます。予想どおり、この機能もキューを使って実装されています。具体的には、スケジュールされたタスクを最も早い予定時刻順に保持する優先度付きキューです。この機能には、タイムアウト割り込みを設定できるタイマーが必要です。タイマーは、タスクの予定時刻が来たときに割り込みを発生させるために使われます。その時点で、タスクはタイマーキューから取り除かれ、適切なレディキューへ移されます。

これがコードでどのように実装されているかを見てみましょう。次のプログラムを考えてください。

``` rust,noplayground
#[rtic::app(device = ..)]
mod app {
    // ..

    #[task(capacity = 2, schedule = [foo])]
    fn foo(c: foo::Context, x: u32) {
        // このタスクが 1M サイクル後に再び実行されるようスケジュールする
        c.schedule.foo(c.scheduled + Duration::cycles(1_000_000), x + 1).ok();
    }

    extern "C" {
        fn UART0();
    }
}
```

## `schedule`

まず `schedule` API を見てみましょう。

``` rust,noplayground
mod foo {
    pub struct Schedule<'a> {
        priority: &'a Cell<u8>,
    }

    impl<'a> Schedule<'a> {
        // ユーザーにこれを改変してほしくないため unsafe かつ非表示
        #[doc(hidden)]
        pub unsafe fn priority(&self) -> &Cell<u8> {
            self.priority
        }
    }
}

mod app {
    type Instant = <path::to::user::monotonic::timer as rtic::Monotonic>::Instant;

    // `schedule` 可能なすべてのタスク
    enum T {
        foo,
    }

    struct NotReady {
        index: u8,
        instant: Instant,
        task: T,
    }

    // タイマーキューは `NotReady` タスクを格納する二分木の（最小）ヒープ
    static mut TQ: TimerQueue<U2> = ..;
    const TQ_CEILING: u8 = 1;

    static mut foo_FQ: Queue<u8, U2> = Queue::new();
    const foo_FQ_CEILING: u8 = 1;

    static mut foo_INPUTS: [MaybeUninit<u32>; 2] =
        [MaybeUninit::uninit(), MaybeUninit::uninit()];

    static mut foo_INSTANTS: [MaybeUninit<Instant>; 2] =
        [MaybeUninit::uninit(), MaybeUninit::uninit()];

    impl<'a> foo::Schedule<'a> {
        fn foo(&self, instant: Instant, input: u32) -> Result<(), u32> {
            unsafe {
                let priority = self.priority();
                if let Some(index) = lock(priority, foo_FQ_CEILING, || {
                    foo_FQ.split().1.dequeue()
                }) {
                    // `index` はこれらのバッファへの所有ポインタ
                    foo_INSTANTS[index as usize].write(instant);
                    foo_INPUTS[index as usize].write(input);

                    let nr = NotReady {
                        index,
                        instant,
                        task: T::foo,
                    };

                    lock(priority, TQ_CEILING, || {
                        TQ.enqueue_unchecked(nr);
                    });
                } else {
                    // 入力 / instant を格納する空き領域が残っていない
                    Err(input)
                }
            }
        }
    }
}
```

これは `Spawn` の実装と非常によく似ています。実際、同じ `INPUTS` バッファとフリーキュー（`FQ`）が `spawn` API と `schedule` API の間で共有されています。この 2 つの主な違いは、`schedule` ではタスクの実行予定時刻である `Instant` も別のバッファ（この場合は `foo_INSTANTS`）に格納することです。

`TimerQueue::enqueue_unchecked` は、単にエントリを最小ヒープに追加するだけではなく、追加された新しいエントリがキューの先頭になった場合にシステムタイマー割り込み（`SysTick`）も pend します。

## システムタイマー

システムタイマー割り込み（`SysTick`）は 2 つのことを担当します。タイマーキュー内で実行可能になったタスクを適切なレディキューに移すことと、次のタスクの予定時刻が来たときに発火するタイムアウト割り込みを設定することです。

関連するコードを見てみましょう。

``` rust,noplayground
mod app {
    #[no_mangle]
    fn SysTick() {
        const PRIORITY: u8 = 1;

        let priority = &Cell::new(PRIORITY);
        while let Some(ready) = lock(priority, TQ_CEILING, || TQ.dequeue()) {
            match ready.task {
                T::foo => {
                    // このタスクを `RQ1` レディキューへ移動する
                    lock(priority, RQ1_CEILING, || {
                        RQ1.split().0.enqueue_unchecked(Ready {
                           task: T1::foo,
                           index: ready.index,
                        })
                    });

                    // タスクディスパッチャを pend する
                    rtic::pend(Interrupt::UART0);
                }
            }
        }
    }
}
```

これはタスクディスパッチャに似ていますが、レディ状態のタスクを実行する代わりに、対応するレディキューにタスクを配置するだけです。そうすることで、タスクは正しい優先度で実行されます。

`TimerQueue::dequeue` は、`None` を返すときに新しいタイムアウト割り込みを設定します。これは `TimerQueue::enqueue_unchecked` と連動しており、基本的に `enqueue_unchecked` は新しいタイムアウト割り込みの設定という仕事を `SysTick` ハンドラに委譲しています。

## `cyccnt::Instant` と `cyccnt::Duration` の分解能と範囲

RTIC は、`DWT`（Data Watchpoint and Trace）のサイクルカウンタに基づく `Monotonic` 実装を提供します。`Instant::now` はこのタイマーのスナップショットを返します。これらの DWT スナップショット（`Instant`）は、タイマーキュー内のエントリを並べ替えるために使われます。サイクルカウンタは、コアクロック周波数で駆動される 32 ビットカウンタです。このカウンタは `(1 << 32)` クロックサイクルごとにラップアラウンドします。このカウンタに関連付けられた割り込みはないため、ラップアラウンド時に特筆すべきことは何も起こりません。

キュー内で `Instant` を順序付けするには、2 つの 32 ビット整数を比較する必要があります。ラップアラウンド動作を考慮するために、2 つの `Instant` の差 `a - b` を使い、その結果を 32 ビット符号付き整数として扱います。結果が 0 未満であれば `b` はより後の `Instant` であり、結果が 0 より大きければ `b` はより前の `Instant` です。これは、キュー内の最初の（最も早い）エントリの予定時刻（`Instant`）より `(1 << 31) - 1` サイクル大きい `Instant` にタスクをスケジュールすると、そのタスクがキュー内の誤った位置に挿入されることを意味します。このユーザーエラーを防ぐためのデバッグアサーションはいくつか用意されていますが、ユーザーは `(instant + duration_a) + duration_b` と書いて `Instant` をオーバーフローさせることができるため、完全には避けられません。

システムタイマー `SysTick` は、同じくコアクロック周波数で駆動される 24 ビットカウンタです。次にスケジュールされたタスクが `1 << 24` クロックサイクルより先にある場合、`1 << 24` サイクル後に発火する割り込みが設定されます。この処理は、次にスケジュールされたタスクが `SysTick` カウンタの範囲内に入るまで何度か繰り返される必要がある場合があります。

結論として、`Instant` と `Duration` はどちらも 1 コアクロックサイクルの分解能を持ち、`Duration` は実質的に `0..(1 << 31)`（終端を含まない）コアクロックサイクルという（半開）範囲を持ちます。

## キュー容量
タイマーキューの容量は、すべての
`schedule` 可能なタスクの容量の総和になるように選ばれます。レディキューの場合と同様に、これは
`INPUTS` バッファ内の空きスロットをいったん確保すれば、そのタスクをタイマーキューに挿入できることが保証されることを意味します。これにより、実行時チェックを省略できます。

## システムタイマーの優先度

システムタイマーの優先度はユーザーが設定できず、フレームワークによって選ばれます。より低い優先度のタスクがより高い優先度のタスクの実行を妨げないようにするため、システムタイマーの優先度はすべての
`schedule` 可能なタスクの中で最大のものにします。

これがなぜ必要なのかを理解するために、優先度 `2` と `3` の 2 つの既に
schedule 済みのタスクがほぼ同時に ready になったものの、先にレディキューへ移されるのが低い優先度のタスクである場合を考えてみましょう。たとえばシステムタイマーの優先度が `1` だったとすると、低い優先度 (`2`) のタスクを移動したあと、そのタスクは完了まで実行されます（システムタイマーより高い優先度であるため）。その結果、より高い優先度 (`3`) のタスクの実行が遅延します。このような状況を防ぐために、システムタイマーは
`schedule` 可能なタスクの中で最も高い優先度に一致していなければなりません。この例ではそれが `3` になります。

## シーリング解析

タイマーキューは、タスクを `schedule` できるすべてのタスクと `SysTick` ハンドラの間で共有されるリソースです。また、`schedule` API はフリーキューをめぐって
`spawn` API と競合します。これらすべてをシーリング解析で考慮する必要があります。

例として、次の例を考えてみましょう。

``` rust,noplayground
#[rtic::app(device = ..)]
mod app {
    #[task(priority = 3, spawn = [baz])]
    fn foo(c: foo::Context) {
        // ..
    }

    #[task(priority = 2, schedule = [foo, baz])]
    fn bar(c: bar::Context) {
        // ..
    }

    #[task(priority = 1)]
    fn baz(c: baz::Context) {
        // ..
    }
}
```

シーリング解析は次のようになります。

- `foo` (prio = 3) と `baz` (prio = 1) は `schedule` 可能なタスクなので、
  `SysTick` はこの 2 つのうち高い方の優先度、つまり `3` で実行されなければなりません。

- `foo::Spawn` (prio = 3) と `bar::Schedule` (prio = 2) は
  `baz_FQ` のコンシューマ側エンドポイントをめぐって競合します。これにより、優先度シーリングは `3` になります。

- `bar::Schedule` (prio = 2) は
`foo_FQ` のコンシューマ側エンドポイントに対して排他的アクセスを持ちます。したがって、`foo_FQ` の優先度シーリングは実質的に `2` です。

- `SysTick` (prio = 3) と `bar::Schedule` (prio = 2) はタイマー
  キュー `TQ` をめぐって競合します。これにより、優先度シーリングは `3` になります。

- `SysTick` (prio = 3) と `foo::Spawn` (prio = 3) はどちらも、
  `foo` のエントリを保持するレディキュー `RQ3` にロックフリーでアクセスします。したがって、
  `RQ3` の優先度シーリングは実質的に `3` です。

- `SysTick` は `baz` のエントリを保持するレディキュー `RQ1` に対して排他的アクセスを持ちます。
  したがって、`RQ1` の優先度シーリングは実質的に `3` です。

## `spawn` 実装の変更

`schedule` API を使う場合、タスクの基準時刻を追跡するために `spawn` 実装が少し変わります。
`schedule` 実装で見たように、タスクが実行されるよう schedule された時刻を保存するための
`INSTANTS` バッファがあります。この `Instant` はタスクディスパッチャで読み出され、タスクコンテキストの一部としてユーザーコードに渡されます。

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
                    let input = baz_INPUTS[ready.index as usize].read();
                    // 追加
                    let instant = baz_INSTANTS[ready.index as usize].read();

                    baz_FQ.split().0.enqueue_unchecked(ready.index);

                    let priority = Cell::new(PRIORITY);
                    // 変更: instant はタスクコンテキストの一部として渡される
                    baz(baz::Context::new(&priority, instant), input)
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

逆に、`spawn` 実装では `INSTANTS`
バッファに値を書き込む必要があります。書き込まれる値は `Spawn` 構造体に格納されており、それは
ハードウェアタスクの `start` 時刻、またはソフトウェアタスクの `scheduled` 時刻です。

``` rust,noplayground
mod foo {
    // ..

    pub struct Spawn<'a> {
        priority: &'a Cell<u8>,
        // 追加
        instant: Instant,
    }

    impl<'a> Spawn<'a> {
        pub unsafe fn priority(&self) -> &Cell<u8> {
            &self.priority
        }

        // 追加
        pub unsafe fn instant(&self) -> Instant {
            self.instant
        }
    }
}

mod app {
    impl<'a> foo::Spawn<'a> {
        /// `baz` タスクを spawn します
        pub fn baz(&self, message: u64) -> Result<(), u64> {
            unsafe {
                match lock(self.priority(), baz_FQ_CEILING, || {
                    baz_FQ.split().1.dequeue()
                }) {
                    Some(index) => {
                        baz_INPUTS[index as usize].write(message);
                        // 追加
                        baz_INSTANTS[index as usize].write(self.instant());

                        lock(self.priority(), RQ1_CEILING, || {
                            RQ1.split().1.enqueue_unchecked(Ready {
                                task: Task::foo,
                                index,
                            });
                        });

                        rtic::pend(Interrupt::UART0);
                    }

                    None => {
                        // 最大容量に達したため、spawn に失敗した
                        Err(message)
                    }
                }
            }
        }
    }
}
```
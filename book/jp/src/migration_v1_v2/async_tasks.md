# `async` ソフトウェアタスクの使用

ソフトウェアタスクにはいくつかの変更があります。以下に概要を示します。

### ソフトウェアタスクは `async` でなければならなくなりました。

すべてのソフトウェアタスクは `async` であることが必須になりました。

#### 必要な変更。

プロジェクト内で割り込みにバインドされていないタスクは、すべて `async fn` でなければならなくなりました。たとえば:

``` rust,noplayground
#[task(
    local = [ some_resource ],
    shared = [ my_shared_resource ],
    priority = 2
)]
fn my_task(cx: my_task::Context) {
    cx.local.some_resource.do_trick();
    cx.shared.my_shared_resource.lock(|s| s.do_shared_thing());
}
```

これは次のようになります

``` rust,noplayground
#[task(
    local = [ some_resource ],
    shared = [ my_shared_resource ],
    priority = 2
)]
async fn my_task(cx: my_task::Context) {
    cx.local.some_resource.do_trick();
    cx.shared.my_shared_resource.lock(|s| s.do_shared_thing());
}
```

## ソフトウェアタスクは無限に実行できるようになりました

新しい `async` ソフトウェアタスクは、1 つの前提条件のもとで無限に実行することが許可されています。**タスクの無限ループ内に `await` が存在しなければなりません**。そのようなタスクの例を次に示します:

``` rust,noplayground
#[task(local = [ my_channel ] )]
async fn my_task_that_runs_forever(cx: my_task_that_runs_forever::Context) {
    loop {
        let value = cx.local.my_channel.recv().await;
        do_something_with_value(value);
    }
}
```

## `spawn_after` と `spawn_at` は削除されました。

[`rtic-monotonics` への移行](./monotonics.md) の章で説明したとおり、`spawn_after` と `spawn_at` はもう利用できません。
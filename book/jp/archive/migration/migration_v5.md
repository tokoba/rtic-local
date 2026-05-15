# v0.5.x から v1.0.0 への移行

このセクションでは、RTIC フレームワークの v0.5.x から v1.0.0 へアップグレードする方法を説明します。

## `Cargo.toml` - バージョン更新

`cortex-m-rtic` のバージョンを `"1.0.0"` に変更します。

## `const` ではなく `mod`

モジュールへの属性付与がサポートされたため、`const APP` による回避策は不要になりました。

次のコードを

``` rust,noplayground
#[rtic::app(/* .. */)]
const APP: () = {
  [code here]
};
```

次のように変更します。

``` rust,noplayground
#[rtic::app(/* .. */)]
mod app {
  [code here]
}
```

通常の Rust モジュールを使用するようになったため、そのモジュール内にユーザー独自のコードを含められるようになりました。
また、ユーザーコードで使用する項目に対する `use` 文は `mod app` の内側に移動するか、`super` を使って参照する必要があります。たとえば、次のコードを変更します。

```rust
use some_crate::some_func;

#[rtic::app(/* .. */)]
const APP: () = {
    fn func() {
        some_crate::some_func();
    }
};
```

次のように変更します。

```rust
#[rtic::app(/* .. */)]
mod app {
    use some_crate::some_func;

    fn func() {
        some_crate::some_func();
    }
}
```

または

```rust
use some_crate::some_func;

#[rtic::app(/* .. */)]
mod app {
    fn func() {
        super::some_crate::some_func();
    }
}
```

## ディスパッチャを `extern "C"` から app 引数へ移動

次のコードを

``` rust,noplayground
#[rtic::app(/* .. */)]
const APP: () = {
    [code here]

    // RTIC では、ソフトウェアタスクを使用する場合、未使用の割り込みを extern ブロックで
    // 宣言する必要がある。これらの空いている割り込みは、ソフトウェアタスクを
    // ディスパッチするために使用される。
    extern "C" {
        fn SSI0();
        fn QEI0();
    }
};
```

次のように変更します。

``` rust,noplayground
#[rtic::app(/* .. */, dispatchers = [SSI0, QEI0])]
mod app {
  [code here]
}
```

これは RAM 関数にも適用されます。examples/ramfunc.rs を参照してください。

## リソース構造体 - `#[shared]`, `#[local]`

以前は、RTIC のリソースは正確に "Resources" という名前の構造体内になければなりませんでした。

``` rust,noplayground
struct Resources {
    // ここでリソースを定義する
}
```

RTIC v1.0.0 では、リソース構造体は `#[task]`、`#[init]`、`#[idle]` と同様に、`#[shared]` および `#[local]` 属性で注釈します。

``` rust,noplayground
#[shared]
struct MySharedResources {
    // タスク間で共有されるリソースをここで定義する
}

#[local]
struct MyLocalResources {
    // ここで定義されたリソースはタスク間で共有できない。各リソースは単一のタスクにローカル
}
```

これらの構造体の名前は開発者が自由に付けられます。

## `#[task]` における `shared` と `local` 引数

v1.0.0 では、リソースは `shared` リソースと `local` リソースに分割されます。
`#[task]`、`#[init]`、`#[idle]` に `resources` 引数はなくなり、代わりに `shared` 引数と `local` 引数を使用する必要があります。

v0.5.x では:

``` rust,noplayground
struct Resources {
    local_to_b: i64,
    shared_by_a_and_b: i64,
}

#[task(resources = [shared_by_a_and_b])]
fn a(_: a::Context) {}

#[task(resources = [shared_by_a_and_b, local_to_b])]
fn b(_: b::Context) {}
```

v1.0.0 では:

``` rust,noplayground
#[shared]
struct Shared {
    shared_by_a_and_b: i64,
}

#[local]
struct Local {
    local_to_b: i64,
}

#[task(shared = [shared_by_a_and_b])]
fn a(_: a::Context) {}

#[task(shared = [shared_by_a_and_b], local = [local_to_b])]
fn b(_: b::Context) {}
```

## 対称ロック

RTIC は対称ロックを利用するようになったため、すべての `shared` リソースアクセスで `lock` メソッドを使用する必要があります。
以前のコードでは、高優先度タスクがそのリソースへの排他的アクセス権を持っていたため、次のように書けました。

``` rust,noplayground
#[task(priority = 2, resources = [r])]
fn foo(cx: foo::Context) {
    cx.resources.r = /* ... */;
}

#[task(resources = [r])]
fn bar(cx: bar::Context) {
    cx.resources.r.lock(|r| r = /* ... */);
}
```

対称ロックでは、両方のタスクでロックを使用する必要があります。

``` rust,noplayground
#[task(priority = 2, shared = [r])]
fn foo(cx: foo::Context) {
    cx.shared.r.lock(|r| r = /* ... */);
}

#[task(shared = [r])]
fn bar(cx: bar::Context) {
    cx.shared.r.lock(|r| r = /* ... */);
}
```

なお、不要なロックは LLVM の最適化によって取り除かれるため、パフォーマンスは変わりません。

## ロックフリーのリソースアクセス

RTIC 0.5 では、同じ優先度で実行されるタスク間で共有されるリソースには、`lock` API を使わ*ずに*アクセスできました。
これは 1.0 でも可能です。`#[shared]` リソースのフィールドに `#[lock_free]` 属性を付ける必要があります。

v0.5 のコード:

``` rust,noplayground
struct Resources {
    counter: u64,
}

#[task(resources = [counter])]
fn a(cx: a::Context) {
    *cx.resources.counter += 1;
}

#[task(resources = [counter])]
fn b(cx: b::Context) {
    *cx.resources.counter += 1;
}
```

v1.0 のコード:

``` rust,noplayground
#[shared]
struct Shared {
    #[lock_free]
    counter: u64,
}

#[task(shared = [counter])]
fn a(cx: a::Context) {
    *cx.shared.counter += 1;
}

#[task(shared = [counter])]
fn b(cx: b::Context) {
    *cx.shared.counter += 1;
}
```

## `static mut` 変換はなくなった

`static mut` 変数は、安全な `&'static mut` 参照へは変換されなくなりました。
その構文の代わりに、`#[init]` の `local` 引数を使用してください。

v0.5.x のコード:

``` rust,noplayground
#[init]
fn init(_: init::Context) {
    static mut BUFFER: [u8; 1024] = [0; 1024];
    let buffer: &'static mut [u8; 1024] = BUFFER;
}
```

v1.0.0 のコード:

``` rust,noplayground
#[init(local = [
    buffer: [u8; 1024] = [0; 1024]
//   型   ^^^^^^^^^^^^   ^^^^^^^^^ 初期値
])]
fn init(cx: init::Context) -> (Shared, Local, init::Monotonics) {
    let buffer: &'static mut [u8; 1024] = cx.local.buffer;

    (Shared {}, Local {}, init::Monotonics())
}
```

## Init は常に late リソースを返す

API をより対称的にするために、`#[init]` タスクは常に late リソースを返すようになりました。

次のコードを:

``` rust,noplayground
#[rtic::app(device = lm3s6965)]
const APP: () = {
    #[init]
    fn init(_: init::Context) {
        rtic::pend(Interrupt::UART0);
    }

    // [さらにコード]
};
```

次のように変更します。

``` rust,noplayground
#[rtic::app(device = lm3s6965)]
mod app {
    #[shared]
    struct MySharedResources {}

    #[local]
    struct MyLocalResources {}

    #[init]
    fn init(_: init::Context) -> (MySharedResources, MyLocalResources, init::Monotonics) {
        rtic::pend(Interrupt::UART0);

        (MySharedResources, MyLocalResources, init::Monotonics())
    }

    // [さらにコード]
}
```

## どこからでも Spawn

新しい spawn/spawn_after/spawn_at インターフェースでは、次のように spawn のためにコンテキスト `cx` を必要としていた従来のコードは:

``` rust,noplayground
#[task(spawn = [bar])]
fn foo(cx: foo::Context) {
    cx.spawn.bar().unwrap();
}

#[task(schedule = [bar])]
fn bar(cx: bar::Context) {
    cx.schedule.foo(/* ... */).unwrap();
}
```

次のように書くようになります。

``` rust,noplayground
#[task]
fn foo(_c: foo::Context) {
    bar::spawn().unwrap();
}

#[task]
fn bar(_c: bar::Context) {
    // “現在” からの相対値として Duration を受け取る
    let spawn_handle = foo::spawn_after(/* ... */);
}

#[task]
fn bar(_c: bar::Context) {
    // Instant を受け取る
    let spawn_handle = foo::spawn_at(/* ... */);
}
```

これにより、コンテキストへアクセスできる必要はなくなりました。

なお、タスク定義内の `spawn` / `schedule` 属性は不要になりました。

---

## 追加機能

### Extern タスク

ソフトウェアタスクとハードウェアタスクの両方を、`mod app` の外部で定義できるようになりました。
以前は、タスク実装を呼び出すトランポリンを実装した場合にのみ可能でした。

例として `examples/extern_binds.rs` と `examples/extern_spawn.rs` を参照してください。

これにより、アプリを複数のファイルに分割できます。
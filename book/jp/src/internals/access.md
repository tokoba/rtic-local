# アクセス制御

RTIC の中核的な基盤の 1 つがアクセス制御です。プログラムのどの部分がどの static 変数にアクセスできるかを制御することは、メモリ安全性を強制するうえで極めて重要です。

static 変数は、割り込みハンドラ間、あるいは割り込みハンドラと下位の実行コンテキストである `main` の間で状態を共有するために使われます。通常の Rust コードでは、どの関数が static 変数にアクセスできるかを細かく制御するのは困難です。これは、static 変数が、それらが宣言されたのと同じスコープ内にある任意の関数からアクセスできるためです。モジュールによって static 変数へのアクセス方法をある程度制御できますが、十分に柔軟ではありません。

タスクが自分の RTIC 属性で指定した static 変数（リソース）にしかアクセスできないようにする、このきめ細かなアクセス制御を実現するために、RTIC フレームワークはソースコードレベルの変換を行います。この変換は、ユーザーが指定したリソース（static 変数）をモジュールの *内側* に配置し、ユーザーコードをモジュールの *外側* に配置することで成り立っています。
これにより、ユーザーコードがこれらの static 変数を参照することは不可能になります。

その後、各タスクには、そのタスクがアクセスできるリソースに対応するフィールドを持つ `Resources` 構造体を使ってリソースへのアクセスが与えられます。このような構造体はタスクごとに 1 つあり、`Resources` 構造体は static 変数への一意な参照（`&mut-`）またはリソースプロキシ（[critical sections](critical-sections.html) の節を参照）のいずれかで初期化されます。

以下のコードは、舞台裏で行われるこの種のソースレベル変換の例です。

``` rust,noplayground
#[rtic::app(device = ..)]
mod app {
    static mut X: u64: 0;
    static mut Y: bool: 0;

    #[init(resources = [Y])]
    fn init(c: init::Context) {
        // .. ユーザーコード ..
    }

    #[interrupt(binds = UART0, resources = [X])]
    fn foo(c: foo::Context) {
        // .. ユーザーコード ..
    }

    #[interrupt(binds = UART1, resources = [X, Y])]
    fn bar(c: bar::Context) {
        // .. ユーザーコード ..
    }

    // ..
}
```

フレームワークは次のようなコードを生成します。

``` rust,noplayground
fn init(c: init::Context) {
    // .. ユーザーコード ..
}

fn foo(c: foo::Context) {
    // .. ユーザーコード ..
}

fn bar(c: bar::Context) {
    // .. ユーザーコード ..
}

// パブリック API
pub mod init {
    pub struct Context<'a> {
        pub resources: Resources<'a>,
        // ..
    }

    pub struct Resources<'a> {
        pub Y: &'a mut bool,
    }
}

pub mod foo {
    pub struct Context<'a> {
        pub resources: Resources<'a>,
        // ..
    }

    pub struct Resources<'a> {
        pub X: &'a mut u64,
    }
}

pub mod bar {
    pub struct Context<'a> {
        pub resources: Resources<'a>,
        // ..
    }

    pub struct Resources<'a> {
        pub X: &'a mut u64,
        pub Y: &'a mut bool,
    }
}

/// 実装の詳細
mod app {
    // このモジュール内のすべてはユーザーコードから隠される

    static mut X: u64 = 0;
    static mut Y: bool = 0;

    // プログラムの実際のエントリポイント
    unsafe fn main() -> ! {
        interrupt::disable();

        // ..

        // ユーザーコードを呼び出す。static 変数への参照を渡す
        init(init::Context {
            resources: init::Resources {
                X: &mut X,
            },
            // ..
        });

        // ..

        interrupt::enable();

        // ..
    }

    // `foo` がバインドされる割り込みハンドラ
    #[no_mangle]
    unsafe fn UART0() {
        // ユーザーコードを呼び出す。static 変数への参照を渡す
        foo(foo::Context {
            resources: foo::Resources {
                X: &mut X,
            },
            // ..
        });
    }

    // `bar` がバインドされる割り込みハンドラ
    #[no_mangle]
    unsafe fn UART1() {
        // ユーザーコードを呼び出す。static 変数への参照を渡す
        bar(bar::Context {
            resources: bar::Resources {
                X: &mut X,
                Y: &mut Y,
            },
            // ..
        });
    }
}
```
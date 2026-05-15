# 内部の仕組み

**この章は現在作業中です。
より完成度が高まった段階で再掲されます**

このセクションでは、RTIC framework の内部実装を*高レベル*で説明します。
手続きマクロ（`#[app]`）によって行われるパースやコード生成といった
低レベルの詳細については、ここでは説明しません。主な焦点は、
ユーザー仕様の解析と、ランタイムで使用されるデータ構造です。

この内容に入る前に、embedonomicon の [concurrency] に関するセクションを
読んでおくことを強く推奨します。

[concurrency]: https://github.com/rust-embedded/embedonomicon/pull/48
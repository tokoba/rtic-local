# RTFM から RTIC への移行

このセクションでは、RTFM v0.5.x 向けに書かれたアプリケーションを、同じバージョンの RTIC にアップグレードする方法を説明します。これは、[RFC #33] に従ってフレームワークの名称が変更されたためです。

**注:** RTFM v0.5.3 と RTIC v0.5.3 の間にコード上の違いはなく、純粋に名称変更のみです。

[RFC #33]: https://github.com/rtic-rs/rfcs/pull/33

## `Cargo.toml`

まず、`cortex-m-rtfm` 依存関係を `cortex-m-rtic` に更新する必要があります。

``` toml
[dependencies]
# これを変更
cortex-m-rtfm = "0.5.3"

# これに
cortex-m-rtic = "0.5.3"
```

## コードの変更

必要なコード変更は、これまで `rtfm` を参照していた箇所を、次のように `rtic` に変更することだけです。

``` rust,noplayground
//
// これを変更
//

#[rtfm::app(/* .. */, monotonic = rtfm::cyccnt::CYCCNT)]
const APP: () = {
    // ...

};

//
// これに変更
//

#[rtic::app(/* .. */, monotonic = rtic::cyccnt::CYCCNT)]
const APP: () = {
    // ...

};
```
# 新しいプロジェクトを開始する

RTIC プロジェクトをゼロから開始する場合の推奨事項として、
RTIC の [`defmt-app-template`] に従うことをお勧めします。

ARMv6-M または ARMv8-M-base アーキテクチャを対象とする場合は、注意すべきハードウェア上の制約の詳細について [ターゲットアーキテクチャ](./internals/targets.md) の節を参照してください。

[`defmt-app-template`]: https://github.com/rtic-rs/defmt-app-template

これにより、[`defmt`] による RTT ロギングのサポートと、[`flip-link`] を使用したスタックオーバーフロー
保護を備えた RTIC アプリケーションを利用できます。コミュニティによって提供されている多数のサンプルもあります:

参考として、[RTIC examples] を参照してください。

## RISC-V デバイスでの RTIC

RTIC は当初 ARM Cortex-M 向けに開発されましたが、RISC-V デバイスでも RTIC を使用できます。
ただし、RISC-V のエコシステムはより異種混在的です。
この問題に対処するため、現在 RTIC は 3 種類の異なるバックエンドを実装しています:

- **`riscv-esp32c3-backend`**: このバックエンドは ESP32-C3 SoC をサポートします。
  これらのデバイスでは、RTIC は Cortex-M 版と非常によく似ています。

- **`riscv-esp32c6-backend`**: このバックエンドは ESP32-C6 SoC をサポートします。
  これらのデバイスでは、RTIC は Cortex-M 版と非常によく似ています。

- **`riscv-mecall-backend`**: このバックエンドは **あらゆる** RISC-V デバイスをサポートします。
  このバックエンドでは、保留中のタスクが Machine Environment Call 例外をトリガーします。
  この例外ソースのハンドラは、優先度に従って保留中のタスクをディスパッチします。
  このバックエンドの動作は `riscv-clint-backend` と同等です。
  このバックエンドの主な違いは、すべてのタスクが **必ず** [ソフトウェアタスク](./by-example/software_tasks.md) でなければならないことです。
  さらに、`#[app]` 属性にディスパッチャの一覧を指定する必要はありません。RTIC がコンパイル時にそれらを生成するためです。

- **`riscv-clint-backend`**: このバックエンドは CLINT ペリフェラルを備えたデバイスをサポートします。
  これは `riscv-mecall-backend` と同等ですが、例外をトリガーする代わりに、CLINT の `MSIP` レジスタを介してソフトウェア割り込みをトリガーします。

[`defmt`]: https://github.com/knurling-rs/defmt/
[`flip-link`]: https://github.com/knurling-rs/flip-link/
[RTIC examples]: https://github.com/rtic-rs/rtic/tree/master/examples
# RTIC vs. Embassy

## 違い

Embassy はハードウェア抽象化レイヤーとエグゼキューター/ランタイムの両方を提供しますが、RTIC は実行フレームワークのみを提供することを目指しています。たとえば、embassy は `embassy-stm32`（HAL）と `embassy-executor`（エグゼキューター）を提供します。一方、RTIC は [`rtic`] の形でフレームワークを提供し、PAC と HAL の実装はユーザーが用意する責任があります（一般的には [`stm32-rs`] プロジェクトのものを使用します）。

さらに、RTIC は可能な限り低いレベルでリソースへの排他的アクセスを提供することを目指しており、理想的には何らかの形のハードウェア保護によって保護されます。これにより、ソフトウェアレベルのロック機構を必ずしも必要とせずにハードウェアへアクセスできます。

## Embassy と RTIC の併用

ほとんどの Embassy および RTIC のライブラリはランタイム非依存であるため、一方のプロジェクトの多くの要素はもう一方でも使用できます。たとえば、`embassy-executor` を使用するプロジェクトで [`rtic-monotonics`] を使用できますし、RTIC プロジェクトで [`embassy-sync`]（ただし [`rtic-sync`] を推奨）を使用することもできます。

[`stm32-rs`]: https://github.com/stm32-rs
[`rtic`]: https://docs.rs/rtic/latest/rtic/
[`rtic-monotonics`]: https://docs.rs/rtic-monotonics/latest/rtic_monotonics/
[`embassy-sync`]: https://docs.rs/embassy-sync/latest/embassy_sync/
[`rtic-sync`]: https://docs.rs/rtic-sync/latest/rtic_sync/
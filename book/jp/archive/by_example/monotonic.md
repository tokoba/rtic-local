# Monotonic と spawn_{at/after}

時間を理解することは組み込みシステムにおける重要な概念であり、時間に基づいてタスクを実行
できることは不可欠です。フレームワークは静的メソッド
`task::spawn_after(/* duration */)` と `task::spawn_at(/* specific time instant */)` を提供します。
`spawn_after` の方が一般的に使われますが、ドリフトなしで、あるいは固定された基準時刻に対して
スポーンを発生させる必要がある場合には `spawn_at` を利用できます。

これをサポートするために、型エイリアス定義に適用される
`#[monotonic]` 属性が存在します。
この型エイリアスは、[`rtic_monotonic::Monotonic`] トレイトを実装する型を指していなければなりません。
これは一般に、システムのタイミングを扱う何らかのタイマーです。
同じシステム内では 1 つ以上の Monotonic を共存させることができ、たとえばスリープからシステムを
復帰させる低速タイマーと、システムが起きている間に高精度なスケジューリングを行うことを
目的とした別のタイマーを併用できます。

[`rtic_monotonic::Monotonic`]: https://docs.rs/rtic-monotonic

この属性には、必須パラメータ `binds` と、オプションパラメータ `default` および
`priority` があります。
必須パラメータ `binds = InterruptName` は、割り込みベクタをタイマーの
割り込みに対応付けます。一方、`default = true` はスポーン時および時刻アクセス時の
簡略 API（`monotonics::now()` と `monotonics::MyMono::now()`）を有効にし、`priority` は
割り込みベクタの優先度を設定します。

> デフォルトの `priority` は、システムの **最大優先度** です。
> システムに厳しいスケジューリング要件を持つ高優先度タスクがある場合、
> 高優先度タスクのスケジューリングジッタを減らすために `monotonic` タスクの優先度を
> より低く下げることが望ましい場合があります。
> ただし、これにより `monotonic` 経由のスケジューリングにジッタや遅延が生じる可能性があり、
> そのためトレードオフになります。

Monotonic は `#[init]` で初期化され、`init::Monotonic( ... )` タプル内で返されます。
これにより Monotonic が有効化され、使用できるようになります。

次の例を参照してください。

``` rust,noplayground
{{#include ../../../../examples/schedule.rs}}
```

``` console
$ cargo xtask qemu --verbose --example schedule
{{#include ../../../../ci/expected/schedule.run}}
```

Monotonic の重要な要件の 1 つは、ハードウェアタイマーのオーバーランを
適切に処理できなければならないことです。

## スケジュール済みタスクのキャンセルまたは再スケジュール

`task::spawn_after` と `task::spawn_at` を使ってスポーンされたタスクは `SpawnHandle` を返し、
これにより将来実行されるようスケジュールされたタスクのキャンセルや再スケジュールが可能になります。

`cancel` または `reschedule_at`/`reschedule_after` が `Err` を返した場合、それは操作が
遅すぎて、タスクがすでに実行のために送られていることを意味します。次の例はその動作を示します。

``` rust,noplayground
{{#include ../../../../examples/cancel-reschedule.rs}}
```

``` console
$ cargo xtask qemu --verbose --example cancel-reschedule
{{#include ../../../../ci/expected/cancel-reschedule.run}}
```
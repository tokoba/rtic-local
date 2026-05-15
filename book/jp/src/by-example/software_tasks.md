# ソフトウェアタスクと spawn

RTIC におけるソフトウェアタスクの概念は、[ハードウェアタスク](./hardware_tasks.md) と多くの共通点があります。中核となる違いは、ソフトウェアタスクが特定の割り込みベクタに明示的に束縛されるのではなく、ソフトウェアタスクの意図した優先度で動作する「ディスパッチャ」割り込みベクタに束縛される点です（以下を参照）。

_ハードウェア_ タスクと同様に、関数に付ける `#[task]` 属性はその関数をタスクとして宣言します。この属性に `binds = InterruptName` 引数がない場合、その関数は _ソフトウェアタスク_ として宣言されます。

静的メソッド `task_name::spawn()` はソフトウェアタスクを spawn（開始）し、より高い優先度のタスクが実行中でなければ、そのタスクは直ちに実行を開始します。

_ソフトウェア_ タスク自体は `async` な Rust 関数として与えられ、これによりユーザーは将来のイベントを任意に `await` できます。これにより、リアクティブプログラミング（_ハードウェア_ タスクによる）と逐次的なプログラミング（_ソフトウェア_ タスクによる）を組み合わせられます。

_ハードウェア_ タスクは run-to-completion で実行されて終了するものと想定される一方、_ソフトウェア_ タスクは一度開始（`spawn`）されると、任意のループ（実行パス）が少なくとも 1 つの `await`（yield 操作）によって中断されることを条件に、永続的に実行できます。

## ディスパッチャ

同じ優先度レベルにあるすべての _ソフトウェア_ タスクは、ソフトウェアタスクをディスパッチする async executor として動作する 1 つの割り込みハンドラを共有します。このディスパッチャのリスト `dispatchers = [FreeInterrupt1, FreeInterrupt2, ...]` は `#[app]` 属性の引数であり、そこで空いていて使用可能な割り込みの集合を定義します。

ディスパッチャとして機能する各割り込みベクタには 1 つの優先度レベルが割り当てられるため、ディスパッチャのリストはソフトウェアタスクで使用されるすべての優先度レベルをカバーする必要があります。

例: ソフトウェアタスクに 3 つの異なる優先度を使用するアプリケーションでは、`dispatchers =` 引数には少なくとも 3 つのエントリが必要です。

提供されたディスパッチャが不足している場合、またはディスパッチャのリストと _ハードウェア_ タスクに束縛された割り込みの間で衝突が発生した場合、フレームワークはコンパイルエラーを出します。

以下の例を参照してください。

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/spawn.rs}}
```

```console
$ cargo xtask qemu --verbose --example spawn
```

```console
{{#include ../../../../ci/expected/lm3s6965/spawn.run}}
```

_ソフトウェア_ タスクが run-to-completion で終了していれば、そのタスクを再度 `spawn` できます。

以下の例では、`idle` タスクから _ソフトウェア_ タスク `foo` を `spawn` しています。_ソフトウェア_ タスクの優先度は 1（`idle` より高い）なので、ディスパッチャは `foo` を実行します（`idle` をプリエンプトします）。`foo` は run-to-completion で終了するため、`foo` タスクを再度 `spawn` しても問題ありません。

技術的には、async executor は `foo` の _future_ を `poll` し、この場合その _future_ は _completed_ 状態になります。

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/spawn_loop.rs}}
```

```console
$ cargo xtask qemu --verbose --example spawn_loop
```

```console
{{#include ../../../../ci/expected/lm3s6965/spawn_loop.run}}
```

すでに `spawn` 済みのタスク（実行中のタスク）を `spawn` しようとすると、エラーになります。ここで注目すべき点は、このエラーが `foo` タスクが実際に実行される前に報告されることです。これは、_ソフトウェア_ タスクの実際の実行がディスパッチャ割り込み（`SSIO`）によって処理される一方で、その割り込みは `init` タスクを抜けるまで有効にならないためです。（`init` はクリティカルセクションで実行され、つまりすべての割り込みが無効化されていることを思い出してください。）

技術的には、_completed_ 状態にない _future_ に対する `spawn` はエラーと見なされます。

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/spawn_err.rs}}
```

```console
$ cargo xtask qemu --verbose --example spawn_err
```

```console
{{#include ../../../../ci/expected/lm3s6965/spawn_err.run}}
```

## 引数の受け渡し

次のように、`spawn` 時に引数を渡すこともできます。

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/spawn_arguments.rs}}
```

```console
$ cargo xtask qemu --verbose --example spawn_arguments
```

```console
{{#include ../../../../ci/expected/lm3s6965/spawn_arguments.run}}
```

## 発散タスク

タスクは 2 つのシグネチャのいずれかを取れます: `async fn({name}::Context, ..)` または `async fn({name}::Context, ..) -> !`。後者は *発散* タスク、つまり決して return しないタスクを定義します。発散タスクの主な利点は、`'static` なコンテキストを受け取り、`local` リソースが `'static` ライフタイムを持つことです。さらに、このシグネチャを使うことでタスクの意図が明示され、短命なタスクと無期限に実行されるタスクを明確に区別できます。`.await` によって制御を明け渡し、同じ優先度レベルの他のタスクを飢餓状態にしないよう注意してください。

## 優先度ゼロのタスク

RTIC では、タスクは互いにプリエンプティブに実行され、優先度ゼロ（0）が最も低い優先度です。優先度ゼロのタスクは、厳密なリアルタイム要件のないバックグラウンド処理に使用できます。

概念的には、そのようなタスクはアプリケーションの `main` スレッドで実行されるものと見なせるため、関連するリソースには [Send] トレイト境界は要求されません。

[Send]: https://doc.rust-lang.org/nomicon/send-and-sync.html

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/zero-prio-task.rs}}
```

```console
$ cargo xtask qemu --verbose --example zero-prio-task
```

```console
{{#include ../../../../ci/expected/lm3s6965/zero-prio-task.run}}
```

> **注意**: 優先度ゼロの _ソフトウェア_ タスクは [idle] タスクと共存できません。理由は、`idle` が優先度ゼロの、復帰しない Rust 関数として実行されるためです。したがって、優先度ゼロの executor が同じ優先度の _ソフトウェア_ タスクに制御を渡す方法がありません。

---

アプリケーション側の安全性: 技術的には、RTIC フレームワークは _completed_ 状態の future を持つ _ソフトウェア_ タスクに対して `poll` が決して実行されないことを保証しており、これにより async Rust の健全性の規則に従います。
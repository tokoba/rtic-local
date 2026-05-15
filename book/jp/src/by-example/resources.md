# リソースの使用

RTIC フレームワークは共有リソースとタスクローカルリソースを管理し、`unsafe` コードを使用せずに永続的なデータ保存と安全なアクセスを可能にします。

RTIC のリソースは `#[app]` モジュール内で宣言された関数からのみ参照可能であり、フレームワークはリソースのアクセス可能性を（タスクごとに）ユーザーが完全に制御できるようにします。

システム全体のリソースは、`#[app]` モジュール内の **2 つ** の `struct` に `#[local]` および `#[shared]` 属性を付けることで宣言します。これらの構造体の各フィールドは、それぞれ異なるリソース（フィールド名で識別）に対応します。この 2 種類のリソースの違いについては以下で説明します。

各タスクは、対応するメタデータ属性で `local` 引数と `shared` 引数を使って、アクセスする予定のリソースを宣言する必要があります。各引数にはリソース識別子のリストを指定します。列挙したリソースは、`Context` 構造体の `local` フィールドおよび `shared` フィールドを通じてコンテキストから利用可能になります。

`init` タスクは、システム全体の（`#[shared]` および `#[local]`）リソースの初期値を返します。

<!-- およびアプリケーションが使用する初期化済みタイマーの集合。モノトニックタイマーについては
[Monotonic と `spawn_{at/after}`](./monotonic.md) でさらに説明します。 -->

## `#[local]` リソース

`#[local]` リソースは特定のタスクからローカルにアクセス可能なリソースであり、そのタスクだけがロックやクリティカルセクションなしでそのリソースにアクセスできます。これにより、通常はドライバーや大きなオブジェクトであるリソースを `#[init]` で初期化し、その後特定のタスクに渡すことができます。

したがって、あるタスクの `#[local]` リソースには、ただ 1 つのタスクだけがアクセスできます。同じ `#[local]` リソースを複数のタスクに割り当てようとすると、コンパイル時エラーになります。

`#[local]` リソースの型は、`init` から対象タスクへ送られ、スレッド境界をまたぐため、[`Send`] トレイトを実装していなければなりません。

[`Send`]: https://doc.rust-lang.org/stable/core/marker/trait.Send.html

以下に示すアプリケーション例には、`foo`、`bar`、`idle` の 3 つのタスクがあり、それぞれが自身の `#[local]` リソースにアクセスできます。

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/locals.rs}}
```

この例を実行するには:

```console
$ cargo xtask qemu --verbose --example locals
```

```console
{{#include ../../../../ci/expected/lm3s6965/locals.run}}
```

`#[init]` と `#[idle]` のローカルリソースは `'static` ライフタイムを持ちます。これは、どちらのタスクも再入可能ではないため安全です。

### タスクローカルな初期化済みリソース

ローカルリソースは、`#[task(local = [my_var: TYPE = INITIAL_VALUE, ...])]` のようにリソース指定内で直接指定することもできます。これにより、`#[init]` で初期化する必要のないローカルを作成できます。

`#[task(local = [..])]` リソースの型は、スレッド境界をまたがないため、[`Send`] でも [`Sync`] でもある必要はありません。

[`Sync`]: https://doc.rust-lang.org/stable/core/marker/trait.Sync.html

以下の例では、さまざまな使い方とライフタイムを示しています。

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/declared_locals.rs}}
```

アプリケーションを実行することもできますが、この例はライフタイムの性質を示すことだけを目的としているため、出力はありません（アプリケーションをビルドするだけで十分です）。

```console
$ cargo build --target thumbv7m-none-eabi --example declared_locals
```

<!-- {{#include ../../../../ci/expected/lm3s6965/declared_locals.run}} -->

## `#[shared]` リソースと `lock`

`#[shared]` リソースにデータ競合なしでアクセスするにはクリティカルセクションが必要です。そのため、渡される `Context` の `shared` フィールドは、そのタスクからアクセス可能な各共有リソースについて [`Mutex`] トレイトを実装します。このトレイトには [`lock`] という 1 つのメソッドしかなく、そのクロージャ引数をクリティカルセクション内で実行します。

[`Mutex`]: ../../../api/rtic/trait.Mutex.html
[`lock`]: ../../../api/rtic/trait.Mutex.html#method.lock

`lock` API によって作成されるクリティカルセクションは動的優先度に基づいています。つまり、他のタスクがそのクリティカルセクションをプリエンプトできないように、コンテキストの動的優先度を一時的に _ceiling_ 優先度まで引き上げます。この同期プロトコルは [Immediate Ceiling Priority Protocol (ICPP)][icpp] として知られており、RTIC の [Stack Resource Policy (SRP)][srp] ベースのスケジューリングに準拠しています。

[icpp]: https://en.wikipedia.org/wiki/Priority_ceiling_protocol
[srp]: https://en.wikipedia.org/wiki/Stack_Resource_Policy

以下の例では、優先度 1 から 3 の 3 つの割り込みハンドラがあります。低い優先度を持つ 2 つのハンドラは `shared` リソースをめぐって競合し、そのデータにアクセスするにはそのリソースのロック取得に成功する必要があります。`shared` リソースにアクセスしない最も高い優先度のハンドラは、最も低い優先度のハンドラが作成したクリティカルセクションを自由にプリエンプトできます。

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/lock.rs}}
```

```console
$ cargo xtask qemu --verbose --example lock
```

```console
{{#include ../../../../ci/expected/lm3s6965/lock.run}}
```

`#[shared]` リソースの型は [`Send`] でなければなりません。

## Multi-lock

`lock` の拡張として、また右方向へのインデントの増大を抑えるため、ロックはタプルとして取得できます。以下の例はその使用方法を示しています。

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/multilock.rs}}
```

```console
$ cargo xtask qemu --verbose --example multilock
```

```console
{{#include ../../../../ci/expected/lm3s6965/multilock.run}}
```

## 共有 (`&-`) アクセスのみ

デフォルトでは、フレームワークはすべてのタスクがリソースへの排他的な可変アクセス (`&mut-`) を必要とすると仮定しますが、`shared` リストで `&resource_name` 構文を使うことで、タスクがリソースへの共有アクセス (`&-`) だけを必要とすることを指定できます。

リソースへの共有アクセス (`&-`) を指定する利点は、異なる優先度で実行される複数のタスクがそのリソースをめぐって競合している場合でも、リソースにアクセスするためのロックが不要になることです。欠点は、そのタスクがリソースへの共有参照 (`&-`) しか得られず、実行できる操作が制限されることですが、共有参照で十分な場合には、この方法によって必要なロックの数を減らせます。単純なイミュータブルデータに加えて、リソース型自体が適切なロックやアトミック操作によって安全に内部可変性を実装している場合にも、この共有アクセスは有用です。

この RTIC のリリースでは、異なるタスクから _同じ_ リソースに対して排他的アクセス (`&mut-`) と共有アクセス (`&-`) の両方を要求することはできない点に注意してください。そうしようとすると、コンパイルエラーになります。

以下の例では、キー（たとえば暗号鍵）を実行時にロード（または作成）し（`init` から返される）、その後、異なる優先度で実行される 2 つのタスクから、いかなる種類のロックも使わずに使用します。

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/only-shared-access.rs}}
```

```console
$ cargo xtask qemu --verbose --example only-shared-access
```

```console
{{#include ../../../../ci/expected/lm3s6965/only-shared-access.run}}
```

## 共有リソースへのロックフリーアクセス
`#[shared]` リソースが _同じ_ 優先度で動作するタスクからのみアクセスされる場合、それにアクセスするためにクリティカルセクションは _不要_ です。この場合、リソース宣言にフィールドレベル属性 `#[lock_free]` を追加することで、`lock` API の使用を省略できます（以下の例を参照）。

<!-- これは、不要なリソースロック用コードを減らすための単なる便宜機能にすぎないことに注意してください。というのも、
`lock` API を使用したとしても、基盤となる resource-ceiling preemption の仕組みにより、実行時にフレームワークが**クリティカルセクションを生成することはない**
ためです。 -->

Rust の[aliasing]規則に従うため、リソースには複数の不変参照を通じてアクセスするか、単一の可変参照を通じてアクセスするかのいずれかのみが可能です（ただし、同時に両方は不可です）。

[aliasing]: https://doc.rust-lang.org/nomicon/aliasing.html

異なる優先度で動作するタスク間で共有されるリソースに `#[lock_free]` を使用すると、_コンパイル時_ エラーになります -- `lock` API を使用しないと、前述の aliasing 規則に違反するためです。同様に、各優先度について、共有リソースにアクセスできる _ソフトウェア_ タスクは 1 つだけです（`async` タスクは、同じ優先度で動作するほかの _ソフトウェア_ または _ハードウェア_ タスクに実行を譲る可能性があるためです）。しかし、この単一タスク制約の下では、そのリソースは実質的にもはや `shared` ではなく、むしろ `local` であるとみなせます。したがって、`#[lock_free]` を付与した共有リソースを使用すると _コンパイル時_ エラーになります -- 適用できる場合は、代わりに `#[local]` リソースを使用してください。

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/lock-free.rs}}
```

```console
$ cargo xtask qemu --verbose --example lock-free
```

```console
{{#include ../../../../ci/expected/lm3s6965/lock-free.run}}
```
# 概要

[はじめに](./preface.md)

---

- [新しいプロジェクトを始める](./starting_a_project.md)
- [例で学ぶ RTIC](./by-example.md)
  - [`app`](./by-example/app.md)
  - [ハードウェアタスク](./by-example/hardware_tasks.md)
  - [ソフトウェアタスクと `spawn`](./by-example/software_tasks.md)
  - [リソース](./by-example/resources.md)
  - [init タスク](./by-example/app_init.md)
  - [idle タスク](./by-example/app_idle.md)
  - [チャネルベースの通信](./by-example/channel.md)
  - [Monotonics を使った遅延とタイムアウト](./by-example/delay.md)
  - [最小のアプリ](./by-example/app_minimal.md)
  - [ヒントとコツ](./by-example/tips/index.md)
    - [リソースのデストラクチャリング](./by-example/tips/destructureing.md)
    - [メッセージ受け渡し時のコピーを避ける](./by-example/tips/indirection.md)
    - [`'static` の強力な機能](./by-example/tips/static_lifetimes.md)
    - [生成されたコードを調べる](./by-example/tips/view_code.md)
- [Monotonics とタイマーキュー](./monotonic_impl.md)
- [RTIC と他の世界](./rtic_vs.md)
- [RTIC と Embassy](./rtic_and_embassy.md)
- [素晴らしい RTIC の例](./awesome_rtic.md)

---

- [v1.0.x から v2.0.0 への移行](./migration_v1_v2.md)
  - [`rtic-monotonics` への移行](./migration_v1_v2/monotonics.md)
  - [ソフトウェアタスクは теперь `async` でなければなりません](./migration_v1_v2/async_tasks.md)
  - [`rtic-sync` の使い方と理解](./migration_v1_v2/rtic-sync.md)
  - [移行のコード例](./migration_v1_v2/complete_example.md)

---

- [内部の仕組み](./internals.md)
  - [対象アーキテクチャ](./internals/targets.md)
  <!--- [割り込み設定](./internals/interrupt-configuration.md)-->
  <!--- [非再入性](./internals/non-reentrancy.md)-->
  <!--- [アクセス制御](./internals/access.md)-->
  <!--- [遅延リソース](./internals/late-resources.md)-->
  <!--- [クリティカルセクション](./internals/critical-sections.md)-->
  <!--- [天井解析](./internals/ceilings.md)-->
  <!--- [ソフトウェアタスク](./internals/tasks.md)-->
  <!--- [タイマーキュー](./internals/timer-queue.md)-->

  <!-- - [タスクの定義](./by-example/app_task.md) -->
  <!-- - [ソフトウェアタスクと `spawn`](./by-example/software_tasks.md)
    - [メッセージ受け渡しと `capacity`](./by-example/message_passing.md)
    - [タスクの優先度](./by-example/app_priorities.md)
    - [Monotonic と `spawn_{at/after}`](./by-example/monotonic.md) 
  -->
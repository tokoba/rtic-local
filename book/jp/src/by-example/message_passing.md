# メッセージパッシングとcapacity

ソフトウェアタスクはメッセージパッシングをサポートしています。つまり、ソフトウェアタスクは引数付きでspawnできます: `foo::spawn(1)` は、引数 `1` を指定してタスク `foo` を実行します。

capacity は、そのタスクのspawnキューのサイズを設定します。指定しない場合、capacity のデフォルト値は 1 です。

以下の例では、タスク `foo` の capacity は `3` であり、`foo` の保留中のspawnを同時に3つまで許可します。この capacity を超えると `Error` になります。

タスクに渡せる引数の数に制限はありません:

``` rust,noplayground
{{#include ../../../../examples/message_passing.rs}}
```

``` console
$ cargo xtask qemu --verbose --example message_passing
{{#include ../../../../ci/expected/message_passing.run}}
```
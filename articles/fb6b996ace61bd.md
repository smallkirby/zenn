---
title: "Zig探訪 - シンタックス編"
emoji: "💓"
type: "tech"
topics: ["zig"]
published: true
---

## イントロ

師も走ると言われる師走も走り去り、2025年となりました。走り去った師の背中を見ると、そこには Zig の3文字が。さあ、やってまいりました。第3回 **Zig探訪** のお時間です。今回のテーマは Zig のシンタックスや基本的な機能のうち、自分が好きな点や微妙だと思っている点です。

前回までの Zig探訪 のバックログはこちら:

- [Zig探訪 - エコシステム編](https://zenn.dev/smallkirby/articles/10dbe02303a201)
- [Zig探訪 - comptime編](https://zenn.dev/smallkirby/articles/54882aee98e2c9)

あと、 [Writing Hypervisor in Zig](https://hv.smallkirby.com/) という Zig を使ってハイパーバイザをフルスクラッチで書くブログを書いたので、全員読んでください。

https://x.com/smallkirby/status/1858091447393652860

## `for` や `while` に `else` 節を付けられる

Zig の `for` は基本的に配列やスライス[^slice]の要素をイテレートする目的でのみ使います[^for-iterate]。以下の例の `0..` は `for` ループでのみ登場する特殊構文であり、ループのたびにインクリメントされるカウンタとして利用できます (オプショナルなので無くても良い):

```zig
const array = [_]u8{ 1, 2, 3 };
for (array, 0..) |element, ix| {
    std.debug.print("{d}: {d}\n", .{ ix, element });
}

// 0: 1
// 1: 2
// 2: 3
```

そんな `for` ループは、`break` を使って値を返すことができます。この場合、`break` せずにループが終了した場合に評価される `else` 節を指定します。以下は配列から最初の4の倍数を探すコードです:

```zig
const array = [_]u8{ 1, 2, 3 };
const result = for (array) |element| {
    if (element % 4 == 0) break element;
} else 0;
std.debug.print("result: {}\n", .{result});

// result: 0
```

同様に `while` ループも `else` 節を使うことができます:

```zig
const array = [_]u8{ 1, 2, 3 };
var ix: usize = 0;
const result = while (ix < array.len) : (ix += 1) {
    if (array[ix] % 4 == 0) break array[ix];
} else 0;
std.debug.print("result: {}\n", .{result});

// result 0
```

ループが終わった後にいちいち例外処理を別途書く必要がなくなり、`result` を `const` にしておけるのがありがたいですね。

## `Optional` 型と `if` / `while`

Zig には Rust の `Option` や Kotlin の Nullable 型に相当する Optional 型があります。これにより、変数がヌルポインタ[^nullptr]を持ったり未初期化[^undefined]状態のまま意図せず放置されることがなくなります:

```zig
const maybeInt: ?u32 = null;
std.debug.print("{}\n", .{maybeInt orelse 0});

// 0
```

`orelse` は `maybeInt` が `null` の場合に評価されます。Kotlin のエルビス演算子 `?:` と同じです。Rust の `unrap()` に相当する `.?` 演算子があり、Kotlin の `?.` に相当する演算子はありません。

そんな Optional 型ですが、`while` や `if` は条件として Optional 型を受け付けます。`null` の場合には条件式が `false` のように扱われます。`null` 出ない場合には、中身の値を捕捉することができます:

```zig
const ListType = std.DoublyLinkedList(u32); // 双方向リスト型
const ListNode = ListType.Node;             // 双方向リストの中身の型
var list = ListType{};

var cur: ?*ListNode = list.first;
while (cur) |node| : (cur = node.next) {
    if (node.data == 3) break;
}
```

`: (cur = node.next)` はループの度に実行される式ですが、個人的にはこの中で捕捉した値 `node` を利用できるのがポイント高いです。
また、`for` / `while` が Optional を受け付けるとは言っても、`if(0)` のようにその他の型を条件式として指定することはできません。条件として受け付けるのは `bool` か Optionlal のみであり、且つ `bool` と整数型は暗黙的な変換を許しません。素晴らしい。

## いろいろなブロックが値を返せる

`Labeled XYZ` という名前の機能でブロックに名前を付け、名前をつけたブロックからは値を返すことができます。Rust みたいですね。`XYZ` に入るのは `if` や `while` や `for` の他、`orelse` や `catch` も含まれます:

```zig
const result: u32 = if (a % 2 == 0) blk: {
    const s = some();
    break :blk s *| a;
} else blk: {
    const s = bar();
    break :blk s +| a;
};
```

ある関数が処理に失敗した場合、特定の操作をしてもう一度リトライするみたいなことも簡単に書けます:

```zig
const result = some() catch blk: {
    bar(); // 何らかの操作をして
    break :blk try some(); // リトライ
};
```

## ポインタが指す構造体に透過的にアクセスできる

これは良いところでもあり、readability の観点では微妙なところでもあります。Zig では、ポインタが指す変数に対する代入は `pointer.* = value` のように `.*` 演算子を使います。しかし、ポインタが構造体を指す場合には `variable.field` のようにポインタではない場合と同様の記法でアクセスすることができます:

```zig
var s = S{ .a = 42 };
const S = struct { a: u32 };
const ptr = &s;
ptr.a = 99;
```

コードを書く上では非常に便利です。いちいち `.*` とかいうきしょい演算子を使いたくないですからね。しかし、自分が Zig の一番好きなところは「とにかく読みやすい」ところです。C++ の対極ですね。コードを読めばその型や意図がひと目ででわかるのが良いところです。その点、この記法はポインタと非ポインタの区別を曖昧にするものなので Zig Zen には反するかもしれません。

Zig Zen の話が出たので少しだけ触れておきましょう。Zig のポリシーは `zig zen`[^zig-zen] コマンドで確認できます。朝起きる時と寝る前には必ず目を通しておきましょう。自分が好きな部分は以下の7つです。順不同であり、空行は筆者による分類のために挿入されています:

```txt
 * Communicate intent precisely.
 * Favor reading code over writing code.

 * Focus on code rather than style.
 * Only one obvious way to do things.

 * Runtime crashes are better than bugs.
 * Compile errors are better than runtime crashes.
```

最初の2つは先に触れた「読みやすさ」についてです。演算子が持つ意味が(その状況において)ただ一つであり、型に継承関係もなく、操作に対する結果が呼び出し側から明確にわかるのが Zig です。
続く2つは Go みたいですね。Ruby の反対とも言えるかもしれません。Zig の唯一のフォーマッタはいかなるコンフィグも受け付けず、コードをただ1つの方法で整形します。書いた時代[^generation]や書いた人・組織に依らず、コードは同じ見た目をします。非常に読みやすいです。あることをするための書き方も(およそ)1つに定まるため、その形から意図を読み取ることも容易です。
最後の項目は [Zig探訪 - comptime編](https://zenn.dev/smallkirby/articles/54882aee98e2c9) でも取り上げたように、Zig の強力なコンパイル時評価による恩恵です。気になる人はバックログを読んでみてください。

## 返り値の discard が明示的 / 利用していない変数が違法

Zig では、関数の返り値を変数に代入せず捨てることが違法です。C のようにコンパイラのオプションで指定する必要はなく、強制的にコンパイル時にエラーが発生します:

```zig
fn some() u64 {
    return 1;
}

some();

// src/main.zig:9:9: error: value of type 'u64' ignored
//     some();
//     ~~~~^~
```

同様に、利用していない変数が存在することも違法です。`[[nodiscard]]` のようなものを指定する必要はありません。Zig の世界では誰一人捨てられることはありません(後述しますが、嘘です):

```zig
const a = 32;

// src/main.zig:5:11: error: unused local constant
//     const a = 32;
//           ^
```

どうしても値を捨てたい場合には、明示的に `_` に代入することでコンパイラに怒られなくなります:

```zig
_ = some();
```

まぁ、そういう時は関数が返り値を返さないようにすればいいのではないでしょうか。一回捨てるのに慣れてしまうと、捨てることに対する躊躇が無くなってしまいます。そんなあなたは路上喫煙して吸い殻をポイ捨てしているのと同じです。読む側からしても、値を意図的に捨てているのかどうかが区別しやすいのはありがたいですね。

## シンボルがどこから来たかが分かりやすい

かなり得点が高いです。C/C++ を読んでいて一番イヤなのが、関数やらクラスやらがどこから来たかが分からないことです。おまけに宣言と定義が別れてるし。想像するのも嫌ですが、以下のコードがあるとします:

```c
#include <a.hpp>
#include <b.hpp>
#include <c.hpp>

void some() {
  SomeThing some;
  some.Party(SuperGlobal);
}
```

エディタを使って読んでいる時はまぁ良いです。嫌だけど。そうではない時は、`a.hpp` / `b.hpp` / `c.hpp` をブルートフォースして `SomeThing()` を探すという虚無が発生します。また、`SuperGlobal` という謎グローバル変数を探すのにも同様に虚無が発生します。これら3ファイルを探すだけでは当然こと足りず、それらがインクルードする先も探す必要があります。こんなに虚無なことがあるでしょうか。

Zig では、ファイルが構造体として扱われます。あるファイルが公開する型・変数・関数は、そのファイルの構造体フィールドとしてアクセスします:

```zig
//! File: a.zig
const b = @import("b.zig");
test {
    b.c.some();
}

//! File: b.zig
pub const c = @import("c.zig");

//! File: c.zig
pub fn some() void {}
```

どこから来たかが一目瞭然です。`a.zig` の中に勝手に `c.zig` 内のシンボルが混入することはなく、`some()` 関数は `b.zig` 経由で `c.zig` から来たことが明らかです。

もちろん、階層が深くなりアクセスが面倒な場合にはアンブレラ的に上位のファイルから下位のファイルを公開することもできます (例: [stdlib の `DoublyLinkedList`](https://github.com/ziglang/zig/blob/e6879e99e26fd11f659be3deec2b5de409b46547/lib/std/std.zig#L19) )。この場合でも、あるシンボルがどこから来たのかは依然として明確です:

```zig
pub const DoublyLinkedList = @import("linked_list.zig").DoublyLinkedList;

// 利用する時は
const std = @import("std");
const list = std.DoublyLinkedList(u32){};
```

一応、`usingnamespace` という黒魔術のような名前の機能を使うことで、ある構造体などの宣言を全て現在の名前空間に展開することはできます。しかし、この場合でも「展開する先の名前空間から、その宣言を利用すること」はできないです。ややこしいですが、以下で `b.zig` の中で直接 `usingnamespace` を使うことはできません。良かった...:

```zig
//! File: a.zig
const b = @import("b.zig");
b.some(); // OK: これはできる

//! File: b.zig
pub usingnamespace @import("c.zig");
b.some(); // NG: これはできない

const S = struct { usingnamespace @import("c.zig"); };
S.some(); // OK: これはできる

//! File: c.zig
pub fn some() void {}
```

## 同一ファイルでは構造体の任意のフィールドにアクセス可能

Zig の構造体は、定数・変数・関数を持つことができます (定数には、型も含まれます):

```zig
const S = struct {
  // 定数
  const a = 4096;
  // 変数
  field: u64,
  // 関数
  fn some(self: *@This()) void { ... }
};
```

このうち、定数と関数には visibility を指定することができます:

```zig
const S = struct {
  pub const public_constant = 1;
  const private_constant = 2;

  pub fn public_some(self: *@This()) void { ... }
  fn private_some(self: *@This()) void { ... }
};
```

しかしながら、これらのアクセス指定子は `@import()` を介してインポートされた場合のみ有効になります。言い換えると、構造体を定義したファイル内では任意のフィールドにアクセスすることが可能です:

```zig
//! File: a.zig
pub const S = struct { ... };
test {
    const s = S{ .field = 3 };
    std.debug.print("{d}\n", .{ s.a }); // 合法
    s.private_some();                   // 合法
}

//! File: b.zig
const a = @import("a.zig");
test {
    const s = a.S{ .field = 3 };
    std.debug.print("{d}\n", .{ s.a }); // 違法
    s.private_some();                   // 違法
}
```

変数に関しては、そもそも visibility を指定することができず、強制的にパブリックになります。これらは賛否が分かれるところだと思いますし、自分は基本プライベートにしたい派です。意図としては、構造体を定義したファイル内で構造体を使ったヘルパー関数を実装しやすくしたり、どうせ設計するうちに本来公開すべきではないものを実装の都合上公開するようにする場合が多いだろうから、最初から全部公開してしまえということなのかもしれません[^pub]。

強いて良い点を上げるならば、テストを書く際に内部関数や定数を直接使うことができるため、ホワイトボックス的にテストを書きやすいという点でしょうか。余計なゲッタを作る必要もなくなります。
ただ、やはり構造体の外からアクセスしてほしくないフィールドというものは存在するもので、自分はネーミング規則でカバーしています:

https://github.com/smallkirby/zig-style

`_` から始まるフィールドは外部からアクセスしては駄目で、`__` で始まるフィールドはアクセスすること自体禁止ということにしています。アクセス禁止のフィールドは、コンパイル時にのみアクセスすることを目的にしています:

```zig
const S = packed struct {
    a: u64,
    b: u32,
    __marker: void,
    c: u32,

    const first_block_size = @offsetOf(@This(), "__marker");
};
```

## Destructuring Assignment みたいなものがある

JS みたいに、構造体のフィールドを取り出して別々の変数に代入することができます。[比較的最近追加](https://github.com/ziglang/zig/pull/17156)されたようです。ただし、無名構造体でしか利用不可なのが惜しいところ。というか、もうあんまり機能は増やさなくていいよ:

```zig
fn hoge() struct { u64, u64 } {
    return .{ 1, 2 };
}
const a, const b = hoge();
```

こいつもそうですが、「Zigの知らなかった機能」というスクラップに知らなかった機能を随時書いています。Zig探訪は、基本的にこのようなスクラップや個人メモから生成しています:

https://zenn.dev/smallkirby/scraps/291bf737294862

https://zenn.dev/smallkirby/scraps/93e608c7af2f75

## アウトロ

そういえば、最近 Zig の GitHub スポンサーを始めました。よいお年を。

[^slice]: [配列](https://ziglang.org/documentation/master/#Arrays) の長さはコンパイル時に決定している必要があり、ランタイムまで長さがわからない場合は [スライス](https://ziglang.org/documentation/master/#Slices) を使います。スライスは先頭要素へのポインタと要素数を持つ型です。
[^for-iterate]: [Why are range variables always usize? - Ziggit](https://ziggit.dev/t/why-are-range-variables-always-usize/3026/2)
[^nullptr]: Zig では暗黙的にヌルポインタを作成することができません。敢えてヌルポインタを作成したい場合は `*allowzero u8` のような型を明示的に使う必要がありますが、まぁ使うことはないでしょう。
[^undefined]: Zig では変数に `undefined` を代入することで未初期化の状態にしておくことができます。ただし、これは変数が未初期化であるかどうかを識別する目的のものではなく、未初期化かどうかを判定する方法もありません。そのような場合は Optional 型を使いましょう。なお、`undefined` で初期化された変数は `Debug` ビルドの場合 `0xAA` で埋められます。それ以外の場合には完全に未定義です。
[^generation]: まぁ、Zig にはそもそも「古いコード」が誕生するほどの歴史がないんですけどね。
[^zig-zen]: "Zig Zen" で検索すると、おそらく Zen 言語に対する [Zigソフトウェア財団からの声明](https://ziglang.org/news/statement-regarding-zen-programming-language/) がトップヒットします。ひと悶着あったようです。
[^pub]: [Reasoning why private method of struct is accessible from within the same file - Ziggit](https://ziggit.dev/t/reasoning-why-private-method-of-struct-is-accessible-from-within-the-same-file/7465)

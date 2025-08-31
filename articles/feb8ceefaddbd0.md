---
title: "Zig v0.15.1 リリース"
emoji: "⚡"
type: "tech"
topics: ["zig"]
published: true
---

## イントロ

暑さも寒さも彼岸までとはよく言いますが、そんなみんな大好き彼岸がいよいよ近づいてきました。もしも彼岸で涼しくならなかったら、気象庁を訴えようと思います。最高裁までは持ち込む覚悟です。さて、みんな大好きで思い出しましたが、**Zig 0.15.1** が2025年08日19日にリリースされました。何故かは分かりませんが、[0.15.0 のリリースはキャンセルされた](https://ziggit.dev/t/zig-0-15-1-released/11583?u=smallkirby)らしいです。

Zig はまだ 1.0 がリリースされていないため、マイナーバージョンアップデートの本バージョンにおいても多数の破壊的変更が含まれています。本記事では、0.15.1 における変更点の内気になったものや、自分のプロジェクトを 0.14.1 からアップデートするに当たり苦労した点を中心にピックアップしていきます。詳細な変更点については[リリースノート](https://ziglang.org/download/0.15.1/release-notes.html)を参照してください。

## `usingnamespace` の削除

C/C++ と比較して Zig は「メモ帳で見たときに読みやすい言語」だと思っています。これは、Zig の zen の一つでもある "Favor reading code over writing code." にも現れています。その例として、ある定数・変数がどこから来たのかがひと目で分かるようになっています。例えば以下のような C++ のコードがあった時、シンボル `do_something` はどこから来たものなのかがひと目ではわかりません:

```cpp
#include <a.hpp>
#include <b.hpp>
#include "saikyo.hpp"
void main() {
    do_something();
}
```

インクルードされているファイルをすべて確認する必要があり、かつヘッダファイルがどこに存在しているのかも定かではありません (同一ディレクトリにあるかもしれないし、`include` ディレクトリにあるかもしれないし、独自のインクルードパスにあるかもしれない)。

一方、Zig では定義がどこから来たのかが明らかです:

```zig
const a = @import("a.zig");
const b = a.b;
const do_something = b.do_something;
fn main() void {
    do_something();
}
```

Zig ではファイルが暗黙的に構造体として扱われ、ファイルで定義されたシンボルは構造体のメンバとして参照することができます。これにより、あるシンボルの定義は構造体メンバの参照チェインを辿っていけば見つかることになり、人間にとっても機械にとっても追跡がとても容易です。

しかし、0.14.1 までは例外として [usingnamespace](https://ziglang.org/documentation/0.14.1/#usingnamespace) がありました。見るからに黒魔術的なこいつは、指定した構造体に属するメンバをすべて現在のスコープに展開します:

```zig
const S = struct {
    usingnamespace @import("std");
};
```

これは Zig が持つシンボルの tracability を損なうものであったため、0.15.1 で削除されました。

自分は自作OSで以下のようにアーキテクチャ固有の API をexport するためのアンブレラコードとして使っていました:

```zig
// arch.zig
pub usingnamespace switch (builtin.target.cpu.arch) {
    .x86_64 => @import("arch/x86/arch.zig"),
    else => unreachable,
};
// 使う時
const arch = @import("arch.zig");
```

0.15.1 ではこのコードは以下のように1段余計な定数定義を噛ませる必要があります:

```zig
// arch.zig
pub const impl = switch (builtin.target.cpu.arch) {
    .x86_64 => @import("arch/x86/arch.zig"),
    else => unreachable,
};
// 使う時
const arch = @import("arch.zig").impl;
```

その他にも、リリースノートでは `usingnamespace` の利用についていくつかは合法的な使い方があったことを認めた上で、各ユースケースにおける[マイグレーションガイド](https://ziglang.org/download/0.15.1/release-notes.html#usingnamespace-Removed)を記載しています。`usingnamespace` をどうやって取り除こうか悩んでいる人は見てください。

## `std.os.uefi` の Ziggify

リリースノートには一切の記載がありませんが、`std.os.uefi` 以下に大量の破壊的変更が入っています。これらの多くは、これまで [UEFI Specification](https://uefi.org/specs/UEFI/2.9_A/) で定義される C API をそのまま使っていた API を、[より Zig っぽくするための変更](https://github.com/ziglang/zig/pull/23214)です。例えば、従来の EFI API は返り値として `Status` を返し、実際の返り値は出力引数として受け取った変数に出力するようになっていました

```zig
// https://github.com/ziglang/zig/blob/6d1f0eca773e688c802e441589495b7bde2f9e3f/lib/std/os/uefi/protocol/file.zig#L55-L57
pub fn read(self: *const File, buffer_size: *usize, buffer: [*]u8) Status {
    return self._read(self, buffer_size, buffer);
}
```

これが 0.15.1 では `Status` の変わりに [Error Union](https://ziglang.org/documentation/master/#Error-Union-Type) を返すようになり、空いた返り値として出力を返すようになりました:

```zig
// https://github.com/ziglang/zig/blob/cbc3c0dc594341faddb8841fb466254f2f6f3b35/lib/std/os/uefi/protocol/file.zig#L130-L140
pub fn read(self: *File, buffer: []u8) ReadError!usize {
    var size: usize = buffer.len;
    switch (self._read(self, &size, buffer.ptr)) {
        ...
    }
}
```

後者のほうが非常に Zig らしいので良いですね。その他にも、これまで単純な整数型であったものを `enum` として定義したり、複雑な入力・出力変数の型を `struct` としてまとめたりなどの変更が入っています。UEFI API の大部分が影響を受けています。個人的には 0.15.1 へのアップデートに当たり最も労力が必要だったのがこの変更でした。なお、0.14.0 リリースで UEFI platform は Tier サポートから外れたためなのか、本変更に対するリリースノートでの記載はありません。

## 整数型の `enum` 化

先程の UEFI API の例でも出てきましたが、いくつかの std API においてこれまで `usize` や `u64` のような整数型を取ってきたものが、新たに専用の `enum` 型を取るようになりました。例として、[`std.mem.Allocator.alignedAlloc()`](https://github.com/ziglang/zig/blame/cbc3c0dc594341faddb8841fb466254f2f6f3b35/lib/std/mem/Allocator.zig#L238-L246) の引数 `alignment` の型が `u29` から `Alignment` に変わりました:

```zig
pub fn alignedAlloc(
    self: Allocator,
    comptime T: type,
    /// null means naturally aligned
    comptime alignment: ?Alignment,
    n: usize,
) Error![]align(if (alignment) |a| a.toByteUnits() else @alignOf(T)) T
```

[`Alignment`](https://github.com/ziglang/zig/blob/0.15.x/lib/std/mem.zig#L22C1-L30C7) は non-exhaustive enum として表現されています:

```zig
pub const Alignment = enum(math.Log2Int(usize)) {
    @"1" = 0,
    @"2" = 1,
    @"4" = 2,
    @"8" = 3,
    @"16" = 4,
    @"32" = 5,
    @"64" = 6,
    _,
    ...
};
```

しかし、何故か `64` までしか定義されていません。`alignedAlloc()` は実装上は `4096` 未満のアラインは受け付けるようになっているため、なぜ `64` で止めたのかは謎です。なお、non-exhaustive であるため以下のように `128` をアラインとして渡すこと自体は可能です:

```zig
allocator.alignedAlloc(u8, @enumFromInt(128), 100);
```

## zig fmt によるキャスト順の変化

これまで `@alignCast()` と `@ptrCast()` を同時に使ったときに `zig fmt` が強制してくるキャストの順番は以下のようになっていました:

```zig
@alignCast(@ptrCast(pointer))
```

0.15.1 では `@alignCast` が先になります:

```zig
@ptrCast(@alignCast(pointer))
```

どうでも良い変更な上、フォーマッタが勝手にやってくれるので、どうでも良いですね。

## `switch` で `else`/`_` prong の併用可能

Zig の `enum` では、あり得る値をすべて列挙しなくても良い [non-exhaustive enum](https://ziglang.org/documentation/master/#Non-exhaustive-enum) というものがあります:

```zig
pub const Color = enum(u8) {
    red = 0,
    blue = 1,
    green = 2,
    _, // <=== non-exhaustive
};
```

これにより、任意の `u8` 型を `@enumFromInt()` で `Color` に変換することができます。Ehaustive enum の場合には `@as(Color, @enumFromInt(3))` のように対応するタグが無い整数値を `Color` に変換しようとするとエラーになります。
さて、`enum` を `switch` で扱う場合には、`else` と `_` を使うことができます。`else` は「明示的に列挙したケース以外すべて」を表し、`_` は「タグ名のついていない値すべて」をキャッチします:

```zig
switch (color) {
    .red, .blue => {},
    else => {}, // <== .green とタグ名のない値すべて
}
switch (color) {
    .red, .blue => {},
    _ => {}, // <== タグ名のない値すべて。.green がキャッチされないのでコンパイルエラー
}
```

これまでは、`else` と `_` の両方を一つの `switch` に使うことはできませんでした。0.15.1 からはこれが可能になります。これによって、「タグ名のあるその他すべての値」と「タグ名の無いすべての値」を別々にハンドリングすることが可能になりました:

```zig
switch (color) {
    .red, .blue => {},
    else => {}, // <== green
    _ => {}, // タグ名のない値すべて
}
```

以前からほしいと思っていた機能だったため、素直に嬉しい。

## inline assembly における clobber 構文

Inline assembly における clobber は、これまで C/C++ と同様に文字列として書くことになっていました:

```zig
asm volatile (
    \\pushfq
    \\pop %[rflags]
    : [rflags] "=r" (-> u64),
    :
    : "memory", "cc", "rflags"
);
```

0.15.1 からは無名構造体として clobber を記述することになります:

```zig
asm volatile (
    \\pushfq
    \\pop %[rflags]
    : [rflags] "=r" (-> u64),
    :
    : .{ .memory = true, .cc = true, .rflags = true }
);
```

型を持つことになるため、うっかり存在しない clobber を指定することがなくなります。この変更は `zig fmt` が勝手にやってくれるため、非常に楽です。助かる。

## Debug ビルドにおけるネイティブバックエンド

0.14.1 の時点で available になっていましたが、[0.15.1 からは x64 環境における Debug ビルドでデフォルトでネイティブバックエンドが使われる](https://ziglang.org/download/0.15.1/release-notes.html#x86-Backend)ことになります。以前からずっと LLVM から脱却したいと書いてあったため、来たるべきして来た変更です。個人的には、Debug と Release でバックエンドが全く違うものが使われるのはどうなんだろうなぁと思わなくもないですが、ネイティブバックエンドのほうがビルド時間が5倍近く早いらしいため、使えるならば使いたいです。
しかし、自作OSのような低レベルなプロジェクトではネイティブに切り替えてすぐにコンパイルが通るはずはなく、[自分のプロジェクト](https://github.com/smallkirby/norn)では100近いエラーが出ています。とりわけインラインアセンブリにおけるサポートされていない構文エラーが多いですね。
ひとまず LLVM バックエンドを利用し続ける場合には、`addExecutable()` 等に渡すオプションに `.use_llvm` を指定すればOKです:

```zig
    const surtr = b.addExecutable(.{
        .name = "BOOTX64.EFI",
        .root_module = b.createModule(.{
            ...
        }),
        .linkage = .static,
        .use_llvm = false,
    });
```

クリティカルな問題が見つからない限りは、早いところ Debug / Release ともにネイティブバックエンドに移行したいと思っています。どうせそのうち Release ビルドのデフォルトもネイティブバックエンドに置き換わるでしょうし...。

## フォーマット指定子の追加

新たに以下のフォーマット指定子が追加されました:

- `{t}`: `@tagName()` または `@errorName()`
- `{b64}` : base64 として出力。誰が使うねん。

これにより、今まで便利に使えていた `{?}` 指定子 + Error Type が使えなくなりました:

```zig
std.log.debug("{?}", .{err});
```

0.15.1 では以下のように書きます:

```zig
std.log.debug("{t}", .{err});
std.log.debug("{s}", .{@errorName(err)});
```

## `std.ArrayList` の unmanaged 化

Zig の良いところとして、暗黙的なメモリ確保が存在しない点が挙げられます。std ライブラリの (恐らく) すべてのAPIは、メモリ確保が必要な場合には引数として `std.mem.Allocator` を受取り、それを使ってメモリを確保します。
一方で、構造体を作成する場合にはこれまで2通りの方法がありました。1つ目が、メモリ確保をする構造体メソッドはセオリー通り必ずアロケータを引数に取るようにする方法 (**Unmanaged**)。2つ目が、構造体の初期化において1度だけアロケータを引数にとり、そのアロケータをメンバ変数として保持しておいて以降はそれを使うという方法 (**Managed**) です。
これまで、`std.ArrayList` は2つ目の [Managed な方法を取っていました](https://github.com/ziglang/zig/blob/6d1f0eca773e688c802e441589495b7bde2f9e3f/lib/std/array_list.zig#L53-L59):

```zig
pub fn init(allocator: Allocator) Self {
    return Self{
        .items = &[_]T{},
        .capacity = 0,
        .allocator = allocator,
    };
}
```

0.15.1 からは、1つ目の Unmanaged がデフォルトになりました。ついでに `std` からの export 方法も変わり、`std.ArrayList` の変わりに `std.array_list.Aligned` のようにしてアクセスするようになりました。一応従来の Managed 型である `std.array_list.Managed` も残っていますが、すでに deprecate であり将来の削除が確定しているので使わない方が良いです。
従来の Managed なコードは以下のように書き換えられます:

```zig
const list: std.array_list.Aligned(u8, null) = .empty;
try list.append(allocator, some);
list.deinit(allocator);
```

なお、`Aligned` ではない型は消えたため、第2引数としてアラインを渡す必要があります。特に木にしない場合は `null` を渡すことで `@alignOf(@TypeOf(arg))` がアラインとして勝手に使われます。`.empty` は従来の `.init()` 関数に相当するもので、`Aligned` 構造体に定数として定義されています。ここでは decl literals を使って書いています。
Unmanaged がデフォルトの (というか deprecate されることを考えると唯一の) 方法になったことに関しては、思うところが無いわけでもないです。[リリースノート](https://ziglang.org/download/0.15.1/release-notes.html#ArrayList-make-unmanaged-the-default)にはその良し悪しが書いてあり、結局は 「Managed の良いところが上回らなかった」と書いてあります。しかし、アロケータを毎回渡すということは誤ったアロケータを渡してしまう可能性も発生するということであり、このデメリットはなかなかでかいんじゃないかなぁ...と思っています。たかだか 16bytes の削減のためにアロケータを毎回渡す面倒さとリスクを取るのかぁという気持ちもありますが、慣れたらそっちのほうが良いのかも知れないし、そうじゃないのかも知れない。

## `DoublyLinkedList` の instrusive 化

[従来の `std.DoublyLinkedList`](https://github.com/ziglang/zig/blob/6d1f0eca773e688c802e441589495b7bde2f9e3f/lib/std/linked_list.zig#L184-L197) は、メンバとして `first` / `last` ポインタを持っており、それが `Node` 型ポインタを持っていました。`Node` は目的のデータ型とリストを構成するための `prev`/`next` ポインタを持つヘッドです:

```zig
pub fn DoublyLinkedList(comptime T: type) type {
    return struct {
        const Self = @This();

        /// Node inside the linked list wrapping the actual data.
        pub const Node = struct {
            prev: ?*Node = null,
            next: ?*Node = null,
            data: T,
        };

        first: ?*Node = null,
        last: ?*Node = null,
        len: usize = 0,
        ...
    };
}
```

0.15.1 では、`Node` が目的のデータ型を持たず、単に `prev` /`next` だけを持つように[なりました](https://github.com/ziglang/zig/blob/cbc3c0dc594341faddb8841fb466254f2f6f3b35/lib/std/DoublyLinkedList.zig#L18-L27):

```zig
first: ?*Node = null,
last: ?*Node = null,

pub const Node = struct {
    prev: ?*Node = null,
    next: ?*Node = null,
};
```

つまり、このリストヘッド `Node` を目的のデータ型に直接埋め込む intrusive 型になったということです:

```zig
const SomeData = struct {
    head: DoublyLinkedList.Node,
    data: u64,
};
```

Linux の `list_head` 型とおおよそ同じ感じです。自分も intrusive な連結リストが欲しいなぁと思い[自前で実装していた](https://github.com/smallkirby/norn/blob/d8444fff60660fb5dc28586e21bc2800b9bb4907/norn/typing.zig#L43)ので、この変更自体は特に違和感はないです。
とはいっても、`DoublyLinkedList` の API の返り値はデータ型ではなく `Node` なので、そこからデータを取得するのにいちいち `@fieldParentPtr()` を使うのは少し面倒ですね。また、`len` フィールドが削除され、代わりに `len()` メソッドが追加されました。この `len()` はリストを最初から最後まで走査するため、ノード数のカウントに O(N) の時間がかかります。8bytes 削減したかったんでしょうが、どうしてこうなった...。0.15.1 では全体的に std 内の構造体のダイエットが行われた代わりに、ユーザがその構造体の内部構造を意識的に管理させるようにする変更が多かった印象ですね。

## callconv の小さい修正

Windows の calling convention を表す `.Win64`  が `.winapi` もしくは `.{.x86_64_win = .{}}` に変更されました。`.winapi` は便利定数であり、[ターゲットアーキテクチャによって自動的に適切なタグが選択されます](https://github.com/ziglang/zig/blob/e2fdaea0b351860da5f560cbe0ec8f056b8047fd/lib/std/builtin.zig#L174-L180):

```zig
pub const winapi: CallingConvention = switch (builtin.target.cpu.arch) {
    .x86_64 => .{ .x86_64_win = .{} },
    .x86 => .{ .x86_stdcall = .{} },
    .aarch64 => .{ .aarch64_aapcs_win = .{} },
    .thumb => .{ .arm_aapcs_vfp = .{} },
    else => unreachable,
};
```

## Writergate

**既存の `std.io.Reader/Writer` がすべて廃止されました**。実に面白い。
代わりに `std.Io.Reader/Writer` という新しい型が導入されています。大文字の `I` です。一応古い reader/writer からの移行用として、古い reader/writer には `adaptToNewApi()` というメソッドが追加され、これが新しい reader/writer を生成してくれるようになっています。まぁでもいつか廃止されるでしょう。多分。

正直自分は `std.io.Reader/Writer` はそこまで使ったことがなかったので、詳細な変更点については分かっていません。自分のプロジェクトにあったいくつかの Reader/Writer を新APIに移行した感想としては、新 API 用の vtable のメソッド名がちょっと分かりにくいかなぁというくらいです。例えば一番使うであろう Writer メソッドが [`drain()` です](https://github.com/ziglang/zig/blob/cbc3c0dc594341faddb8841fb466254f2f6f3b35/lib/std/Io/Writer.zig#L18-L44):

```zig
drain: *const fn (w: *Writer, data: []const []const u8, splat: usize) Error!usize,
```

doc comment を見ると、`Writer` の内部バッファ `buffer` にデータが収まらなくなったときにフラッシュする関数であることは分かるのですが、それにしてももうちょっといい名前があったのではという気がしないでもないです。自分は unbuffered Writer しか実装したことがないため、`data` が二重配列になっている理由や `splat` については全く理解していません。

ところで、これに伴い `std.elf` の各 API は [`std.fs.File.Reader` を引数に取る](https://github.com/ziglang/zig/blob/cbc3c0dc594341faddb8841fb466254f2f6f3b35/lib/std/elf.zig#L498)ようになりました (これまでは `anytype` だった)。[`fs.File.Reader` は以下のように内部的に `std.Io.Reader` を持ちます](https://github.com/ziglang/zig/blob/cbc3c0dc594341faddb8841fb466254f2f6f3b35/lib/std/fs/File.zig#L1117-L1127):

```zig
pub const Reader = struct {
    file: File,
    err: ?ReadError = null,
    mode: Reader.Mode = .positional,
    /// Tracks the true seek position in the file. To obtain the logical
    /// position, use `logicalPos`.
    pos: u64 = 0,
    size: ?u64 = null,
    size_err: ?SizeError = null,
    seek_err: ?Reader.SeekError = null,
    interface: std.Io.Reader,
    ...
};
```

`File` を引数に持つことからも分かるように、これは POSIX 実装を持つプラットフォームでしか使うことができません。つまり、これまで freestanding でも使えてきたストリーム型の ELF パーサが、もう POSIX 環境でしか使えなくなったということです。どうしてこうなった...。
一応、ELF イメージ全体を最初にすべてバッファに読んでからパースするための [API](https://github.com/ziglang/zig/blob/cbc3c0dc594341faddb8841fb466254f2f6f3b35/lib/std/elf.zig#L505) は提供されています。しかしながら、必要な部分だけを逐一ロードしてパースする従来のストリーム型パーサはもう UEFI 環境では使えないですね。

## アウトロ

良くも悪くも、最適化優先・メモリリソース優先という姿勢が顕著に現れたリリースだったのではないかと思います。Writergate という名前は、v0.9.0 において `*Allocator` を引数に取っていた API をすべて `Allocator` を取るように置き換えたという[スキャンダル](https://ziglang.org/download/0.9.0/release-notes.html#Allocgate) に準えて命名されているようです。自分は 0.9.0 のときは Zig を使っていなかったので知りませんし、今回の Writergate に関してもそもそも Reader/Writer をそこまで使ってこなかったということで実感できずにいます。
1.0 リリースに至るまでの間は思う存分破壊的変更をしてもらって、そのたびに顕著になる Zig の思想をもとに、この言語を使い続けるかを決めていくのが良いのではないでしょうか。ちなみに自分は乾パンとかも好きです。あと抹茶。

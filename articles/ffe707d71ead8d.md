---
title: "みんなでZigのきしょいところ(褒め言葉)を眺めよう"
emoji: "🤮"
type: "tech"
topics: ["zig"]
published: true
---

## さっそくですが

以下のコードをご覧ください:

```zig
const std = @import("std");
const log = std.log;

fn SomeType(n: u2) type {
    return struct {
        const Self = @This();
        const LowerType = switch (n) {
            3 => Two,
            2 => One,
            1 => Zero,
            0 => struct {},
        };

        value: u64,

        fn printMeGetChild(self: Self) LowerType {
            log.debug("{s}: {d} (lower={s})", .{
                @typeName(Self),
                self.value,
                @typeName(LowerType),
            });
            return .{ .value = self.value };
        }

        fn printMe(self: Self) void {
            log.debug("{s}: {d}", .{ @typeName(Self), self.value });
        }
    };
}

const Zero = SomeType(0);
const One = SomeType(1);
const Two = SomeType(2);
const Three = SomeType(3);

pub fn main() !void {
    const two = Two{ .value = 99 };
    _ = two.printMeGetChild().printMeGetChild();
    const zero = Zero{ .value = 42 };
    zero.printMe();
}
```

実行結果:

```txt
debug: main.SomeType(2): 99 (lower=main.SomeType(1))
debug: main.SomeType(1): 99 (lower=main.SomeType(0))
debug: main.SomeType(0): 42
```

コードの意図が分からなくてきしょいというのは置いておいて、なんとも言えぬきしょさを感じることと思います。
`SomeType()`関数は、C++でいうところのテンプレートクラスを生成してくれる関数です。この関数は引数に応じて`struct`型を生成して返します。生成された型は共通して`value`というメンバ変数を持ちます。また、自分よりも1レベル低い(?)型を`LowerType`として保持し、この型は`printMeGetChild()`メソッドにおいて型の名前が表示されます。また、`printMeGetChild()`メソッドは自身の`value`を引き継いで`LowerType`型のインスタンスを生成して返します。
Zigを知っている人からすると、おそらくこのコードは何の文句もなく合法であり、きしょいところは一つもないのではないかと思います。でもやっぱりぱっと見きしょいです。以下、なぜこの気持ちが湧いてくるのかを考えてみましょう。

## きしょポイント

### 1. テンプレートクラスみたいなものからインスタンス化された型を参照している

やっぱり一番違和感があるのは`LowerType`の`switch`文ですよね。なにせこの部分、`SomeType()`を使ってインスタンス化された型(`One`とか)を参照しています。テンプレートの中からインスタンスを触っている、ぞくぞくしますね。

### 2. `Zero`の`LowerType`が存在しないのに合法

`Zero`の`LowerType`は、空の`struct`になっています。こいつは空なので、当然`printMeGetChild()`における以下のインスタンス化が違法です:

```zig
            return .{ .value = self.value };
```

しかし、`Zero.printMeGetChild()`を呼び出さない限りは全く問題ありません。`Zero`型を定義することも問題なければ、`zero`インスタンスを生成することも合法です。未定義でもなんでもなく、意図したとおりに動きます。きしょいですね。
これは、Zigの原則(?)の一つである「参照されない限りは評価されず、評価されない限りは(lexicalに正しい限り)間違っていても良い」というのが理由になっています。

## コードの意図

このコードは、x64のページテーブルを書いている時に現れました。x64にはページテーブルが4レベル(または5レベル)あるんですが、中身はだいたい一緒なので共通化しようと思ったらこんなコードになりました。気になる人は以下のコードを見てみてください:
https://github.com/smallkirby/ymir/blob/1813870ffbe7c60553e94a173cb1e455eab9aa0e/ymir/arch/x86/page.zig#L469

## 最後に

思いつきで書いたけど、意外にそんなに書くことがありませんでした。
きしょいところと書いたけど、ぼくは意外と嫌いじゃないです。むしろZigの好きなところかもしれない。参照されない限り評価されないので、型だけを実装してまだ利用していない時にはエラーが出ず、のちのち利用する時にエラーがでるのはちょっと嫌ですが。それを防ぐには`std.testing.refAllDecls()`を使えばいいので、許容範囲です。

みんなでZigの成長を見守っていきましょう。

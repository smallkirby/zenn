---
title: 自作OS on Raspberry Pi 5 - Part 2. Ethernet MAC
emoji: 🍓
type: tech
topics:
  - Zig
  - 自作OS
  - Raspberry Pi 5
  - aarch64
  - Ethernet
published: true
---

「気が向いたので自作 OS on Raspberry Pi 5 in Zig を書いてみよう」シリーズのパート2です。

https://github.com/smallkirby/urthr/tree/61bd2910c4620e91c9ea6b1ef5f697f9ed7071a6

---

## 概要

せっかく実機でやるなら *卍 The Internet... 卍* に繋ぎたいと思うのが人のサガです。 というわけで、今回は Ethernet MAC のドライバを書いて適当な Ethernet パケットを受信できるようにするまでを目標とします。
Raspberry Pi 5 には、Cadence Gigabit Ethernet MAC (GEM) が RP1 Southbridge 経由で接続されています。[RP1 データシート](https://pip.raspberrypi.com/documents/RP-008370-DS-1-rp1-peripherals.pdf) によると、この MAC は 10/100/1000 Mbps をサポートし、RGMII インタフェースを持っているようです。今回書く GEM ドライバは 1000 Mbps / RGMII を使うように作ります。

## PHY のリセット

PHY 層のリセットは GPIO で行う模様。Active Low なので assert が pull-down / de-assert が pull-up になる。使う GPIO ピンは 32 番で、function select は 5 番。assert 中にやることは特になさそうなので、5ms 程度待った後 de-assert した。この手順を抜かすと MDIO アクセスで返ってくる値が全部 `0xFFFF` になったので、おそらく必要な手順。

## ソフトウェアリセットと Auto-Negotiation

MAC と PHY 層のやり取りは、[MDIO](https://www.n-s-c.co.jp/Ethernet-TSN/network/physical-layer/physical-layer-xmll/3823/) というインタフェースを介して行う。MDIO が何者か詳しくは知らないが、PHY アドレスとレジスタアドレスを指定するとデータを読んだり書いたりできるらしい。例えばレジスタ `0x2`, `0x3` は PHY ID を持っている (手元のやつは `600D-84A2` だった)。存在しない PHY に対して read すると `0xFFFF` が返ってくる事を利用し、PHY アドレスを enumeration したところ、どうやら使っている PHY アドレスは `0x1` のようだった。なお、MDIO コマンドの発行自体は GEM のレジスタ `Man` への I/O を介して行う。RP1 における GEM モジュールのオフセットは `0x0010_0000` で、今回利用した GEM のレジスタは以下に列挙するもののみ:

```zig
const gem = mmio.Module(.{ .size = u32 }, &.{
    .{ 0x0000, Ncr },
    .{ 0x0004, Ncfgr },
    .{ 0x0008, Nsr },
    .{ 0x0010, Dmacfg },
    .{ 0x0018, Rxbqb },
    .{ 0x001C, Txbqb },
    .{ 0x0020, Rsr },
    .{ 0x0034, Man }, // <== MDIO 用
    .{ 0x0088, Sa1b },
    .{ 0x008C, Sa1t },
    .{ 0x00C0, Usrio },
    .{ 0x00FC, Mid },
    .{ 0x0158, Rxcnt },
    .{ 0x0280, Dconfig1 },
    .{ 0x04C8, Txbqbh },
    .{ 0x04D4, Rxbqbh },
});
```

PHY の存在が確認できたら、次は MDIO を介して PHY のソフトウェアリセットを行う。レジスタ `0` に `0x8000` を書き込むとリセットできるらしい。さっき物理的にリセットしたのに、ここでまたリセットをかける必要があるのかと思ってスキップしてみたら普通に動いたので、多分このソフトウェアリセットは不要な気がする。
続いてリンクアップを行う。Auto-Negotiation の前に **ANAR: Auto-Negotiation Advertisement Register** (`0x4`) をセットして要求する duplex / speed などなどを指定する。今回は `0x01E1` を指定した。よく分からないけど、ANAR に 1000BASE-T 用の設定が無いのはなんで？その後 **BMCR: Basic Mode Control Register** (`0x0`) を使って Auto-Negotiation を開始する。Negotiation 開始後は、定期的に **BMSR: Basic Mode Status Register** (`0x1`) を監視して、negotiation と link-up が済んだことを確認すれば完了。Auto-Negotiation 開始から完了までは体感で 1~2 秒程度かかっている気がする。リンクアップすると、LAN ポートの LED ランプが点灯する。平安時代の人はこの明かりを夜中に見て蛍だと勘違いし、風情があると感じていたようだ。

![](/images/lanport-hotaru.jpeg)

## RxQueue の設定

まずは適当なパケットを受信したい。同一ネットワークに接続された Linux マシンから毎秒 ARP パケットを流す & RPi5 側の **NCFGR** レジスタで Promiscuous モードにしておけば、とりあえずはパケットが受信できるはず。そのために、RxQueue を用意する。RxQueue は 64-bit addressing の場合 128bit のディスクリプタの配列で、以下のような構造になっている:

```zig
/// RX descriptor for 64-bit addressing.
const Desc = packed struct(u128) {
    /// Buffer address (lower 32 bits).
    addr_lo: u32,
    /// Control and status.
    ctrl_stat: Control,
    /// Buffer address (upper 32 bits).
    addr_hi: u32,
    /// Reserved.
    _rsvd: u32 = 0,
};
```

Control / Status は MAC 側が書き込み、受信したフレーム長などの情報を保持する。なお、`addr_lo` の下位 2bit はそれぞれ特殊な意味を持っている:

- bit 0: USED
    - これが `0` の場合は MAC が所有。MAC がパケットをバッファに書き込むと `1` にセットされ、SW 側がバッファを読むことができる
- bit 1: WRAP
    - Queue 内の最後のディスクリプタで `1` にする

RxQueue とそこで指定される受信バッファは MAC が DMA で読み書きする。Queue のアドレスは **RXBQB** (`0x0018`) と **RXBQBH** (`0x04D4`) にセットする。なお、DMA に使うバッファは Normal Cacheable としてマップしているため、RxQueue をセットするときには cache clean (write-back) / 読むときには cache invalidate する必要がある:

```zig
pub fn init(self: *RxQueue) PageAllocator.Error!void {
    const descs = self.getDescs();
    for (descs[0..], 0..) |*desc, i| {
        const buffer = try self.createBuffer();
        self.buffers[i] = buffer;

        desc._rsvd = 0;
        desc.setAddr(self.translateD(buffer));
        desc.setHwOwn();
        if (i == num_desc - 1) {
            desc.setWrap();
        }

        desc.ctrl_stat = std.mem.zeroInit(Control, .{});
    }

    arch.cache(.clean, self.memory, self.memory.len);
}
```

これでパケットが受信できる！とわくわくしたものの、Linux 側から ARP を送っても **USED ビットがセットされたディスクリプタは1つも無かったのだった...**。

## 受信できない問題のデバッグ

以下、受信できない原因をなんとか探したログ。なんやかんやで原因特定まで2週間ほどかかっている。途中1週間くらい Claude Code に原因を見つけさせようと思って CLI だけで Urthr を動かせる環境を整えたが、Claude Code は全く見当違いなことを言ってコードを汚すだけだったので結局人力でやった。これのせいで1ヶ月分のトークンを消費したようで、Monthly Limit に引っかかった。食い逃げされた気分。トークンを食って遊ぶ化け物め。

### GEM のレジスタ

まず、GEM のレジスタを読んでエラーなどが無いかを確認した。**RXCNT** (`0x0158`) には MAC が検知した受信フレーム数が格納される。受信を有効化して数秒待った後にこの値を確認すると、`6` になっていた。これは、**PHY → MAC まではちゃんとフレームが受信・検知できている**ことを示している。
**RSR** (`0x0020`) には Receive Status が記録されている:

```zig
/// RX Status Register.
const Rsr = packed struct(u32) {
    /// Buffer not available.
    bna: bool,
    /// Frame received.
    received: bool,
    /// Overrun.
    overrun: bool,
    /// Reserved.
    _rsvd: u29 = 0,
};
```

同様にして数秒待った後に読んだところ、*Buffer Not Available* ビットだけがセットされていた。これにより、**RxQueue または受信バッファが DMA で読めていない** ことが確定した。

### PCIe Advanced Error Reporting

前述したように GEM は RP1 と直接接続されており、RP1 は SoC と PCIe 接続されている。つまり、GEM の DMA は PCIe を介して行われることになる。PCIe でエラーのログとか残ってないかな〜〜と思いながら RP1 のデータシートを読んでいると、以下の記述が:

> The PCIe EP controller has been configured with:
> - Advanced Error Reporting capability

どうやら PCIe には [AER: Advanced Error Reporting](https://docs.kernel.org/PCI/pcieaer-howto.html) という機能があるらしい。今回関係しているのは EP Controller というよりも PCIe Bridge (RPi5 だと `0002:00:00.0`) の方なので、まずは機能が存在するかを確認してみる。PCI Configuration Space の `+0x100` の位置以降には **Extended Capability Register** が置いてある。このレジスタ群は必ず以下のヘッダを持つ:

```zig
pub const ExtCapHeader = packed struct(u32) {
    /// Capability ID.
    id: u16,
    /// Version.
    version: u4,
    /// Next Capability Pointer.
    next: u12,
};
```

`next` フィールドは次の capability の configuration space 先頭からのオフセットを保持しており、これによって単方向リストが形成されている。AER の Capability ID は `0x1` であるため、PCIe Bridge のこのリストをたどって `id == 1` の capability を探したところ、ちゃんと存在していた。Bridge も AER をサポートしているらしい。
AER は主に *Uncorrectable Error* (`+0x04`) と *Correctable Error* (`+0x10`) の2つのエラー状態を記録してくれる。これらはビットフィールドになっており、各フィールドがエラーの発生状態を記録する。また、各エラーのレポート機能を有効にするには対応するマスク (`+0x08`, `+0x14`) を外してやる必要がある:

```zig
/// AER register set starting from the extended capability header.
const Aer = mmio.Module(.{ .size = u32 }, &.{
    .{ 0x00, dd.pci.ExtCapHeader },
    .{ 0x04, AerUncorrectableErr },
    .{ 0x08, AerUncorrectableMask },
    .{ 0x0C, AerUncorrectableSeverity },
    .{ 0x10, AerCorrectableErr },
    .{ 0x14, AerCorrectableMask },
    .{ 0x18, AerCorrectableSeverity },
    .{ 0x1C, mmio.Marker(.log0) },
    .{ 0x20, mmio.Marker(.log1) },
    .{ 0x24, mmio.Marker(.log2) },
    .{ 0x28, mmio.Marker(.log3) },
});

fn initAer() void {
    ...
    // Unmask all error reporting.
    aer.write(AerUncorrectableMask, 0);
    aer.write(AerCorrectableMask, 0);
}
```

Bridge の設定時に `initAer()` を呼び、受信失敗後に AER の値を見てみると以下のように Correctable Error の *Non-Fatal Advisory Error* が記録されていた:
```sh
[INFO ] brcstb  | Bridge AER Offset 04: 00100000 <== Uncorrectable Error
[INFO ] brcstb  | Bridge AER Offset 08: 00000000
[INFO ] brcstb  | Bridge AER Offset 0C: 00462030
[INFO ] brcstb  | Bridge AER Offset 10: 00002000 <== Correctable Error
[INFO ] brcstb  | Bridge AER Offset 14: 00000000
[INFO ] brcstb  | Bridge AER Offset 18: 000000B4
[INFO ] brcstb  | Bridge AER Offset 1C: 00000001
[INFO ] brcstb  | Bridge AER Offset 20: 0100000F
[INFO ] brcstb  | Bridge AER Offset 24: 00005000 <== Header Logs [3]
[INFO ] brcstb  | Bridge AER Offset 28: 00000000
[INFO ] brcstb  | Bridge AER Offset 2C: 00000000
[INFO ] brcstb  | Bridge AER Offset 30: 00000000
[INFO ] brcstb  | Bridge AER Offset 34: 00000000
```

なお、このときの RxQueue の先頭は以下のようになっていた。左側のアドレス表示は仮想アドレスであり、`FFFF000000005000`  は物理アドレス `0000000000005000` にマップされていることに注意:
```sh
[DEBUG] gem     | FFFF000000005000 | 00 60 00 00 00 00 00 00 00 00 00 00 00 00 00 00
[DEBUG] gem     | FFFF000000005010 | 00 70 00 00 00 00 00 00 00 00 00 00 00 00 00 00
[DEBUG] gem     | FFFF000000005020 | 00 C0 20 00 00 00 00 00 00 00 00 00 00 00 00 00
[DEBUG] gem     | FFFF000000005030 | 00 D0 20 00 00 00 00 00 00 00 00 00 00 00 00 00
[DEBUG] gem     | FFFF000000005040 | 00 E0 20 00 00 00 00 00 00 00 00 00 00 00 00 00
```

ご覧の通り RxQueue は物理アドレス `0x5000` に配置されており、RXBQB にも `0x5000` を指定している。しかし、このアドレスは `AER Offset 24` (Header Log) に記録されてしまっている。つまり、**DMA として PCIe TLP は発行されているものの、PCIe Bridge で拒否されていることが確定した**。

### Inbound Address Translation と DeviceTree

:::message
以下の情報はとりあえず動いた設定と DeviceTree の記述と動かしたときの挙動をもとに、自分がこういうものっぽいと想像した内容です。筆者は PCIe にも DeviceTree にも DMA にも詳しくないため、以下の情報は誤りを含んでいる可能性がめっちゃくちゃあります。詳しい人、ぜひ正解を教えてください。教えるときは優しく教えてください。厳しい口調で教えてきたら、右瞼をひっくり返します！
:::

:::message
(追記: 2026.01.26) 
**この章に書いたことは、嘘でした**。Raspberry Pi 5 のフォーラムで質問したところ、RP1 ATU はそんな変換をしない (透過的) だそうです。以下の内容は嘘言ってるなぁと思いながら読んでください。未だに正解は分かっていないけど、とりあえずは `dma-ranges` が示すとおりホスト RC が PCI `0x10_0000_0000` からの 64GiB をシステムRAMとして解釈する (設定不可) ようになっているんじゃないかなぁと思っています。まぁこれも嘘の可能性がめっっちゃありますけどね。
質問スレッドはこちら→ https://forums.raspberrypi.com/viewtopic.php?p=2360460
:::

さて、そもそもなぜ DMA 用のアドレスとして CPU 物理アドレスと同じ `0x5000` を指定していたかという話になる。RPi5 の PCIe Root Bridge には **Inbound Address Translation** という仕組みが存在しており、自分は以下のように設定していた [^1]:

| PCIe Address | AXI Address | Size   |
| ------------ | ----------- | ------ |
| `0`          | `0`         | 64 GiB |

つまり、PCIe と AXI アドレスを identity-map したつもりになっていた。RXBQB に書いた PCIe アドレス `0x5000` は、Address Translation で AXI `0x5000` に変換されると考えていた。しかし、AER でレポートされている以上はどうも違うようだ。

ここで Linux における [RPi5 の DeviceTree](https://github.com/raspberrypi/linux/blob/619bd3498567e9db963e3d4cc2988b513d6cd841/arch/arm64/boot/dts/broadcom/bcm2712-rpi-5-b.dts#L134) を読むと以下の記述があった [^2]:

```dts
&rp1 {
	// PCIe address space layout:
	// 00_00000000-00_00xxxxxx = RP1 peripherals
	// 10_00000000-1x_xxxxxxxx = up to 64GB system RAM

	// outbound access aimed at PCIe 0_00xxxxxx -> RP1 c0_40xxxxxx
	// This is the RP1 peripheral space
	ranges = <0xc0 0x40000000
		  0x02000000 0x00 0x00000000
		  0x00 0x00410000>;

	dma-ranges =
	// inbound RP1 1x_xxxxxxxx -> PCIe 1x_xxxxxxxx
		     <0x10 0x00000000
		      0x43000000 0x10 0x00000000
		      0x10 0x00000000>,

	// inbound RP1 c0_40xxxxxx -> PCIe 00_00xxxxxx
	// This allows the RP1 DMA controller to address RP1 hardware
		     <0xc0 0x40000000
		      0x02000000 0x0 0x00000000
		      0x0 0x00410000>,

	// inbound RP1 0x_xxxxxxxx -> PCIe 1x_xxxxxxxx
		     <0x00 0x00000000
		      0x02000000 0x10 0x00000000
		      0x10 0x00000000>;
};
```

正直、**DeviceTree の記述が誰視点で Inbound / Outbound なのかという理解にはかなり自信がない**。とりあえず自分の理解としては以下のようになっている。 

#### `ranges`

`ranges` プロパティは子ノードと親ノードのアドレス空間の説明を記述しており、以下の3ブロックから構成される:

- *Child Address*: 今回は 3 cells
- *Parent Address*: 今回は 3 cells
- *Size*: 今回は 2 cells

つまり、Parent (PCIe) の `0` が Child (RP1) の `0xC0_4000_0000` に変換される。自分の Outbound Translation や BAR の設定も含めると、CPU からの Outbound アクセスは以下のように変換されて RP1 に届くと推察される (なお、CPU は PCIe を物理アドレス `0x001F_0000_0000` にマップしている):

| 変換する人                  | 変換内容                               | 説明                   |
| ---------------------- | ---------------------------------- | -------------------- |
| PCIe RC ATU (Outbound) | AXI `0x001F_0000_0000` → PCI `0x0` | CPU 側で決めた設定          |
| (BAR1)                 | (PCI `0x0`)                        | (BAR1 は PCI `0` を指定) |
| PCIe EP ATU (`ranges`) | PCI `0x0` → RP1 `0xC0_4000_0000`   | RP1 が決めた設定           |

これを踏まえた上で RP1 Datasheet を読み直すと、以下のような記述があった:

| System Addr (40-bit) Base | Proc Addr (32-bit) Base | Description           |
| ------------------------- | ----------------------- | --------------------- |
| `0xC0.4000.0000`          | `0x4000.0000`           | Periphs (APB0) 16x16k |

そして、各ペリフェラルのアドレスマップは `0x4000_0000` から始まるのだが、以下の記述があった:

> The addresses given in Table 2 are compatible with the processor cluster’s 32-bit view of the system address map. 40-bit bus masters internal to the chip (i.e. DMA) must use the System Addr Bases in Table 1. (注: Table 1 が前述したテーブル、Table2 が `0x4000_0000` から始まるペリフェラルのアドレステーブル)

先述の流れで変換された RP1 のアドレス `0xC0_4000_0000` は "System Addr" というもので、CM3 から見ると `0x4000_0000` と同値になっていそう。そして、`0x4000_0000` はまさに CM3 から見たペリフェラルのアドレスと一致している。Outbound Translation については、こんな感じでおおよそ納得した。

#### `dma-ranges`

続いて、`dma-ranges` プロパティ。これは Outbound Translation を表現した `ranges` に対して Inbound Translation を表現するものらしい。各エントリは `ranges` と同様に Child, Parent, Size のトリプレットになっている。今回の場合は、以下の3つが存在する:

| Child            | Parent           | Size             | Attribute           |
| ---------------- | ---------------- | ---------------- | ------------------- |
| `0x10_0000_0000` | `0x10_0000_0000` | `0x10_0000_0000` | Prefetchable, 64bit |
| `0xC0_4000_0000` | `0`              | `0x41_0000`      | 32bit               |
| `0`              | `0x10_0000_0000` | `0x10_0000_0000` | 32bit               |

1番目のエントリを見ると、`0x10_0000_0000` が `0x10_0000_0000` に identity-map されている。先頭付近のコメントを見ても、`0x10_0000_0000` から 64GiB 分がシステム RAM にマップされることを意図している模様。

### 成功と疑問点

さて、ここまでつらつらと書いてきたが、結果を先に書くと**以下のような設定にすることで DMA ができた**:

- RC Inbound Translation: PCI `0x10_0000_0000` → AXI `0`
- RXBQB に設定するアドレス: 物理アドレス + `0x10_0000_0000`

おそらくこれにより、以下のような変換経路を辿ることになると推測している:

| 変換する人                        | 変換内容                                    | 説明              |
| ---------------------------- | --------------------------------------- | --------------- |
| EP ATU (`dma-ranges`)        | `0x10_0000_5000` → PCI `0x10_0000_5000` | RXBQB に指定するアドレス |
| RC ATU (Inbound Translation) | PCI `0x10_0000_5000` → AXI `0x5000`     | CPU が設定したアドレス   |

これが正しいかどうかは全くわからない。だが、観測した挙動的にはこうなっていると考えられるような感じだった。だが、疑問点として `dma-ranges` の3つ目のエントリが気になっている。こいつは `0` → `0x10_0000_0000` に変換してくれる。だとすると、最初に誤って(?) `0x5000` を RXBQB に設定した場合でも `0x10_0000_5000` に変換してくれ、その後 RC ATU で AXI `0x5000` に変換してくれそうな気がしている。だが、実際に試してみると最初に見たのと同じエラーが AER で報告されることになる。Attribute を見ると 32bit memory になっているから GEM の 32-bit addressing ではないのかと思い試してみたが、それでも駄目だった。詳しい人、なにか知っていたら教えてください。

## 受信した ARP パケット

さてさて、何はともあれ Ethernet フレームを受信することができたので喜んでおこう。やった〜〜〜〜〜〜〜:

![](/images/mac-ether-packet.png)

さて、受信したパケットはこんな感じ:

```sh
[DEBUG] gem     | FFFF00000020C000 | FF FF FF FF FF FF 50 EB F6 2A C5 2C 08 06 00 01
[DEBUG] gem     | FFFF00000020C010 | 08 00 06 04 00 01 50 EB F6 2A C5 2C C0 A8 01 09
[DEBUG] gem     | FFFF00000020C020 | 00 00 00 00 00 00 C0 A8 01 13 00 00 00 00 00 00
[DEBUG] gem     | FFFF00000020C030 | 00 00 00 00 00 00 00 00 00 00 00 00 E8 CA 5F 91
```

分解するとこうなる:

```yml
Ethernet Header:
    FF_FF_FF_FF_FF_FF : Destination MAC
    50_EB_F6_2A_C5_2C : Source MAC
    08_06             : EtherType
ARP Header:
    00_01             : Hardware Type (Ethernet)
    08_00             : Protocol Type (IPv4)
    06                : Hardware Size
    04                : Protocol Size
    00_01             : Op (ARP Request)
    50_EB_F6_2A_C5_2C : Source MAC
    C0_A8_01_09       : Sender IP (192.168.1.9)
    00_00_00_00_00_00 : Target MAC
    C0_AB_01_13       : Target IP (192.168.1.19)
```

ARP を送っていた Linux はこんな感じ:

```sh
# NIC 情報
> ip link | grep enp -A1
2: enp7s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 50:eb:f6:2a:c5:2c brd ff:ff:ff:ff:ff:ff
# ARP を送ったコマンド
sudo arping -I enp7s0 192.168.1.19
```

正しく送れてそう！

## アウトロ

これでネットワークスタックを実装する準備が整ったのではないでしょうか。と言いたいところですが、今のところ Urthr にはまだ割り込みもなければユーザランドもなければカーネルスレッドもスケジューラもありません。でも、この辺のOSコアの部分ってあんまり実機固有の部分が無いので、RPi5 実機で動かす楽しみっていうものがあんまりないんだよなぁ。QEMU でやっても同じだし。ということで、次は xHCI ドライバでも書こうかと思っている今日この頃です。Bloodborne を未クリアのまま積んでいたことを思い出したので、はじめからやり直し始めました。ガスコインまで倒しました。

[^1]: Inbound Address Translation で参考にした資料: https://speakerdeck.com/tnishinaga/kernelvm-tokyo17
[^2]: `dma-ranges` の読み方: https://elinux.org/Device_Tree_Usage#Ranges_.28Address_Translation.29

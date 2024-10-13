+++
title = 'Rubyでゲームボーイのエミュレータを作った'
date = 2024-10-12T15:45:14+09:00
draft = false
+++

## はじめに
Rubyでゲームボーイのエミュレータを作って、rubyboyという名前のgemで公開しました！
(スターをいただけると嬉しいです！)

{{< github-repo-card "https://github.com/sacckey/rubyboy" >}}

![](https://storage.googleapis.com/zenn-user-upload/58504c563998-20240407.png)

<blockquote data-align="center" class="twitter-tweet" data-media-max-width="560"><p lang="ja" dir="ltr">Rubyでゲームボーイのエミュレータを作りました！<br>カラー対応やWasmでブラウザ対応もやっていきたい💪<br><br>GitHub: <a href="https://t.co/hFwmZD6FNp">https://t.co/hFwmZD6FNp</a> <a href="https://t.co/qWbx8v4mef">pic.twitter.com/qWbx8v4mef</a></p>&mdash; sacckey (@sacckey) <a href="https://twitter.com/sacckey/status/1769334370102587629?ref_src=twsrc%5Etfw">March 17, 2024</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

## この記事
RUBY BOYの実装手順を説明しながら、ハマった点や工夫した点を紹介します。
またRUBY BOYの高速化のためにやったことを紹介します。

## なぜゲームボーイのエミュレータをつくったのか
- なにか個人開発をしたいが、Webサービスは維持費がかかるので無料で維持できるものを作りたい
- 業務でRubyを使っていることもあり、以前からRubyのgemを作ってみたかった
- ゲームのエミュレータ開発は「ゴールが明確＆動くと楽しい」ので、モチベを維持しやすそう
  - 特にゲームボーイには思い入れがある

→ Rubyでゲームボーイのエミュレータを作って、gemで公開しよう！

## エミュレータの概要
以下は、ゲームボーイのアーキテクチャです。
<p align="center">
  <img src="https://storage.googleapis.com/zenn-user-upload/2a6af06959a6-20240408.png" width="500">
  <em>"Game Boy / Color Architecture - A Practical Analysis" by Rodrigo Copetti, Published: February 21, 2019, Last Modified: January 9, 2024. Available at: https://www.copetti.org/writings/consoles/game-boy/. Licensed under Creative Commons Attribution 4.0 International License.</em>
</p>

これらのハードウェアを模倣したプログラムを実装することを目指します。
RUBY BOYのクラス図と各クラスの役割は以下の通りです。

![](https://storage.googleapis.com/zenn-user-upload/c11260f6c5a5-20240420.jpeg)

- Console: mainクラス
- Lcd: 画面描画を行う
- Bus: メモリマップドI/Oを実現するためのコントローラ。Cpuから各種ハードウェアの設定値の読み書きを中継する
- Cpu: Romから命令を読み込み、解釈して実行する
- Registers: レジスタの読み書きを行う
- Cartridge: カセット内のROMやRAMの読み書きを行う。MBCチップの種類ごとに実装が異なる(後述)
- Apu: オーディオデータの生成を行う
- Rom: カートリッジ内のゲームプログラムを読み込む
- Ram: カートリッジ及びゲームボーイ内のRAMデータの読み書きを行う
- Interrupt: 割り込みを管理する。割り込みは次の3クラスから行われる
- Timer: サイクル数をカウントする
- Ppu: ディスプレイに描画するピクセル情報を生成する
- Joypad: ゲームボーイのボタン入力を受け取る

RUBY BOYでは、Cpuが命令を実行し、実行にかかったサイクル数だけPpu, Timer, Apuのサイクル数を進めていくことでコンポーネント間の同期をとります。
そのためメインループの中身は以下のようになります。
```ruby
cycles = @cpu.exec
@timer.step(cycles)
@apu.step(cycles)
if @ppu.step(cycles)
  @lcd.draw(@ppu.buffer)
  key_input_check
  throw :exit_loop if @lcd.window_should_close?
end
```

## 実装の流れ
実装中のメモを[スクラップ](https://zenn.dev/sacckey/scraps/380c2f3ad3318d)にまとめています。ここから抜粋して説明していきます。

### UIの実装
画面描画、オーディオ再生、キーボード入力を行うUI部分は、[Ruby-FFI gem](https://github.com/ffi/ffi)経由でSDL2を使用して実装しました。
必要なSDL2のメソッドを集約した[ラッパークラス](https://github.com/sacckey/rubyboy/blob/main/lib/rubyboy/sdl.rb)を作成し、そこからSDL2のメソッドを呼び出す設計になっています。

### ROMの読み込み
まずはゲームのデータを読み込んで使えるようにしました。
例えばタイトルは0x0134~0x0143に入っているので、以下のように取得できます。

```ruby
data = File.open('tobu.gb', 'r') { _1.read.bytes }
p data[0x134..0x143].pack('C*').strip
=> "TOBU"
```

### MBC(メモリバンクコントローラ)の実装
ゲームボーイでは多くのゲームでMBC(メモリバンクコントローラ)が使用されており、バンク切り替えによってアドレス空間の拡張を実現しています。
MBCチップにはMBC1, MBC3, MBC5などの種類があり、それぞれで使えるROMやRAMのサイズが異なるので、MBCチップの種類に応じた実装が必要です。
RUBY BOYはNoMBC(MBCなし)とMBC1のゲームに対応しており、Factoryパターンで適切なMBCの実装を返すようにしました。チップの実装と場合分けを追加すれば、他の種類のMBCチップにも対応可能です。

```ruby
module Rubyboy
  module Cartridge
    class Factory
      def self.create(rom, ram)
        case rom.cartridge_type
        when 0x00
          Nombc.new(rom)
        when 0x01..0x03
          Mbc1.new(rom, ram)
        when 0x08..0x09
          Nombc.new(rom)
        else
          raise "Unsupported cartridge type: #{rom.cartridge_type}"
        end
      end
    end
  end
end
```

### CPUの実装
以下のCPUの実行サイクルを繰り返すプログラムを実装します。
- ROMから命令をフェッチ
- 命令のデコード
- 命令の実行

モチベ維持のためにも、いきなりCPUやPPUの処理を全て実装するのではなく、まずは以下の最小限のテストROMを動かすことを目標にしました。

{{< github-repo-card "https://github.com/dusterherz/gb-hello-world" >}}

デバッグのために[BGB](https://bgb.bircd.org/)というゲームボーイのエミュレータを使用しました。レジスタやメモリの情報を表示しながら1ステップずつ実行できるので、自身のCPUと動作を比較でき、便利です。

BGBで実行される命令を確かめ、それらを実装していきます。
PPUもbgだけ描画処理を作ればテストが動きます。
<p align="center">
  <img src="https://storage.googleapis.com/zenn-user-upload/a05ad4a1c905-20240330.png" width="500">
</p>

次にすべての命令及び割り込み処理を実装していき、[cpu_instrs](https://github.com/retrio/gb-test-roms/tree/master/cpu_instrs/)と[instr_timing](https://github.com/retrio/gb-test-roms/tree/master/instr_timing)というCPUのテスト用ROMのテストがパスすることを目指します。

#### ここではまったポイント
- pop afでfの値をsetするとき、下位4bitを0000にしていなかった
  - fレジスタの下位4bitは常に0000
- opcode=0xe8, 0xf8の命令のcフラグの計算方法が間違っていた
  - cflag = (@sp & 0xff) + (byte & 0xff) > 0xff　で通った

これらを修正することで、無事にパスしました。
<p align="center">
  <img src="https://storage.googleapis.com/zenn-user-upload/4c3b6b917e15-20240330.png" width="500">
</p>

...なんか描画がおかしいですが、CPUの処理はOKです。

### PPU
残りの描画処理を実装して、PPU用のテストROMである[dmg-acid2](https://github.com/mattcurrie/dmg-acid2)をパスすることを目指します。

windowとspriteの描画、割り込み処理、DMA転送を実装するとテストをパスします。
sprite表示の優先順位に注意が必要です。
<p align="center">
  <img src="https://storage.googleapis.com/zenn-user-upload/92f2cf1d5e93-20240414.png" width="500">
</p>

描画ができるようになったので、あとはJoypadを実装するとゲームを動かせるようになります。
以下は[Tobu Tobu Girl](https://tangramgames.dk/tobutobugirl/)というゲームを動かしたときの動画です。

<blockquote data-align="center" class="twitter-tweet" data-media-max-width="560"><p lang="ja" dir="ltr">Tobu Tobu Girl動いた！<br>30fpsぐらいしか出ていないので、最適化する <a href="https://t.co/szPtv3F37R">pic.twitter.com/szPtv3F37R</a></p>&mdash; sacckey (@sacckey) <a href="https://twitter.com/sacckey/status/1727284351371796791?ref_src=twsrc%5Etfw">November 22, 2023</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

ここまででゲームが動作するようになりました！が、とてつもなく遅いです。
ここからは高速化をしていきました。

## 高速化
RUBY BOYの高速化のためにやったことを紹介していきます。
ここからの話はエミュレータ実装に限らず、一般的なRubyプログラムの高速化に使えると思います。

実行環境
- PC: MacBook Pro(13-inch, 2018)
- プロセッサ: 2.3 GHz クアッドコアIntel Core i5
- メモリ: 16 GB 2133 MHz LPDDR3

### ベンチマーク
Tobu Tobu Girlの最初の1500フレームを音声と描画なしで実行したときにかかる時間を3回測定しました。
ベンチマークは何度も行うことになるので、専用のプログラムを用意しておき、コマンド実行でベンチマークをすぐに開始できる状態をつくることをおすすめします。

### プロファイラ
Stackprofというgemを使いました。

{{< github-repo-card "https://github.com/tmm1/stackprof" >}}

測定したい箇所をブロックで囲うだけで使うことができ、オーバーヘッドも少ないのでおすすめです。

### 高速化Part1
#### YJITの有効化
RubyのJITコンパイラであるYJITを有効にすることでFPSが向上しました。
Ruby 3.2から実用段階になっており、実行時に`--yjit`オプションを追加すると有効になります。

```
Ruby: 3.2.2
YJIT: false
1: 36.740829 sec
2: 36.468515 sec
3: 36.177083 sec
FPS: 41.1385591742566

Ruby: 3.2.2
YJIT: true
1: 32.305559 sec
2: 32.094778 sec
3: 31.889601 sec
FPS: 46.73385499531633
```

FPS: 41.1385591742566 → 46.73385499531633

#### sprite用のハッシュを毎回作らないようにする
Stackprofを実行してみると、render_spritesというメソッドがボトルネックになっていることがわかります。

```
==================================
  Mode: cpu(1000)
  Samples: 9081 (1.08% miss rate)
  GC: 4 (0.04%)
==================================
     TOTAL    (pct)     SAMPLES    (pct)     FRAME
      3727  (41.0%)        1920  (21.1%)     Rubyboy::Ppu#render_sprites
      1800  (19.8%)        1800  (19.8%)     Rubyboy::Operand#initialize
      1448  (15.9%)        1448  (15.9%)     Integer#zero?
      3346  (36.8%)        1296  (14.3%)     Enumerable#each_slice
       919  (10.1%)         919  (10.1%)     Integer#<<
       424   (4.7%)         424   (4.7%)     Integer#<=>
      3552  (39.1%)         294   (3.2%)     Array#each
...
```

さらにrender_spritesの中でどこがボトルネックなのかを調べてみます。

```
  code:
                                  |   220  |     def render_sprites
    3    (0.0%)                   |   221  |       return if @lcdc[LCDC[:sprite_enable]].zero?
                                  |   222  |
    2    (0.0%)                   |   223  |       sprite_height = @lcdc[LCDC[:sprite_size]].zero? ? 8 : 16
                                  |   224  |       sprites = []
                                  |   225  |       cnt = 0
 3346   (36.8%)                   |   226  |       @oam.each_slice(4).each do |sprite_attr|
                                  |   227  |         sprite = {
                                  |   228  |           y: (sprite_attr[0] - 16) % 256,
                                  |   229  |           x: (sprite_attr[1] - 8) % 256,
                                  |   230  |           tile_index: sprite_attr[2],
                                  |   231  |           flags: sprite_attr[3]
                                  |   232  |         }
                                  |   233  |         next if sprite[:y] > @ly || sprite[:y] + sprite_height <= @ly
                                  |   234  |
                                  |   235  |         sprites << sprite
                                  |   236  |         cnt += 1
   15    (0.2%) /    15   (0.2%)  |   237  |         break if cnt == 10
 1887   (20.8%) /  1887  (20.8%)  |   238  |       end
  386    (4.3%) /    12   (0.1%)  |   239  |       sprites = sprites.sort_by.with_index { |sprite, i| [-sprite[:x], -i] }
                                  |   240  |
...
```

226行目のブロックが高い実行時間の割合を占めています。よく見ると、spriteというHashを作ったあとに、条件を満たすのであればspritesという配列にspriteを追加しています。
ここを条件を満たすときのみspriteを作るように修正した結果、速度が向上しました。

FPS: 46.73385499531633 → 49.2233733053377

このように、地道にボトルネックの発見と修正を繰り返していきます。

#### PPUのリファクタリング
「ループの外でできることはループの外でやる」という基本的なことですが、
エミュレータにおいては致命的に重要で、解消することでパフォーマンスがみるみる向上します。

##### tile_map_addrの計算をループの外で行う
[commit](https://github.com/sacckey/rubyboy/commit/08273596838631334ee83fbd540feab5da09233e)
FPS: 49.2233733053377 → 56.6580741129914

##### tile_indexの計算をループの外で行う
[commit](https://github.com/sacckey/rubyboy/commit/3e79628cf69f15bc690049f1695e3767ae48e8a9)
FPS: 56.6580741129914 → 60.44140113483162

#### Ruby v3.2 -> v3.3
ここまでで描画無しで60FPSぐらい出るようになったのですが、行き詰まってしまいました。
高速化とコードの可読性はトレードオフです。例えば定数の使用を辞めて謎の整数をそのまま書き込んだほうが速くなりますが、やりたくないです。

そんなことを悩んでいると2023/12/25にRuby 3.3.0がリリースされました。
Ruby 3.3はYJITが[更に速くなったとのこと](https://k0kubun.hatenablog.com/entry/ruby-3-3-yjit)なので、とりあえずアップデートしてみると...

![](https://storage.googleapis.com/zenn-user-upload/0809e3724053-20240330.png)

!?

なんかめっちゃはやくなった！！！！！
想像以上にRuby 3.3ははやくなってました。ありがとうございます🙏
ちなみにこのときの[比較post](https://twitter.com/sacckey/status/1740342734857306595)がMatzにもリポストされました。嬉しい。

### GCを減らす
Ruby 3.3のおかげで速度が出るようになったのですが、そのかわり？にGCがかなり増えてしまいました。

```
rubyboy % stackprof stackprof-cpu-myapp.dump
==================================
  Mode: cpu(1000)
  Samples: 16405 (4.57% miss rate)
  GC: 5593 (34.09%)
==================================
     TOTAL    (pct)     SAMPLES    (pct)     FRAME
      3688  (22.5%)        3688  (22.5%)     (sweeping)
      2332  (14.2%)        2109  (12.9%)     Enumerable#flat_map
      2050  (12.5%)        2050  (12.5%)     Integer#<=>
      5593  (34.1%)        1679  (10.2%)     (garbage collection)
      1038   (6.3%)        1038   (6.3%)     Rubyboy::Ppu#to_signed_byte
      1004   (6.1%)        1004   (6.1%)     Rubyboy::SDL.RenderClear
       646   (3.9%)         646   (3.9%)     Rubyboy::Ppu#get_pixel
       437   (2.7%)         437   (2.7%)     Integer#>>
       701   (4.3%)         332   (2.0%)     Rubyboy::Ppu#render_sprites
      1354   (8.3%)         278   (1.7%)     Rubyboy::Lcd#draw
      3825  (23.3%)         257   (1.6%)     Rubyboy::Ppu#step
      1627   (9.9%)         255   (1.6%)     Rubyboy::Ppu#render_bg
       633   (3.9%)         247   (1.5%)     Enumerable#each_slice
       230   (1.4%)         230   (1.4%)     Rubyboy::Registers#read8
       226   (1.4%)         226   (1.4%)     (marking)
...
```

また、ポケモン赤のタイトル画面〜博士の話のシーンの動作が依然重く、これらを解決することを次の目標にしました。

#### GCプロファイラ
GC発生箇所の検出にはHeapProfilerを使いました。

{{< github-repo-card "https://github.com/Shopify/heap-profiler" >}}

こちらも検出したい箇所をブロックで囲んで使う方式で簡単に使えるのですが、長い時間起動していると検出結果が返ってこなくなるので注意が必要です。

実行結果(一部)

```
rubyboy % heap-profiler tmp/report
Total allocated: 563.01 MB (4198804 objects)
Total retained: 10.13 kB (252 objects)

allocated memory by file
-----------------------------------
 454.17 MB  rubyboy/lib/rubyboy/cpu.rb
  93.18 MB  rubyboy/lib/rubyboy/ppu.rb
  10.06 MB  rubyboy/lib/rubyboy/apu.rb

allocated memory by class
-----------------------------------
 462.20 MB  Hash
  49.79 MB  Array
  14.61 MB  Enumerator

allocated objects by file
-----------------------------------
   2839605  rubyboy/lib/rubyboy/cpu.rb
   1105342  rubyboy/lib/rubyboy/ppu.rb
    251462  rubyboy/lib/rubyboy/apu.rb

allocated objects by class
-----------------------------------
   2888757  Hash
    416967  Array
    273888  <memo> (IMEMO)
    273888  <ifunc> (IMEMO)
    251442  Float

retained memory by file
-----------------------------------
   3.92 kB  rubyboy/lib/rubyboy/cpu.rb
   2.20 kB  rubyboy/lib/rubyboy/ppu.rb

retained objects by file
-----------------------------------
        98  rubyboy/lib/rubyboy/cpu.rb
        54  rubyboy/lib/rubyboy/ppu.rb
        24  rubyboy/lib/rubyboy.rb
        18  rubyboy/lib/rubyboy/lcd.rb
        18  rubyboy/lib/rubyboy/apu.rb
```

これを見ると、Cpuクラス内でHashを大量に作っていることがGC発生の原因であることがわかります。

#### Cpuクラス内のHashの生成を減らす
##### 命令の引数をHashからSymbolに変える
[commit](https://github.com/sacckey/rubyboy/commit/7cc25746dffcc91e9f6ad4162ec279e8547b9fe0)

```diff ruby
case opcode
-  when 0x01 then ld16({ type: :register16, value: :bc }, { type: :immediate16 }, cycles: 12)
+  when 0x01 then ld16(:bc, :immediate16, cycles: 12)
```

##### フラグの参照時にHashを作らないようにする
[commit](https://github.com/sacckey/rubyboy/commit/947617508e726716b3905d41df1f4a4c1b5793c8)

```diff ruby
- def flags
-   f_value = @registers.f
-   {
-     z: f_value[7] == 1,
-     n: f_value[6] == 1,
-     h: f_value[5] == 1,
-     c: f_value[4] == 1
-   }
- end

+ def flag_z
+   @registers.f[7] == 1
+ end

+ def flag_n
+   @registers.f[6] == 1
+ end

+ def flag_h
+   @registers.f[5] == 1
+ end

+ def flag_c
+   @registers.f[4] == 1
+ end
```

これらの修正によってGCを2.71%まで減らすことができました。

### 高速化Part2
#### Integer#<=>を減らす
この時点でのベンチマークとStackprofの結果は以下の通りです。
この結果は描画ありかつ、ポケモン赤の一番重い箇所に合わせて計測を行ったものなので、これまでの結果との比較は無意味であることに注意が必要です。

```
Ruby: 3.3.0
YJIT: true
1: 26.798767 sec
FPS: 55.97272441676141
```

```
==================================
  Mode: cpu(1000)
  Samples: 10430 (5.57% miss rate)
  GC: 283 (2.71%)
==================================
     TOTAL    (pct)     SAMPLES    (pct)     FRAME
      2275  (21.8%)        2275  (21.8%)     Integer#<=>
      1267  (12.1%)        1267  (12.1%)     Rubyboy::SDL.RenderClear
      1186  (11.4%)        1186  (11.4%)     Rubyboy::Ppu#to_signed_byte
      2366  (22.7%)         864   (8.3%)     Rubyboy::Ppu#render_bg
       784   (7.5%)         784   (7.5%)     Rubyboy::Ppu#get_pixel
      1773  (17.0%)         641   (6.1%)     Rubyboy::Ppu#render_window
       992   (9.5%)         415   (4.0%)     Rubyboy::Ppu#render_sprites
       334   (3.2%)         334   (3.2%)     Integer#>>
       852   (8.2%)         319   (3.1%)     Enumerable#each_slice
      5453  (52.3%)         311   (3.0%)     Rubyboy::Ppu#step
      4199  (40.3%)         213   (2.0%)     Integer#times
       188   (1.8%)         188   (1.8%)     Rubyboy::Timer#step
       187   (1.8%)         187   (1.8%)     (sweeping)
       142   (1.4%)         142   (1.4%)     Rubyboy::SDL.UpdateTexture
       129   (1.2%)         129   (1.2%)     Array#size
       426   (4.1%)         114   (1.1%)     Rubyboy::Ppu#get_color
       851   (8.2%)         109   (1.0%)     Array#each
       981   (9.4%)         105   (1.0%)     Rubyboy::Cpu#get_value
       283   (2.7%)          85   (0.8%)     (garbage collection)
...
```

GCは減っていますが60FPSは出ておらず、`Integer#<=>`(数値の比較)がボトルネックになっていることがわかります。
数値の比較は、以下のようなアドレスによる分岐で多く発生してしまいます。
```ruby
def read_byte(addr)
  case addr
  when 0x0000..0x7fff
    @mbc.read_byte(addr)
  when 0x8000..0x9fff
    @ppu.read_byte(addr)
...
```

この比較を無くすために、前処理で`@read_methods`という配列にアドレスと処理の対応を入れておき、実行時は配列参照のみで呼び出せるようにしました。
```ruby
def set_methods
  0x10000.times do |addr|
    case addr
    when 0x0000..0x7fff
      @read_methods[addr] = -> { @mbc.read_byte(addr) }
    when 0x8000..0x9fff
      @read_methods[addr] = -> { @ppu.read_byte(addr) }
...
```

この手法は、Rubyで書かれたファミコンエミュレータである[Optcarrot](https://github.com/mame/optcarrot)を真似しました。

再度benchとStackprofを実行してみます。

```
rubyboy % RUBYOPT=--yjit bundle exec rubyboy bench
Ruby: 3.3.0
YJIT: true
1: 21.75409 sec
FPS: 68.95255099156066
```

```
rubyboy % bundle exec stackprof stackprof-cpu-myapp.dump
==================================
  Mode: cpu(1000)
  Samples: 9505 (6.87% miss rate)
  GC: 325 (3.42%)
==================================
     TOTAL    (pct)     SAMPLES    (pct)     FRAME
      1238  (13.0%)        1238  (13.0%)     Rubyboy::Ppu#to_signed_byte
      1208  (12.7%)        1208  (12.7%)     Rubyboy::SDL.RenderClear
      2558  (26.9%)         907   (9.5%)     Rubyboy::Ppu#render_bg
       865   (9.1%)         865   (9.1%)     Rubyboy::Ppu#get_pixel
       849   (8.9%)         849   (8.9%)     Rubyboy::Cartridge::Mbc1#set_methods
      1803  (19.0%)         663   (7.0%)     Rubyboy::Ppu#render_window
      1053  (11.1%)         460   (4.8%)     Rubyboy::Ppu#render_sprites
      5782  (60.8%)         346   (3.6%)     Rubyboy::Ppu#step
       906   (9.5%)         343   (3.6%)     Enumerable#each_slice
       313   (3.3%)         313   (3.3%)     Integer#>>
      4412  (46.4%)         245   (2.6%)     Integer#times
       237   (2.5%)         237   (2.5%)     (sweeping)
       197   (2.1%)         197   (2.1%)     Rubyboy::Timer#step
       193   (2.0%)         193   (2.0%)     Rubyboy::SDL.UpdateTexture
      1141  (12.0%)         162   (1.7%)     Rubyboy::Bus#set_methods
       433   (4.6%)         134   (1.4%)     Rubyboy::Ppu#get_color
       114   (1.2%)         114   (1.2%)     Array#size
       478   (5.0%)         109   (1.1%)     Rubyboy::Cpu#get_value
       918   (9.7%)          99   (1.0%)     Array#each
        75   (0.8%)          75   (0.8%)     Rubyboy::Cpu#increment_pc_by_byte
      9180  (96.6%)          68   (0.7%)     Rubyboy::Console#bench
       325   (3.4%)          65   (0.7%)     (garbage collection)
        49   (0.5%)          49   (0.5%)     Integer#<=>
...
```

FPSが55.97272441676141から68.95255099156066になり、`Integer#<=>`の割合も21.8%から0.5%に減らすことができました。
まだまだ高速化はやる余地はありますが、目標を達成したのでここで一旦完了としています。

### 高速化結果

|Before(rubyboy v1.0.0, Ruby 3.2.2)|After(rubyboy v1.3.1, Ruby 3.3.0 + YJIT)|
|---|---|
|![](https://storage.googleapis.com/zenn-user-upload/b29a44007817-20240416.gif)|![](https://storage.googleapis.com/zenn-user-upload/6dff7cf43943-20240416.gif)|


## おわりに
### よかったこと
#### エミュレータ開発は楽しい
当初の目論見通り、楽しみながら実装することができました。エミュレータは動くと楽しいというのもありますが、ドキュメントやテストROMが充実していることが要因として大きいと思います。特にテストのROMが誤っている箇所をフィードバックしてくれるため途中で詰むことなく進めることができ、リファクタリングも気軽に行うことができました。
あと実家に眠っていたカセットを動かすことができてうれしい。

#### Rubyのgemを公開できた
前からRubyを書いていて、1つぐらいは何かちゃんと動くgemを公開したかったのでよかったです。
https://rubygems.org/gems/rubyboy

いますぐ `gem install rubyboy`！

#### 低レイヤ技術の勉強になった
CPU、メモリ、レジスタ、RAMなどの役割や動作について、それらを模倣したプログラムの実装を通じて知識を深めることができました。教科書的に知っていたことが実際に出てきて、「こういうことだったのか」という発見があり、楽しかったです。大学や高専の実験科目にいかがでしょうか。

#### プログラムの高速化の経験を積めた
「ベンチマークプログラムを作り、 プロファイラを実行して、怪しい箇所を修正する」という地道な高速化を経験しました。高速化の解像度が上がり、ボトルネック発見の嗅覚も鋭くなったのではないかと思っています。

#### 大きめなプログラムの設計と実装を経験した
Webプログラム以外で大きめなプログラムを書いたことがあまりなかったので、その経験を積めたのも良かったです。エミュレータだと、各クラスの責務を適切に分担することや、一般化したプログラムを書いて再利用性を高めるところ(特にCPU)が重要だと感じました。

### Rubyの感想
- Syntaxがシンプルでうれしい！
- 気の利くメソッドが多くてうれしい！
- 便利なgemが多くてうれしい！
- 処理系の速度が遅くてつらい！
    - YJITの進化のおかげで以前より格段にはやいのはうれしい！

### 今後
以下をやっていきたいと思っています。
- 描画バグの修正
- MBCタイプの追加
- ゲームボーイカラー対応
- Wasm対応
- ベンチマークまわりの整備
  - Rubyのベンチマークプログラムとして使えるようにしたい

## 参考資料
### 自作ブログ記事
ゲームボーイのエミュレータ作ってみた系の記事です。実装の進め方や工夫、ハマりポイントを参考にしました🙏

- [C++でゲームボーイエミュレータを自作しています | voidProc | ゲーム製作ログ](https://voidproc.com/blog/archives/664)
- [Rustでゲームボーイエミュレータを自作した話 - MJHD](https://mjhd.hatenablog.com/entry/2021/04/14/221813)
- [ゲームボーイのエミュレータを自作した話 · Keichi Takahashi](https://keichi.dev/post/write-yourself-a-game-boy-emulator/)
- [AQBoy: Yet Another Game Boy Emulator 開発記 - HackMD](https://hackmd.io/@anqou/HJcvRrwy9)
- [OCaml でゲームボーイエミュレータを書いた話 #関数型言語 - Qiita](https://qiita.com/linoscope/items/244d931aaae07df2c27e)

### 発表スライド
- [Ruby で高速なプログラムを書く | PPT](https://www.slideshare.net/mametter/ruby-65182128)
  - [Optcarrot](https://github.com/mame/optcarrot)の作者による発表スライド。Rubyプログラムの最適化の本質情報が盛り沢山です。RUBY BOYの高速化もこのスライドを参考にしながら進めることで達成できました。

### ドキュメント
- [Pan Docs](https://gbdev.io/pandocs/)
  - ゲームボーイの仕様が網羅されているページで、辞書のように使います。
- [Game Boy CPU (SM83) instruction set](https://gbdev.io/gb-opcodes/optables/)
  - CPUの仕様表。各命令の内容、オペコード、サイクル数、更新されるフラグの内容が一覧表示で確認でき、JSONも提供されています。CPU命令はここを見ながら実装しました。
- [Rustで作るGAME BOYエミュレータ：低レイヤ技術部](https://techbookfest.org/product/sBn8hcABDYBMeZxGvpWapf?productVariantID=2q95kwuw4iuRAkJea4BnKT)
  - Rustでゲームボーイのエミュレータ実装する本。章ごとに目標が設定されており、段階的に進めることができます。各章の説明もかなり詳しく、Rust以外で実装する場合でもおすすめです。特にPPUやAPUの実装でお世話になりました。

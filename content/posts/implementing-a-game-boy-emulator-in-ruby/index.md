+++
title = 'Implementing a Game Boy emulator in Ruby'
date = 2024-10-12T15:46:14+09:00
draft = false
+++

## Introduction
I created a Game Boy emulator in Ruby and released it as a gem called rubyboy!
(I'd be happy if you could give it a star!)

{{< github-repo-card "https://github.com/sacckey/rubyboy" >}}

|![](https://raw.githubusercontent.com/sacckey/rubyboy/main/resource/screenshots/pokemon.png)|![](https://raw.githubusercontent.com/sacckey/rubyboy/main/resource/screenshots/puyopuyo.png)|
|---|---|

<blockquote data-align="center" class="twitter-tweet" data-media-max-width="560"><p lang="ja" dir="ltr">Rubyでゲームボーイのエミュレータを作りました！<br>カラー対応やWasmでブラウザ対応もやっていきたい💪<br><br>GitHub: <a href="https://t.co/hFwmZD6FNp">https://t.co/hFwmZD6FNp</a> <a href="https://t.co/qWbx8v4mef">pic.twitter.com/qWbx8v4mef</a></p>&mdash; sacckey (@sacckey) <a href="https://twitter.com/sacckey/status/1769334370102587629?ref_src=twsrc%5Etfw">March 17, 2024</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

## This Article
While explaining the implementation process of Ruby Boy, I'll introduce the points where I got stuck and the techniques I devised.
I'll also introduce what I did to speed up Ruby Boy.

## Why I Created a Game Boy Emulator
- I wanted to do some personal development, but since web services incur maintenance costs, I wanted to create something that could be maintained for free
- As I use Ruby for work, I had been wanting to create a Ruby gem for a while
- Developing a game emulator has "clear goals & is fun when it works", so it seemed easier to maintain motivation
  - In particular, I have a special attachment to the Game Boy

→ Let's create a Game Boy emulator in Ruby and release it as a gem!

## Emulator Overview
The following is the architecture of the Game Boy:
<p align="center">
  <img src="https://storage.googleapis.com/zenn-user-upload/2a6af06959a6-20240408.png" width="500">
  <em>"Game Boy / Color Architecture - A Practical Analysis" by Rodrigo Copetti, Published: February 21, 2019, Last Modified: January 9, 2024. Available at: https://www.copetti.org/writings/consoles/game-boy/. Licensed under Creative Commons Attribution 4.0 International License.</em>
</p>

The goal is to implement a program that emulates this hardware.
The class diagram of Ruby Boy and the roles of each class are as follows:

![](https://storage.googleapis.com/zenn-user-upload/c11260f6c5a5-20240420.jpeg)

- Console: Main class
- Lcd: Handles screen rendering
- Bus: Controller for implementing memory-mapped I/O. Mediates the reading and writing of configuration values from the CPU to various hardware
- Cpu: Reads instructions from ROM, interprets and executes them
- Registers: Performs reading and writing of registers
- Cartridge: Performs reading and writing of ROM and RAM in the cartridge. Implementation differs for each type of MBC chip (explained later)
- Apu: Generates audio data
- Rom: Loads the game program from the cartridge
- Ram: Performs reading and writing of RAM data in the cartridge and Game Boy
- Interrupt: Manages interrupts. Interrupts are performed from the following three classes
- Timer: Counts the number of cycles
- Ppu: Generates pixel information to be rendered on the display
- Joypad: Receives button inputs from the Game Boy

In Ruby Boy, synchronization between components is achieved by having the CPU execute instructions and advancing the cycle count of Ppu, Timer, and Apu by the number of cycles taken for execution.
Therefore, the contents of the main loop are as follows:
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

## Implementation Process
I have compiled my implementation notes in a [scrap](https://zenn.dev/sacckey/scraps/380c2f3ad3318d). I will explain by extracting from this.

### UI Implementation
The UI part that handles screen rendering, audio playback, and keyboard input was implemented using SDL2 via the [Ruby-FFI gem](https://github.com/ffi/ffi).
I created a [wrapper class](https://github.com/sacckey/rubyboy/blob/main/lib/rubyboy/sdl.rb) that aggregates the necessary SDL2 methods, and the design is to call SDL2 methods from there.

### ROM Loading
First, I made it possible to load and use the game data.
For example, the title is stored at 0x0134~0x0143, so it can be retrieved as follows:

```ruby
data = File.open('tobu.gb', 'r') { _1.read.bytes }
p data[0x134..0x143].pack('C*').strip
=> "TOBU"
```

### Implementation of MBC (Memory Bank Controller)
Many Game Boy games use MBC (Memory Bank Controller), which achieves address space expansion through bank switching.
There are different types of MBC chips such as MBC1, MBC3, MBC5, etc., each with different sizes of usable ROM and RAM, so implementation specific to the type of MBC chip is necessary.
Ruby Boy supports NoMBC (without MBC) and MBC1 games, and I used the Factory pattern to return the appropriate MBC implementation. By adding chip implementations and case handling, it's possible to support other types of MBC chips as well.

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

### CPU Implementation
I implemented a program that repeats the following CPU execution cycle:
- Fetch instruction from ROM
- Decode the instruction
- Execute the instruction

To maintain motivation, instead of implementing all CPU and PPU processes at once, I first aimed to run the following minimal test ROM:

{{< github-repo-card "https://github.com/dusterherz/gb-hello-world" >}}

For debugging, I used a Game Boy emulator called [BGB](https://bgb.bircd.org/). It's useful because you can execute step by step while displaying register and memory information, allowing you to compare the behavior with your own CPU.

I checked the instructions executed in BGB and implemented them.
For the PPU, creating just the bg rendering process is enough to make the test run.
<p align="center">
 <img src="https://storage.googleapis.com/zenn-user-upload/a05ad4a1c905-20240330.png" width="500">
</p>

Next, I implemented all instructions and interrupt handling, aiming to pass the tests of CPU test ROMs called [cpu_instrs](https://github.com/retrio/gb-test-roms/tree/master/cpu_instrs/) and [instr_timing](https://github.com/retrio/gb-test-roms/tree/master/instr_timing).

#### Points Where I Got Stuck
- When setting the value of f in pop af, I hadn't set the lower 4 bits to 0000
 - The lower 4 bits of the f register are always 0000
- The calculation method for the c flag in instructions with opcode=0xe8, 0xf8 was incorrect
 - It passed with cflag = (@sp & 0xff) + (byte & 0xff) > 0xff

By fixing these, it successfully passed.
<p align="center">
 <img src="https://storage.googleapis.com/zenn-user-upload/4c3b6b917e15-20240330.png" width="500">
</p>

...The rendering looks a bit off, but the CPU processing is OK.

### PPU
I aimed to implement the remaining rendering processes and pass [dmg-acid2](https://github.com/mattcurrie/dmg-acid2), which is a test ROM for PPU.

The test passes when window and sprite rendering, interrupt handling, and DMA transfer are implemented.
Care must be taken with the priority of sprite display.
<p align="center">
 <img src="https://storage.googleapis.com/zenn-user-upload/92f2cf1d5e93-20240414.png" width="500">
</p>

Now that rendering is possible, implementing the Joypad will allow us to run games.
The following is a video of running a game called [Tobu Tobu Girl](https://tangramgames.dk/tobutobugirl/):

<blockquote data-align="center" class="twitter-tweet" data-media-max-width="560"><p lang="ja" dir="ltr">Tobu Tobu Girl動いた！<br>30fpsぐらいしか出ていないので、最適化する <a href="https://t.co/szPtv3F37R">pic.twitter.com/szPtv3F37R</a></p>&mdash; sacckey (@sacckey) <a href="https://twitter.com/sacckey/status/1727284351371796791?ref_src=twsrc%5Etfw">November 22, 2023</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

At this point, the game became operational! However, it's extremely slow.
From here on, I worked on optimization.

## Optimization
I'll introduce what I did to speed up Ruby Boy.
These techniques are not limited to emulator implementation and can be applied to optimizing Ruby programs in general.

Execution environment:
- PC: MacBook Pro (13-inch, 2018)
- Processor: 2.3 GHz Quad-Core Intel Core i5
- Memory: 16 GB 2133 MHz LPDDR3

### Benchmarking
I measured the time it took to execute the first 1500 frames of Tobu Tobu Girl without audio and rendering, repeating the measurement three times.
Since benchmarking will be done repeatedly, I recommend preparing a dedicated program and setting up a system where you can start benchmarking immediately with a command execution.

### Profiler
I used a gem called Stackprof.

{{< github-repo-card "https://github.com/tmm1/stackprof" >}}

It can be used simply by enclosing the area you want to measure in a block, and it's recommended because it has low overhead.

### Optimization Part 1
#### Enabling YJIT
By enabling YJIT, Ruby's JIT compiler, the FPS improved.
YJIT has become practical from Ruby 3.2, and can be enabled by adding the `--yjit` option at runtime.

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

#### Avoid Creating a Hash for Sprites Every Time
When running Stackprof, we can see that the render_sprites method is becoming a bottleneck.

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

Let's investigate further to see which part within render_sprites is the bottleneck.

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

Line 226's block occupies a high percentage of execution time. Upon closer inspection, it creates a Hash called sprite, and then adds sprite to an array called sprites if it meets certain conditions.
By modifying this to create sprite only when the conditions are met, the speed improved.

FPS: 46.73385499531633 → 49.2233733053377

In this way, we continue to diligently identify and fix bottlenecks.

#### Refactoring PPU
This is a basic principle of "do outside the loop what can be done outside the loop",
but it's critically important in emulators, and resolving this dramatically improves performance.

##### Calculate tile_map_addr outside the loop
[commit](https://github.com/sacckey/rubyboy/commit/08273596838631334ee83fbd540feab5da09233e)
FPS: 49.2233733053377 → 56.6580741129914

##### Calculate tile_index outside the loop
[commit](https://github.com/sacckey/rubyboy/commit/3e79628cf69f15bc690049f1695e3767ae48e8a9)
FPS: 56.6580741129914 → 60.44140113483162

#### Ruby v3.2 -> v3.3
At this point, we achieved about 60 FPS without rendering, but we hit a wall.
There's a trade-off between optimization and code readability. For example, abandoning the use of constants and directly writing mysterious integers would make it faster, but that's not desirable.

While pondering this, Ruby 3.3.0 was released on 2023/12/25.
Ruby 3.3's YJIT was [reported to be even faster](https://k0kubun.hatenablog.com/entry/ruby-3-3-yjit), so I tried updating...

![](https://storage.googleapis.com/zenn-user-upload/0809e3724053-20240330.png)

!?

It got incredibly fast!!!!
Ruby 3.3 was faster than I imagined. Thank you so much 🙏
By the way, this [comparison post](https://twitter.com/sacckey/status/1740342734857306595) was even reposted by Matz. I'm thrilled.

### Reducing GC
Thanks to Ruby 3.3, we achieved higher speed, but in exchange? the number of GC occurrences increased significantly.

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

Also, the operation from the title screen to the professor's dialogue scene in Pokemon Red was still heavy, so resolving these issues became the next goal.

#### GC Profiler
I used HeapProfiler to detect GC occurrence locations.

{{< github-repo-card "https://github.com/Shopify/heap-profiler" >}}

This can also be easily used by enclosing the area you want to detect in a block, but be careful as it may stop returning detection results when running for a long time.

Execution results (partial)

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

Looking at this, we can see that creating a large number of Hashes within the Cpu class is the cause of GC occurrences.

#### Reduce Hash generation within the Cpu class
##### Change instruction arguments from Hash to Symbol
[commit](https://github.com/sacckey/rubyboy/commit/7cc25746dffcc91e9f6ad4162ec279e8547b9fe0)

```diff ruby
case opcode
-  when 0x01 then ld16({ type: :register16, value: :bc }, { type: :immediate16 }, cycles: 12)
+  when 0x01 then ld16(:bc, :immediate16, cycles: 12)
```

##### Avoid creating Hash when referencing flags
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

With these modifications, we were able to reduce GC occurrences to 2.71%.

### Optimization Part 2
#### Reducing Integer#<=>
At this point, the benchmark and Stackprof results are as follows:
It's important to note that this result was measured with rendering enabled and at the heaviest part of Pokemon Red, so comparison with previous results is not meaningful.

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

While GC has been reduced, we're not achieving 60 FPS, and we can see that `Integer#<=>` (number comparison) has become a bottleneck.
Number comparisons occur frequently in address-based branching like the following:
```ruby
def read_byte(addr)
  case addr
  when 0x0000..0x7fff
    @mbc.read_byte(addr)
  when 0x8000..0x9fff
    @ppu.read_byte(addr)
...
```

To eliminate these comparisons, I created an array called `@read_methods` in preprocessing, which contains the correspondence between addresses and processes. This allows for calling with just array reference during execution.
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

This technique was inspired by [Optcarrot](https://github.com/mame/optcarrot), a NES emulator written in Ruby.

Let's run the benchmark and Stackprof again.

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

The FPS improved from 55.97272441676141 to 68.95255099156066, and the proportion of `Integer#<=>` was reduced from 21.8% to 0.5%.
There's still room for further optimization, but since we've achieved our goal, I'm considering this complete for now.

### Optimization Results

|Before(rubyboy v1.0.0, Ruby 3.2.2)|After(rubyboy v1.3.1, Ruby 3.3.0 + YJIT)|
|---|---|
|![](https://storage.googleapis.com/zenn-user-upload/b29a44007817-20240416.gif)|![](https://storage.googleapis.com/zenn-user-upload/6dff7cf43943-20240416.gif)|


## Conclusion
### Positive Aspects
#### Emulator Development is Fun
As initially planned, I was able to implement while having fun. While it's enjoyable when an emulator runs, I think a major factor is the abundance of documentation and test ROMs. Especially, since the test ROMs provide feedback on incorrect parts, I was able to progress without getting stuck and could refactor easily.
Also, I'm happy that I could run the cartridges that were lying dormant at my parents' home.

#### Published a Ruby Gem
I've been writing Ruby for a while, and I'm glad I could finally publish a properly functioning gem.
https://rubygems.org/gems/rubyboy

Install it now with `gem install rubyboy`!

#### Learned About Low-Level Technology
Through implementing programs that mimic CPU, memory, registers, RAM, etc., I was able to deepen my knowledge about their roles and operations. It was enjoyable to see things I knew theoretically actually come up, leading to "so that's what it meant" discoveries. How about using this for experimental subjects in universities or technical colleges?

#### Gained Experience in Program Optimization
I experienced the steady process of "creating a benchmark program, running a profiler, and fixing suspicious areas" for optimization. I believe my resolution for optimization has improved and my sense for detecting bottlenecks has sharpened.

#### Experienced Designing and Implementing a Larger Program
I hadn't written many large programs outside of web programming, so it was good to gain that experience. With emulators, I felt it was important to appropriately distribute responsibilities among classes and write generalized programs to increase reusability (especially for the CPU).

### Impressions of Ruby
- Happy with the simple syntax!
- Delighted by the many thoughtful methods!
- Pleased with the abundance of useful gems!
- Frustrated by the slow processing speed!
   - Glad that it's significantly faster than before thanks to YJIT's evolution!

### Future Plans
I'm planning to work on the following:
- Fixing rendering bugs
- Adding more MBC types
- Supporting Game Boy Color
- WebAssembly support
- Improving the benchmark system
 - Want to make it usable as a benchmark program for Ruby

## References
### Self-Made Blog Posts
These are articles about creating Game Boy emulators. I referred to them for implementation approaches, techniques, and potential pitfalls 🙏

- [C++でゲームボーイエミュレータを自作しています | voidProc | ゲーム製作ログ](https://voidproc.com/blog/archives/664)
- [Rustでゲームボーイエミュレータを自作した話 - MJHD](https://mjhd.hatenablog.com/entry/2021/04/14/221813)
- [ゲームボーイのエミュレータを自作した話 · Keichi Takahashi](https://keichi.dev/post/write-yourself-a-game-boy-emulator/)
- [AQBoy: Yet Another Game Boy Emulator 開発記 - HackMD](https://hackmd.io/@anqou/HJcvRrwy9)
- [OCaml でゲームボーイエミュレータを書いた話 #関数型言語 - Qiita](https://qiita.com/linoscope/items/244d931aaae07df2c27e)

### Presentation Slides
- [Ruby で高速なプログラムを書く | PPT](https://www.slideshare.net/mametter/ruby-65182128)
  - Presentation slides by the creator of [Optcarrot](https://github.com/mame/optcarrot). These slides are packed with essential information about optimizing Ruby programs. I was able to achieve the optimization of Ruby Boy by referring to these slides.

### Documentation
- [Pan Docs](https://gbdev.io/pandocs/)
  - A page that covers Game Boy specifications comprehensively, used like a dictionary.
- [Game Boy CPU (SM83) instruction set](https://gbdev.io/gb-opcodes/optables/)
  - CPU specification table. It displays a list of each instruction's content, opcode, cycle count, and updated flags, and also provides JSON. I implemented CPU instructions while referring to this.
- [Rustで作るGAME BOYエミュレータ：低レイヤ技術部](https://techbookfest.org/product/sBn8hcABDYBMeZxGvpWapf?productVariantID=2q95kwuw4iuRAkJea4BnKT)
  - A book about implementing a Game Boy emulator in Rust. It sets goals for each chapter, allowing for step-by-step progress. The explanations for each chapter are quite detailed, and it's recommended even if you're implementing in a language other than Rust. It was especially helpful for implementing PPU and APU.
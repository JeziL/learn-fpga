# 从闪灯器到 RISC-V

本教程将带你循序渐进地从一个简单的 LED 闪灯器设计走向 RISC-V 内核。

本教程支持以下开发板：
- IceStick
- IceBreaker
- ULX3S
- ARTY
- tang nano 9K

如果你手头没有开发板，也可以在仿真环境中运行全部内容（只是乐趣会少一些）。

## 关于本教程

- 这是一份渐进式入门教程，每次只改变一个要素。它整理自我在 2020—2022 年学习这些概念时记录的日志。我也尽量保留了探索过的死胡同和踩过的坑，通常会以题外话或注释的形式标出；
- 我尽量把硬件要求降到最低。即便使用最小的 FPGA（IceStick Ice40HX1K），你也能完成本教程的第一篇，并最终把它变成一颗功能完整、能够执行编译后 C 代码的 RV32I 微控制器；
- 最终得到的处理器并非效率最高，但绝不是玩具：它可以执行任何程序。你可能正想问——没错，它能[运行 DOOM](https://github.com/BrunoLevy/learn-fpga/tree/master/LiteX/software/Doom)！（不过 IceStick 不行，你需要更大的 FPGA。）这是借助 LiteX 实现的；LiteX 提供了很不错的 SDRAM 控制器，而 Doom 需要一定容量的内存；
- 本教程同时涉及硬件和软件：你将学会如何为自己的内核编译汇编程序和 C 程序；
- 我尽量让所有示例程序既有趣又好玩，同时保持篇幅适中。附带的演示程序包括：
    - 用汇编和 C 实现曼德博集合
    - 旋转缩放（rotozoom）图形效果
    - 绘制填充多边形
    - 光线追踪
  这些图形程序都使用 ANSI 转义序列，以文本模式显示在终端中（没错，因此像素会非常大）。如果想更有趣，也可以改用一块小型 OLED 显示屏（以后会补充相关说明）；
- [第二篇](PIPELINE.md)讨论流水线。你将学习如何把本教程末尾得到的基础处理器改造成一颗更高效、带分支预测的流水线处理器；
- [第三篇](INTERRUPTS.md)仍在编写中，主题是中断和 RISC-V 特权级 ISA；
- 本教程使用 VERILOG 编写，目前正移植到其他 HDL：
    - @bl0x 编写的 [Amaranth/nMigen 版本](https://github.com/bl0x/learn-fpga-amaranth)
    - TODO：Silice 版本
    - TODO：SpinalHDL 版本

## 处理器设计简介与参考资料

为了理解处理器设计，我最先阅读的是 Stack Overflow 上的[这个回答](https://stackoverflow.com/questions/51592244/implementation-of-simple-microprocessor-using-verilog/51621153#51621153)，它给了我很多启发。@mithro 还推荐了[这篇文章](http://www.fpgacpu.org/papers/xsoc-series-drafts.pdf)。如果想系统学习一门完整课程，我强烈推荐 [MIT 的这门课](http://web.mit.edu/6.111/www/f2016/)；它还介绍了如何远远超越我在这里完成的内容（例如流水线等）。

关于 Verilog 基础和语法，我读的是 Blaine C. Readler 的 _Verilog by Example_，同样简洁明了、直奔主题。

上面那个 Stack Overflow 回答有两个优点：
- 直击本质，只保留必不可少的内容；
- 使用 RISC 处理器作为示例，它与 RISC-V 有不少相似之处（区别之一是它有状态标志，而 RISC-V 没有）。

我们从中了解到，处理器需要一个_寄存器堆_（register file），用于存放所谓的_通用_寄存器。“通用”是指：指令每次读取寄存器时，可以读取其中任意一个；每次写入寄存器时，也可以写入其中任意一个。这与 x86（CISC）的_专用_寄存器不同。为了实现最通用的指令（`寄存器 <- 寄存器 OP 寄存器`），寄存器堆每个周期要读取两个寄存器，并可选择回写一个寄存器。

处理器还需要一个 _ALU_（算术逻辑单元），用于对两个值执行运算。

此外还需要一个_译码器_，根据当前指令的位模式生成所需的全部内部信号。

如果你想从零自行设计 RISC-V 处理器，我建议认真研读[这个 Stack Overflow 回答](https://stackoverflow.com/questions/51592244/implementation-of-simple-microprocessor-using-verilog/51621153#51621153)，并亲手画几张原理图，在继续之前先把总体思路理清……当然，你也可以直接进入本教程，一步一步来。它会带你从最简单的闪灯器设计平稳过渡到一颗功能完整的 RISC-V 内核。

## 准备工作

第一步是克隆 learn-fpga 仓库：
```
$ git clone https://github.com/BrunoLevy/learn-fpga.git
```

开始前，需要安装以下软件：
- iverilog/icarus（仿真）
```
  $ sudo apt-get install iverilog
```
- yosys/nextpnr，即适用于你的开发板的工具链。请参阅[此链接](../toolchain.md)。

请注意，仅用 iverilog/icarus 就足以运行和尝试本教程的所有步骤，但体验并不相同。我强烈建议在真实设备上完成每一步。当你亲手设计的处理器第一次真正运行程序时，那份感受和兴奋绝非仿真所能比拟！！！

## 第 1 步：第一个闪灯器

让我们从创建第一个闪灯器开始！闪灯器实现为一个 VERILOG 模块，并按如下方式连接输入和输出（[step1.v](step1.v)）：
```verilog
   module SOC (
       input  CLK,
       input  RESET,
       output [4:0] LEDS,
       input  RXD,
       output TXD
   );

   reg [4:0] count = 0;
   always @(posedge CLK) begin
      count <= count + 1;
   end
   assign LEDS = count;
   assign TXD  = 1'b0; // not used for now

   endmodule
```
我们称它为 SOC（System On Chip，片上系统）。对于一个闪灯器而言，这个名字似乎有点夸张，不过完成本教程的全部步骤后，它确实会蜕变成 SOC。我们的 SOC 连接到以下信号：

- `CLK`（输入）是系统时钟；
- `LEDS`（输出）连接开发板上的 5 个 LED；
- `RESET`（输入）是复位按钮。你可能会说 IceStick 根本没有按钮，但其实……（稍后再谈）；
- `RXD` 和 `TXD`（分别为输入、输出）连接到 FTDI 芯片，后者通过 USB 模拟串口。这个也会在后面介绍。

可以使用以下命令综合设计，并把位流发送到设备：
```
$ BOARDS/run_xxx.sh step1.v
```
其中 `xxx` 对应你所使用的开发板。

五个 LED 都会亮起……但看起来并没有闪烁。这是为什么？其实它们确实在闪，只是速度太快，肉眼无法分辨。

为了看清变化，可以使用仿真。我们新建一个 VERILOG 文件 [bench_iverilog.v](bench_iverilog.v)，其中包含一个封装了 `SOC` 的 `bench` 模块：
```verilog
module bench();
   reg CLK;
   wire RESET = 0;
   wire [4:0] LEDS;
   reg  RXD = 1'b0;
   wire TXD;

   SOC uut(
     .CLK(CLK),
     .RESET(RESET),
     .LEDS(LEDS),
     .RXD(RXD),
     .TXD(TXD)
   );

   reg[4:0] prev_LEDS = 0;
   initial begin
      CLK = 0;
      forever begin
	 #1 CLK = ~CLK;
	 if(LEDS != prev_LEDS) begin
	    $display("LEDS = %b",LEDS);
	 end
	 prev_LEDS <= LEDS;
      end
   end
endmodule
```
`bench` 模块驱动 `SOC` 的所有信号（这里将 `SOC` 命名为 `uut`，即“unit under test”，受测单元）。`forever` 循环不断翻转 `CLK` 信号，并在 LED 状态发生变化时将其显示出来。

现在可以开始仿真：
```
  $ iverilog -DBENCH -DBOARD_FREQ=10 bench_iverilog.v step1.v
  $ vvp a.out
```
……不过要记的内容有点多，所以我为此编写了一个脚本。你大概会更喜欢这样运行：
```
  $ ./run.sh step1.v
```

你会看到 LED 不断计数。仿真非常宝贵，因为它允许你在 VERILOG 代码中插入“打印”语句（`$display`）；直接在设备上运行时无法这样做！

退出仿真：
```
  <ctrl><c>
  finish
```
_注：我在开发 femtorv 的第一个版本时完全是在真实设备上进行的。因为当时不会使用仿真，只能靠 LED 调试。千万别学我，这样做太蠢了！_

**动手试试**：应该如何修改 `step1.v`，把速度降到足以让人看清 LED 闪烁？

**动手试试**：能否实现一种类似《霹雳游侠》（Knight Rider）扫描灯的闪烁模式，而不是简单计数？

## 第 2 步：更慢的闪灯器

你大概已经想到了：要让闪灯器慢下来，可以让计数器使用更多位（并把最高几位连接到 LED），也可以插入一个“分频器”（clock divider，也叫“变速箱” gearbox）。分频器使用较多位进行计数，再用最高位驱动计数器。第二种方案很有意思，因为不必修改原设计，只需把分频器插在开发板的 `CLK` 信号和设计之间。这样即便在真实设备上，也能看清 LED 的变化。

为此，我在 [clockworks.v](clockworks.v) 中创建了一个 `Clockworks` 模块。它包含变速箱，以及一套与 `RESET` 信号有关的机制（稍后介绍）。`Clockworks` 的实现如下：
```verilog
module Clockworks
(
   input  CLK,   // clock pin of the board
   input  RESET, // reset pin of the board
   output clk,   // (optionally divided) clock for the design.
   output resetn // (optionally timed) negative reset for the design (more on this later)
);
   parameter SLOW;
...
   reg [SLOW:0] slow_CLK = 0;
   always @(posedge CLK) begin
      slow_CLK <= slow_CLK + 1;
   end
   assign clk = slow_CLK[SLOW];
...
endmodule
```
这样会把时钟频率除以 `2^SLOW`。

接下来，在 [step2.v](step2.v) 中把 `Clockworks` 模块插到开发板的 `CLK` 信号与设计之间，并使用内部 `clk` 信号：

```verilog
`include "clockworks.v"

module SOC (
    input  CLK,        // system clock
    input  RESET,      // reset button
    output [4:0] LEDS, // system LEDs
    input  RXD,        // UART receive
    output TXD         // UART transmit
);

   wire clk;    // internal clock
   wire resetn; // internal reset signal, goes low on reset

   // A blinker that counts on 5 bits, wired to the 5 LEDs
   reg [4:0] count = 0;
   always @(posedge clk) begin
      count <= !resetn ? 0 : count + 1;
   end

   // Clock gearbox (to let you see what happens)
   // and reset circuitry (to workaround an
   // initialization problem with Ice40)
   Clockworks #(
     .SLOW(21) // Divide clock frequency by 2^21
   )CW(
     .CLK(CLK),
     .RESET(RESET),
     .clk(clk),
     .resetn(resetn)
   );

   assign LEDS = count;
   assign TXD  = 1'b0; // not used for now
endmodule
```
它还会处理 `RESET` 信号。

现在可以在仿真中试一试：
```
  $ ./run.sh step2.v
```

可以看到，计数器现在慢多了。再到真实设备上试试：
```
  $ BOARDS/run_xxx.sh step2.v
```
没错，现在可以清楚地看到发生了什么！那么 `RESET` 按钮呢？IceStick 不是没有按钮吗？其实它有！

![](IceStick_RESET.jpg)

用手指按住图中圈出的区域（47 号引脚附近）。

**动手试试**：实现《霹雳游侠》扫描灯模式，并用 `RESET` 切换移动方向。

查看 [clockworks.v](clockworks.v) 可以发现，它还能创建 `PLL`。PLL 是一种可以生成_更高频率_时钟的组件。例如，IceStick 的系统时钟为 12 MHz，而我们将要生成的内核却能以 45 MHz 运行。稍后会介绍这一点。

## 第 3 步：从 ROM 加载 LED 图案的闪灯器

现在所需工具已经齐备，让我们看看如何把这个闪灯器变成一颗功能完整的 RISC-V 处理器。这个目标似乎还遥不可及，但到第 16 步时，我们做出的处理器还不到 200 行 VERILOG！发现创建处理器竟然这么简单时，我非常惊讶。好了，还是一步一步来吧。

我们已经知道，处理器拥有存储器，并且大多数时候会从中依次取出指令（跳转和分支除外）。让我们先做一个类似但简单得多的东西：一串预先编程的圣诞彩灯，从存储器中加载 LED 图案（参见 [step3.v](step3.v)）。彩灯的存储器中保存着这些图案：
```verilog
   reg [4:0] MEM [0:20];
   initial begin
       MEM[0]  = 5'b00000;
       MEM[1]  = 5'b00001;
       MEM[2]  = 5'b00010;
       MEM[3]  = 5'b00100;
       ...
       MEM[19] = 5'b10000;
       MEM[20] = 5'b00000;
   end
```
_请注意，`initial` 块中的内容在综合时不会生成任何电路，而会直接转换成 FPGA 块 RAM（BRAM）的初始化数据。_

我们还要有一个“程序计数器” `PC`，它在每个时钟周期递增；另外还要有一套以 `PC` 为索引、读取 `MEM` 内容的机制：

```verilog
   reg [4:0] PC = 0;
   reg [4:0] leds = 0;

   always @(posedge clk) begin
      leds <= MEM[PC];
      PC <= (!resetn || PC==20) ? 0 : (PC+1);
   end
```
_请注意，为了让它循环，这里测试了 `PC==20`。_

现在分别在仿真和真实设备上试一试。

**动手试试**：创建几种不同的闪烁模式，并使用 `RESET` 在模式之间切换。

## RISC-V 指令集架构

一个重要的信息来源当然是 [RISC-V 参考手册](https://github.com/riscv/riscv-isa-manual/releases/download/Ratified-IMAFDQC/riscv-spec-20191213.pdf)。从中可以了解到，RISC-V 标准有多种配置。我们从最简单的 RV32I 开始，也就是 32 位基础整数指令集，然后每次增加一种功能。这正是 RISC-V 的一大优点：指令集是_模块化_的，可以从一个非常小且自成体系的内核起步，而这个内核仍然符合标准。这意味着标准工具（编译器、汇编器和链接器）都能为它生成代码。

随后我阅读了第 2 章（第 13—30 页）。再结合第 130 页的表格来看，实际上只有 11 种不同的指令！（例如，我把 AND、OR、ADD 等视为同一种指令，具体运算只是它的附加参数。）现在只需先把握全局，不必钻进细节。让我们概览这 11 种指令：

| 指令 | 说明 | 算法 |
|------|------|------|
| 分支 | 条件跳转，6 种变体 | `if(reg OP reg) PC<-PC+imm` |
| 寄存器 ALU | 三寄存器 ALU 运算，10 种变体 | `reg <- reg OP reg` |
| 立即数 ALU | 双寄存器 ALU 运算，9 种变体 | `reg <- reg OP imm` |
| 加载 | 存储器到寄存器，5 种变体 | `reg <- mem[reg + imm]` |
| 存储 | 寄存器到存储器，3 种变体 | `mem[reg+imm] <- reg` |
| `LUI` | 加载高位立即数 | `reg <- (im << 12)` |
| `AUIPC` | 将高位立即数加到 PC | `reg <- PC+(im << 12)` |
| `JAL` | 跳转并链接 | `reg <- PC+4 ; PC <- PC+imm` |
| `JALR` | 寄存器间接跳转并链接 | `reg <- PC+4 ; PC <- reg+imm` |
| `FENCE` | 多核存储器顺序控制 | （此处不详述，暂时跳过） |
| `SYSTEM` | 系统调用、断点 | （此处不详述，暂时跳过） |

- 6 种分支变体都是条件跳转，具体是否跳转取决于对两个寄存器的测试；

- ALU 运算可以采用 `寄存器 <- 寄存器 OP 寄存器` 或 `寄存器 <- 寄存器 OP 立即数` 的形式；

- 接下来是 load 和 store，它们可以操作字节、16 位值（称为半字，half-word）或 32 位值（称为字，word）。此外，加载字节和半字时还可以进行符号扩展。源地址或目标地址由寄存器内容加上立即数偏移量得到；

- 剩下的指令更加特殊（第一次阅读时可以跳过详细说明，只需知道它们用于实现无条件跳转、函数调用、多核存储器顺序控制、系统调用和断点）：

    - `LUI`（load upper immediate，加载高位立即数）用于加载常数的高 20 位，随后可用 `ADDI` 或 `ORI` 设置低位。乍看之下，用两条指令才能把一个 32 位常数载入寄存器似乎有些奇怪；但由于所有指令本身都只有 32 位长，这其实是一种聪明的选择；

    - `AUIPC`（add upper immediate to PC，将高位立即数加到 PC）把常数加到当前程序计数器上，并将结果放入寄存器。它要与 `JALR` 配合使用，以访问相对于 PC 的 32 位地址；

    - `JAL`（jump and link，跳转并链接）把偏移量加到 PC，并把跳转之后那条指令的地址存入寄存器。它可以用来实现函数调用。`JALR` 的作用类似，只不过偏移量加在某个寄存器上；

    - `FENCE` 和 `SYSTEM` 分别用于实现多核系统中的存储器顺序控制，以及系统调用/断点。

总结一下，我们有分支（条件跳转）、ALU 运算、加载和存储，还有几条用于实现无条件跳转和函数调用的特殊指令。另外还有用于存储器顺序控制和系统调用的两类指令（暂时忽略）。好吧，实际上只剩 9 种指令，看起来可以做到……此时我还没有完全理解所有内容，于是决定先从我认为最简单的部分入手（指令译码器、寄存器堆和 ALU），然后再研究它们如何互连，以及怎样实现跳转、分支和全部指令。

## 第 4 步：指令译码器

现在的思路是：准备一块装有 RISC-V 指令的存储器，像读取圣诞彩灯图案那样，依次把所有指令载入 `instr` 寄存器，再尝试识别这 11 种指令（根据识别出的指令点亮不同的 LED）。每条指令编码在一个 32 位字中，我们需要对其中的各个位进行译码，才能识别指令及其参数。

[RISC-V 参考手册](https://github.com/riscv/riscv-isa-manual/releases/download/Ratified-IMAFDQC/riscv-spec-20191213.pdf)第 130 页的两张表（RV32/64G Instruction Set Listings）汇总了所需的全部信息。

先看较大的那张表。首先要注意，最低 7 位（LSB）指明了具体是哪种指令（共有 10 种可能；暂时不计 `FENCE`）。

```verilog
   reg [31:0] instr;
   ...
   wire isALUreg  =  (instr[6:0] == 7'b0110011); // rd <- rs1 OP rs2
   wire isALUimm  =  (instr[6:0] == 7'b0010011); // rd <- rs1 OP Iimm
   wire isBranch  =  (instr[6:0] == 7'b1100011); // if(rs1 OP rs2) PC<-PC+Bimm
   wire isJALR    =  (instr[6:0] == 7'b1100111); // rd <- PC+4; PC<-rs1+Iimm
   wire isJAL     =  (instr[6:0] == 7'b1101111); // rd <- PC+4; PC<-PC+Jimm
   wire isAUIPC   =  (instr[6:0] == 7'b0010111); // rd <- PC + Uimm
   wire isLUI     =  (instr[6:0] == 7'b0110111); // rd <- Uimm
   wire isLoad    =  (instr[6:0] == 7'b0000011); // rd <- mem[rs1+Iimm]
   wire isStore   =  (instr[6:0] == 7'b0100011); // mem[rs1+Simm] <- rs2
   wire isSYSTEM  =  (instr[6:0] == 7'b1110011); // special
```

除了指令类型，还需要译码指令的操作数。上方的表格根据指令操作数及其在 32 位指令字中的编码方式，区分了 6 种指令格式（`R-type`、`I-type`、`S-type`、`B-type`、`U-type`、`J-type`）。

`R-type` 指令接收两个源寄存器 `rs1` 和 `rs2`，对它们执行运算，并把结果存入第三个目标寄存器 `rd`（`ADD`、`SUB`、`SLL`、`SLT`、`SLTU`、`XOR`、`SRL`、`SRA`、`OR`、`AND`）。

RISC-V 有 32 个寄存器，因此 `rs1`、`rs2` 和 `rd` 各占指令字的 5 位。有趣的是，在所有指令格式中，这些字段的位置都相同。所以“译码” `rs1`、`rs2` 和 `rd` 其实只需从指令字相应位置引出几根线：
```verilog
   wire [4:0] rs1Id = instr[19:15];
   wire [4:0] rs2Id = instr[24:20];
   wire [4:0] rdId  = instr[11:7];
```

接下来还要在 10 条 R-type 指令中进行区分。这主要依靠 `funct3` 字段，即一个 3 位编码。3 位只能编码 8 种不同指令，因此还需要 `funct7` 字段（指令字的最高 7 位）。指令字的第 30 位用于区分 `ADD`/`SUB` 和 `SRA`/`SRL`（带符号扩展的算术右移/逻辑右移）。指令译码器为 `funct3` 和 `funct7` 引出如下信号：
```verilog
   wire [2:0] funct3 = instr[14:12];
   wire [6:0] funct7 = instr[31:25];
```

`I-type` 指令接收一个寄存器 `rs1` 和一个立即数 `Iimm`，对二者执行运算，并把结果存入目标寄存器 `rd`（`ADDI`、`SLTI`、`SLTIU`、`XORI`、`ORI`、`ANDI`、`SLLI`、`SRLI`、`SRAI`）。

_等一下：_R-Type 指令有 10 条，I-Type 指令却只有 9 条，为什么？仔细观察会发现，这里没有 `SUBI`，因为可以改用带负立即数的 `ADDI`。这是 RISC-V 的一条通用原则：如果已有功能可以复用，就不要再创造新功能。

与 R-type 指令一样，可以使用 `funct3` 和 `funct7` 区分这些指令（`funct7` 中只用到指令字的第 30 位，用于区分算术右移 `SRAI` 和逻辑右移 `SRLI`）。

立即数编码在指令字的最高 12 位中，因此再引出一些线即可得到它：
```verilog
   wire [31:0] Iimm={{21{instr[31]}}, instr[30:20]};
```

可以看到，指令字的第 31 位重复了 21 次，这就是“符号扩展”（把一个 12 位有符号量转换成 32 位有符号量）。

另外还有四种指令格式：`S-type`（Store）、`B-type`（Branch）、`U-type`（左移 12 位的高位立即数）和 `J-type`（Jump）。每种指令格式在指令字中编码立即数的方式都不同。

要理解它的含义，请回到第 2 章第 16 页。不同指令类型的区别，就在于其中_立即数_的编码方式。

| 指令类型 | 说明 | 立即数编码 |
|----------|------|------------|
| `R-type` | 寄存器—寄存器 ALU 运算。[更多说明](https://www.youtube.com/watch?v=pVWtI0426mU) | 无 |
| `I-type` | 寄存器—立即数整数 ALU 运算和 `JALR` | 12 位，符号扩展 |
| `S-type` | 存储 | 12 位，符号扩展 |
| `B-type` | 分支 | 12 位，符号扩展，位于高位 `[31:1]`（第 0 位为 0） |
| `U-type` | `LUI`、`AUIPC` | 20 位，位于高位 `31:12`（`[11:0]` 位为 0） |
| `J-type` | `JAL` | 12 位，符号扩展，位于高位 `[31:1]`（第 0 位为 0） |

请注意，`I-type` 与 `S-type` 编码的是同一类值（只是从 `instr` 的不同位置取出）。`B-type` 与 `J-type` 也是如此。

不同类型的立即数可以按如下方式译码：
```verilog
   wire [31:0] Uimm={    instr[31],   instr[30:12], {12{1'b0}}};
   wire [31:0] Iimm={{21{instr[31]}}, instr[30:20]};
   wire [31:0] Simm={{21{instr[31]}}, instr[30:25],instr[11:7]};
   wire [31:0] Bimm={{20{instr[31]}}, instr[7],instr[30:25],instr[11:8],1'b0};
   wire [31:0] Jimm={{12{instr[31]}}, instr[19:12],instr[20],instr[30:21],1'b0};
```
请注意，`Iimm`、`Simm`、`Bimm` 和 `Jimm` 都进行了符号扩展（按所需次数复制第 31 位，以填满最高位）。

指令译码器到这里就完成了！总结一下，指令译码器从指令字中获取以下信息：
- 用于识别 11 种 RISC-V 指令的 `isXXX` 信号；
- 源寄存器和目标寄存器 `rs1`、`rs2`、`rd`；
- 功能码 `funct3` 和 `funct7`；
- 五种格式的立即数（其中 `Iimm`、`Simm`、`Bimm` 和 `Jimm` 带符号扩展）。

现在用几条 RISC-V 指令初始化存储器，再根据指令类型点亮不同的 LED，看看能否正确识别它们（[step4.v](step4.v)）。为此要使用 [RISC-V 参考手册](https://github.com/riscv/riscv-isa-manual/releases/download/Ratified-IMAFDQC/riscv-spec-20191213.pdf)第 130 页的大表格。这个过程稍显繁琐（后面会看到更简单的方法！）。在这种情况下，使用 `_` 字符分隔二进制常量中的各个字段格外方便。

```verilog
   initial begin
      // add x1, x0, x0
      //                    rs2   rs1  add  rd  ALUREG
      MEM[0] = 32'b0000000_00000_00000_000_00001_0110011;
      // addi x1, x1, 1
      //             imm         rs1  add  rd   ALUIMM
      MEM[1] = 32'b000000000001_00001_000_00001_0010011;
      ...
      // lw x2,0(x1)
      //             imm         rs1   w   rd   LOAD
      MEM[5] = 32'b000000000000_00001_010_00010_0000011;
      // sw x2,0(x1)
      //             imm   rs2   rs1   w   imm  STORE
      MEM[6] = 32'b000000_00001_00010_010_00000_0100011;
      // ebreak
      //                                        SYSTEM
      MEM[7] = 32'b000000000001_00000_000_00000_1110011;
   end
```

然后可以按如下方式取出并识别指令：
```verilog
   always @(posedge clk) begin
      if(!resetn) begin
	 PC <= 0;
      end else if(!isSYSTEM) begin
	 instr <= MEM[PC];
	 PC <= PC+1;
      end
   end
   assign LEDS = isSYSTEM ? 31 : {PC[0],isALUreg,isALUimm,isStore,isLoad};
```
（第一个 LED 连接到 `PC[0]`，这样即使连续出现多条相同指令，也能看到它闪烁。）

可以看到，只有当前指令不是 `SYSTEM` 时，程序计数器才会递增。目前我们支持的唯一 `SYSTEM` 指令是用于停止执行的 `EBREAK`。

在仿真模式下，还可以显示识别出的指令名称和各个字段：
```verilog
`ifdef BENCH
   always @(posedge clk) begin
      $display("PC=%0d",PC);
      case (1'b1)
	isALUreg: $display("ALUreg rd=%d rs1=%d rs2=%d funct3=%b",rdId, rs1Id, rs2Id, funct3);
	isALUimm: $display("ALUimm rd=%d rs1=%d imm=%0d funct3=%b",rdId, rs1Id, Iimm, funct3);
	isBranch: $display("BRANCH");
	isJAL:    $display("JAL");
	isJALR:   $display("JALR");
	isAUIPC:  $display("AUIPC");
	isLUI:    $display("LUI");
	isLoad:   $display("LOAD");
	isStore:  $display("STORE");
	isSYSTEM: $display("SYSTEM");
      endcase
   end
`endif
```

**动手试试**：分别在仿真和真实设备上运行 `step4.v`。尝试用不同的 RISC-V 指令初始化存储器，并测试译码器能否识别它们。

## 题外话：RISC-V 的优雅之处

本节可以跳过。它只是记录了我对 RISC-V 指令集的一些个人感受和思考，灵感来自 [RISC-V 参考手册](https://github.com/riscv/riscv-isa-manual/releases/download/Ratified-IMAFDQC/riscv-spec-20191213.pdf)中以斜体写出的评论与问答。

至此，我终于意识到_指令集架构_的含义：它当然是一份规定“什么位模式执行什么操作”的规范（指令集），同时也由这些内容如何转化为连线来驱动（架构）。ISA 并不_抽象_；它_独立于_具体实现，却在设计时强烈考虑了实现方式！不同实现的流水线、分支预测单元、多执行单元和缓存或许各不相同，但指令译码器很可能大同小异。

起初有很多东西让我觉得非常奇怪：五花八门的立即数格式、立即数散落在 `instr` 不同位中的编码方式、`zero` 寄存器，以及 `LUI`、`AUIPC`、`JAL`、`JALR` 这些奇怪指令。在动手编写指令译码器后，你会更明白其中的原因。这套 ISA 确实非常聪明，而且是长期演化的结果（此前有 RISC-I、RISC-II……）。在我看来，它像是一次_蒸馏_的产物。到 2020 年，ISA 领域已经尝试过许多方案，而 RISC-V 似乎吸取了以往尝试的经验，保留优秀选择并避开次优方案。

这套 ISA 真正出色的地方包括：
- 指令长度固定，让事情简单得多。_（虽然也有可变指令长度的扩展，但至少核心指令集很简单。）_
- `rs1`、`rs2`、`rd` 始终编码在 `instr` 的相同位置；
- 需要符号扩展的立即数格式都从同一位（`instr[31]`）开始扩展；
- 看似奇怪的 `LUI`、`AUIPC`、`JAL`、`JALR` 可以组合起来实现更高级的任务（把 32 位常数载入寄存器、跳转到任意地址、调用函数）。它们的存在使硬件设计更简单；与此同时，`CALL`、`RET` 等_伪指令_又让汇编程序员的工作更轻松。请参阅 [RISC-V 汇编手册](https://github.com/riscv/riscv-asm-manual/blob/master/riscv-asm.md)末尾的两张表。通过交换参数得到的测试/分支指令也是如此（例如 `a < b <=> b > a` 等），相应的伪指令会替你完成转换。

换一种说法，要体会 RISC-V ISA 的优雅，可以想象你的任务是_发明它_：既要发明整套指令，也要发明把指令编码成位模式的方式。约束条件如下：
- 指令长度固定为 32 位；
- 尽可能简单——极致的复杂归于简洁（达·芬奇）！！
- 源寄存器和目标寄存器始终编码在相同位置；
- 需要符号扩展时，始终从同一位开始；
- 应当容易把任意 32 位立即数载入寄存器（即使可能需要多条指令）；
- 应当容易跳转到任意存储器位置（即使可能需要多条指令）；
- 应当容易实现函数调用（即使可能需要多条指令）。

这样你就会明白为什么需要多种立即数格式。以 `JAL` 和 `JALR` 为例：前者没有源寄存器，后者则有。二者都带立即数，但由于 `JAL` 无需编码源寄存器，它多出 5 位可用于存放立即数。哪怕只剩一位可用，也会拿来扩展立即数的动态范围。这既解释了为什么存在多种立即数格式，也解释了这些格式为何要由 `instr` 的多个片段拼成——它们需要在三个位置固定、却不一定同时出现的 5 位寄存器编码之间穿梭。

现在可以理解 `LUI`、`AUIPC`、`JAL` 和 `JALR` 这些奇怪指令背后的理由了：它们提供一组可以组合使用的功能，既能把任意 32 位值载入寄存器，也能跳到存储器中的任意位置，还能尽可能简单地实现函数调用协议。考虑到上述约束，这些起初让我感到奇怪的选择完全合情合理。此外，这样的设计还让指令译码器相当简单，逻辑深度也很低。除了 7 位指令译码部分之外，它基本只包含从 `instr` 各个位引出的连线，以及为构造立即数而复制并符号扩展第 31 位的逻辑。

继续之前，我还想谈谈 `zero` 寄存器。我认为它实在是高明之举。有了它，就不再需要 `MOV rd rs` 指令（使用 `ADD rd rs zero` 即可），不需要 `NOP` 指令（使用 `ADD zero zero zero` 即可），所有分支变体也都能与 `zero` 比较！我认为 `zero` 是一项伟大的发明；虽然没有数字“0”本身那么伟大，却确实让指令集更加紧凑。

## 第 5 步：寄存器堆和状态机

寄存器堆实现如下：
```verilog
   reg [31:0] RegisterBank [0:31];
```

让我们仔细看看执行一条指令需要做什么。以一串 R-type 指令为例，每条指令都要完成以下四件事：

- 取指令：`instr <= MEM[PC]`；
- 读取 `rs1` 和 `rs2` 的值：`rs1 <= RegisterBank[rs1Id]; rs2 <= RegisterBank[rs2Id]`，其中 `rs1` 和 `rs2` 是两个寄存器。之所以需要这一步，是因为 `RegisterBank` 会被综合成一块 BRAM，而访问 BRAM 内容需要一个周期；
- 计算 `rs1` `OP` `rs2`（其中 `OP` 由 `funct3` 和 `funct7` 决定）；
- 把结果存入 `rd`：`RegisterBank[rdId] <= writeBackData`。如果 `OP` 由组合电路计算，这一步可以与上一步在同一周期完成。

前三项操作由一个状态机实现，如下所示（参见 [step5.v](step5.v)）：
```verilog
   localparam FETCH_INSTR = 0;
   localparam FETCH_REGS  = 1;
   localparam EXECUTE     = 2;
   reg [1:0] state = FETCH_INSTR;
   always @(posedge clk) begin
	 case(state)
	   FETCH_INSTR: begin
	      instr <= MEM[PC];
	      state <= FETCH_REGS;
	   end
	   FETCH_REGS: begin
	      rs1 <= RegisterBank[rs1Id];
	      rs2 <= RegisterBank[rs2Id];
	      state <= EXECUTE;
	   end
	   EXECUTE: begin
	      PC <= PC + 1;
	      state <= FETCH_INSTR;
	   end
	 endcase
      end
   end
```

第四项操作（寄存器回写）在下面的块中实现：
```verilog
   wire [31:0] writeBackData = ... ;
   wire writeBackEn = ...;
   always @posedge(clk) begin
      if(writeBackEn && rdId != 0) begin
          RegisterBank[rdId] <= writeBackData;
      end
   end
```
请记住，写入 0 号寄存器不会产生任何效果（因此要测试 `rdId != 0`）。当需要把 `writeBackData` 写入 `rdId` 所指寄存器时，信号 `writeBackEn` 就会置位。下一节将说明，要回写的数据 `writeBackData` 来自 ALU。

**动手试试**：在仿真和真实设备上运行 [step5.v](step5.v)。你会看到这颗准 CPU 的状态机在 LED 上跳起华尔兹（LED 显示当前状态）。

## 第 6 步：ALU

现在，我们已经能从存储器取出指令、对其译码并读取寄存器值，但这颗准 CPU 仍然什么也做不了。下面来看看如何真正对寄存器值执行计算。

_那么，你打算创建一个 `ALU` 模块吗？话说回来，为什么之前没有创建 `Decoder` 模块和 `RegisterBank` 模块？_

我的第一版设计使用了多个模块和多个文件，总计约 1000 行代码。后来 Matthias Koch 写出了一个单体版本，只有 200 行。不仅更加紧凑，而且所有内容都集中在一个地方，理解起来也容易得多。

**经验法则：**如果盒子和盒子之间的连线比盒子里的电路还多，那就说明盒子分得太多了！

_可等一下，模块化设计不是很好吗？_

模块化设计本身无所谓好坏；当它能让事情更简单时才有用，而目前的情形并非如此。当然，这个问题没有绝对答案，取决于个人品味和风格！本教程采用（大体上）单体式设计。

现在要实现两类指令：
- Rtype：`rd` <- `rs1` `OP` `rs2`（由 `isALUreg` 识别）；
- Itype：`rd` <- `rs1` `OP` `Iimm`（由 `isALUimm` 识别）。

ALU 接收两个输入 `aluIn1` 和 `aluIn2`，计算 `aluIn1` `OP` `aluIn2`，并把结果存入 `aluOut`：
```verilog
   wire [31:0] aluIn1 = rs1;
   wire [31:0] aluIn2 = isALUreg ? rs2 : Iimm;
   reg [31:0] aluOut;
```
根据指令类型，`aluIn2` 要么取第二个源寄存器 `rs2` 中的值，要么取 `Itype` 格式的立即数（`Iimm`）。运算 `OP` 主要由 `funct3` 决定（也会用到 `funct7`）。请把 [RISC-V 参考手册](https://github.com/riscv/riscv-isa-manual/releases/download/Ratified-IMAFDQC/riscv-spec-20191213.pdf)翻到第 130 页，放在手边或另一个窗口中：

| funct3 | 运算 |
|--------|------|
| 3'b000 | `ADD` 或 `SUB` |
| 3'b001 | 左移 |
| 3'b010 | 有符号比较（<） |
| 3'b011 | 无符号比较（<） |
| 3'b100 | `XOR` |
| 3'b101 | 逻辑右移或算术右移 |
| 3'b110 | `OR` |
| 3'b111 | `AND` |

- 对于 `ADD`/`SUB`，若为 `ALUreg` 运算（Rtype），则通过测试 `funct7` 的第 5 位区分 `ADD` 与 `SUB`（为 1 时是 `SUB`）。若为 `ALUimm` 运算（Itype），则只能是 `ADD`。在此情形下，只需测试 `instr` 的第 5 位即可区分 `ALUreg`（为 1）和 `ALUimm`（为 0）；
- 对于逻辑右移和算术右移，同样测试 `funct7` 的第 5 位来区分：1 表示带符号扩展的算术右移，0 表示逻辑右移；
- 对 `ALUreg` 指令，移位量来自 `rs2` 的内容；对 `ALUimm` 指令，移位量来自 `instr[24:20]`（与 `rs2Id` 使用相同的位）。

把这些内容组合起来，就得到下面的 ALU VERILOG 代码：
```verilog
   reg [31:0] aluOut;
   wire [4:0] shamt = isALUreg ? rs2[4:0] : instr[24:20]; // shift amount
   always @(*) begin
      case(funct3)
	3'b000: aluOut = (funct7[5] & instr[5]) ? (aluIn1-aluIn2) : (aluIn1+aluIn2);
	3'b001: aluOut = aluIn1 << shamt;
	3'b010: aluOut = ($signed(aluIn1) < $signed(aluIn2));
	3'b011: aluOut = (aluIn1 < aluIn2);
	3'b100: aluOut = (aluIn1 ^ aluIn2);
	3'b101: aluOut = funct7[5]? ($signed(aluIn1) >>> shamt) : (aluIn1 >> shamt);
	3'b110: aluOut = (aluIn1 | aluIn2);
	3'b111: aluOut = (aluIn1 & aluIn2);
      endcase
   end
```
_注：_虽然 `aluOut` 声明为 `reg`，但它仍是组合逻辑函数（不会生成触发器）。因为它的值由组合逻辑块（`always @(*)`）决定，而且 `case` 语句枚举了所有情况。

寄存器回写配置如下：
```verilog
   assign writeBackData = aluOut;
   assign writeBackEn = (state == EXECUTE && (isALUreg || isALUimm));
```

**动手试试**：分别在仿真和真实设备上运行 [step6.v](step6.v)。仿真会为每次寄存器回写显示写入值和目标寄存器；真实设备则会用 LED 显示 `x1` 的最低 5 位。然后可以尝试修改程序，观察寄存器值受到的影响。

**你到这里了！** 下表列出了需要实现的指令，这颗准 RISC-V 内核目前已经支持其中 20 条。接下来先实现跳转，再实现分支，然后……完成其余部分。不过在那之前，你大概已经注意到，把 RISC-V 程序翻译成二进制（即手工汇编）极其痛苦。下一节会给出一种容易得多的办法。

| ALUreg | ALUimm | Jump  | Branch | LUI | AUIPC | Load  | Store | SYSTEM |
|--------|--------|-------|--------|-----|-------|-------|-------|--------|
| [*] 10 | [*] 9  | [ ] 2 | [ ] 6  | [ ] | [ ]   | [ ] 5 | [ ] 3 | [*] 1  |

## 第 7 步：使用 VERILOG 汇编器

为了避免手工把 RISC-V 汇编转换成二进制，可以使用 GNU 汇编器生成二进制文件，再将其转换成十六进制，最后用 VERILOG 函数 `readmemh()` 按文件内容初始化存储器。后面会介绍具体做法。

不过对我们而言，如果能把小型汇编程序直接写在与设计相同的 VERILOG 文件里，会非常方便。事实上完全可以做到：[riscv_assembly.v](riscv_assembly.v) 直接使用 task 和 function 在 VERILOG 中实现了一个 RISC-V 汇编器。

在 [step7.v](step7.v) 中，存储器由与 [step6.v](step6.v) 相同的汇编程序初始化。现在看起来是下面这样，易读多了，不是吗？
```verilog
   `include "riscv_assembly.v"
   initial begin
      ADD(x0,x0,x0);
      ADD(x1,x0,x0);
      ADDI(x1,x1,1);
      ADDI(x1,x1,1);
      ADDI(x1,x1,1);
      ADDI(x1,x1,1);
      ADD(x2,x1,x0);
      ADD(x3,x1,x2);
      SRLI(x3,x3,3);
      SLLI(x3,x3,31);
      SRAI(x3,x3,5);
      SRLI(x1,x3,26);
      EBREAK();
   end
```
_注：_`riscv_assembly.v` 需要从使用汇编代码的模块内部包含进来。

这一步还要做一项修改：在前面的步骤中，`PC` 是当前指令的索引；接下来要让它表示当前指令的_地址_。由于每条指令长 32 位，因此：
- 递增 `PC` 时执行 `PC <= PC + 4`（而不再像以前那样执行 `PC <= PC + 1`）；
- 取当前指令时执行 `instr <= MEM[PC[31:2]];`（忽略 `PC` 的最低两位）。

## 第 8 步：跳转

跳转指令有两条：`JAL`（jump and link，跳转并链接）和 `JALR`（jump and link register，寄存器间接跳转并链接）。“并链接”是指可以把当前 PC 写入寄存器。因此，`JAL` 和 `JALR` 不仅可以实现跳转，也可以实现函数调用。两条指令应当产生如下效果：

| 指令 | 效果 |
|------|------|
| JAL rd,imm | rd<-PC+4; PC<-PC+Jimm |
| JALR rd,rs1,imm | rd<-PC+4; PC<-rs1+Iimm |

为了实现这两条指令，需要对内核作出以下修改。首先是寄存器回写：对跳转指令而言，回写值现在可以是 `PC+4`，而不只是 `aluOut`：
```verilog
   assign writeBackData = (isJAL || isJALR) ? (PC + 4) : aluOut;
   assign writeBackEn = (state == EXECUTE &&
			 (isALUreg ||
			  isALUimm ||
			  isJAL    ||
			  isJALR)
			 );
```

还需要声明一个 `nextPC` 值，以实现三种可能情况：
```verilog
   wire [31:0] nextPC = isJAL  ? PC+Jimm :
	                isJALR ? rs1+Iimm :
	                PC+4;
```

然后在状态机中，把 `PC <= PC + 4;` 替换成 `PC <= nextPC;`，这样就完成了！

现在可以实现一个简单的无限循环，测试新加入的跳转指令：
```verilog
`include "riscv_assembly.v"
      integer L0_=4;
      initial begin
	 ADD(x1,x0,x0);
      Label(L0_);
	 ADDI(x1,x1,1);
	 JAL(x0,LabelRef(L0_));
	 EBREAK();
	 endASM();
      end
```

整数 `L0_` 是一个标签。与真正的汇编器不同，这里需要手工指定 `L0_` 的值。本例很简单：`L0_` 紧跟在第一条指令之后，因此它对应 RAM 起始地址（0）加上一个 32 位字，也就是 4。对于带有许多标签的较长程序，可以不初始化标签（`integer L0_;`）。程序第一次运行时会计算并显示各个标签应使用的值。这种方式不算特别方便，但仍比手工汇编、手工确定标签好得多。

`LabelRef()` 函数计算标签相对于当前程序计数器的偏移量。此外，在仿真模式下，它会显示当前地址（用于初始化标签）；如果标签已经初始化（如这里的 `L0_=4`），它还会检查该标签是否与汇编器生成的当前地址相符。若不相符，`endASM()` 语句会显示错误消息并退出。

_注 1：_我习惯在程序末尾插入一条 `EBREAK()` 指令。本例中并非必需（因为存在无限循环），但如果之后改变主意、让程序退出循环，`EBREAK()` 已经就位。

_注 2：_`endASM();` 语句会检查所有标签是否有效，一旦发现无效标签就退出仿真。如果使用 RISC-V VERILOG 汇编器，请务必在综合前先仿真设计（因为综合时无法进行这项检查）。

**动手试试**：分别在仿真和真实设备上运行设计 [step8.v](step8.v)。没错，经过 8 个步骤，我们得到的还是一个傻乎乎的闪灯器！但这一次，它执行的可是真正的 RISC-V 程序！它还不是完整的 RISC-V 内核，却已经有了浓厚的 RISC-V 味道。耐心一点，我们的内核很快就能运行比闪灯器更有意思的 RISC-V 程序。

**你到这里了！**
还有一些工作要做，但我们正在稳步前进。
| ALUreg | ALUimm | Jump  | Branch | LUI | AUIPC | Load  | Store | SYSTEM |
|--------|--------|-------|--------|-----|-------|-------|-------|--------|
| [*] 10 | [*] 9  | [*] 2 | [ ] 6  | [ ] | [ ]   | [ ] 5 | [ ] 3 | [*] 1  |

**动手试试**：在循环前添加几条指令，运行仿真，按照仿真器的提示修正标签，再次运行仿真，然后在真实设备上运行。

## 第 9 步：分支

分支与跳转类似，只不过它会比较两个寄存器，并根据比较结果更新 `PC`。另一个区别是，分支从 `PC` 出发所能到达的地址范围更有限（12 位偏移量）。共有 6 条不同的分支指令：

| 指令 | 效果 |
|------|------|
| BEQ rs1,rs2,imm | if(rs1 == rs2) PC <- PC+Bimm |
| BNE rs1,rs2,imm | if(rs1 != rs2) PC <- PC+Bimm |
| BLT rs1,rs2,imm | if(rs1 <  rs2) PC <- PC+Bimm（有符号比较） |
| BGE rs1,rs2,imm | if(rs1 >= rs2) PC <- PC+Bimm（有符号比较） |
| BLTU rs1,rs2,imm | if(rs1 <  rs2) PC <- PC+Bimm（无符号比较） |
| BGEU rs1,rs2,imm | if(rs1 >= rs2) PC <- PC+Bimm（无符号比较） |

_等一下：_这里有 `BLT`，但 `BGT` 去哪里了？RISC-V 处理器始终遵循同一条原则：如果已有功能能完成某件事，就不要再增加新功能！在这里，`BGT rs1,rs2,imm` 等价于 `BLT rs2,rs1,imm`（只需交换前两个操作数）。在 RISC-V 汇编程序中使用 `BGT` 也能正常工作，汇编器会把它替换成操作数互换后的 `BLT`。`BGT` 称为“伪指令”。RISC-V 提供了许多伪指令，让汇编程序员的工作更轻松（后面还会继续介绍）。

回到分支指令。我们需要在 ALU 中增加一些连线，用于计算测试结果：
```verilog
   reg takeBranch;
   always @(*) begin
      case(funct3)
	3'b000: takeBranch = (rs1 == rs2);
	3'b001: takeBranch = (rs1 != rs2);
	3'b100: takeBranch = ($signed(rs1) < $signed(rs2));
	3'b101: takeBranch = ($signed(rs1) >= $signed(rs2));
	3'b110: takeBranch = (rs1 < rs2);
	3'b111: takeBranch = (rs1 >= rs2);
	default: takeBranch = 1'b0;
      endcase
```
_注 1：_可以创建更加紧凑的 ALU，使综合后使用的 LUT 少得多，后面会介绍（目前的目标是先得到一颗能工作的 RISC-V 处理器，之后再优化）。

_注 2：_`funct3` 给出的 8 种可能中，分支指令只使用 6 种。`case` 中必须有 `default:` 语句，否则综合器无法把 `takeBranch` 保持为纯组合逻辑，反而会生成我们不想要的锁存器。

要实现分支，现在只剩下一件事：为 `nextPC` 增加一种情况，如下所示：
```verilog
   wire [31:0] nextPC = (isBranch && takeBranch) ? PC+Bimm :
   	                isJAL                    ? PC+Jimm :
	                isJALR                   ? rs1+Iimm :
	                PC+4;
```

现在可以测试一个简单循环：从 0 数到 31，每次迭代都显示在 LED 上（还记得吗？LED 连接到 `x1`），然后停止：

```c++
`include "riscv_assembly.v"
      integer L0_ = 8;

      initial begin
         ADD(x1,x0,x0);
         ADDI(x2,x0,32);
      Label(L0_);
	 ADDI(x1,x1,1);
         BNE(x1, x2, LabelRef(L0_));
         EBREAK();

	 endASM();
      end
```

**动手试试**：分别在仿真和真实设备上运行 [step9.v](step9.v)。尝试修改程序，用一个外层循环和两个内层循环（一个从左向右，另一个从右向左）创建《霹雳游侠》扫描灯。

**你到这里了！**
哇，38 条指令已经实现了 28 条！继续吧……

| ALUreg | ALUimm | Jump  | Branch | LUI | AUIPC | Load  | Store | SYSTEM |
|--------|--------|-------|--------|-----|-------|-------|-------|--------|
| [*] 10 | [*] 9  | [*] 2 | [ *] 6 | [ ] | [ ]   | [ ] 5 | [ ] 3 | [*] 1  |

## 第 10 步：LUI 和 AUIPC

还有这两条奇怪的指令需要实现。它们做什么呢？其实很简单：

| 指令 | 效果 |
|------|------|
| LUI rd, imm | rd <= Uimm |
| AUIPC rd, imm | rd <= PC + Uimm |

观察 `Uimm` 格式可知，它从指令编码的立即数中读取最高位（`imm[31:12]`），最低 12 位则设为零。这两条指令极其有用：其他所有指令支持的立即数格式都只能修改最低位；与这两项功能结合后，就可以把任意值载入寄存器（但最多可能需要两条指令）。

实现这两条指令，只需按如下方式修改 `writeBackEn` 和 `writeBackData`：
```verilog
   assign writeBackData = (isJAL || isJALR) ? (PC + 4) :
			  (isLUI) ? Uimm :
			  (isAUIPC) ? (PC + Uimm) :
			  aluOut;

   assign writeBackEn = (state == EXECUTE &&
			 (isALUreg ||
			  isALUimm ||
			  isJAL    ||
			  isJALR   ||
			  isLUI    ||
			  isAUIPC)
			 );
```

**你到这里了！**
看来快要完成了！还剩 8 条指令……

| ALUreg | ALUimm | Jump  | Branch | LUI | AUIPC | Load  | Store | SYSTEM |
|--------|--------|-------|--------|-----|-------|-------|-------|--------|
| [*] 10 | [*] 9  | [*] 2 | [ *] 6 | [*] | [*]   | [ ] 5 | [ ] 3 | [*] 1  |


**动手试试**：分别在仿真和真实设备上运行 [step10.v](step10.v)。

_啊！！_我的 IceStick 装不下了（设计需要 1283 个 LUT，而 IceStick 只有 1280 个）。怎么办？别忘了，此前我们完全没有考虑资源消耗，只想先写出一个能工作的设计。事实上，这个设计还有_大量_改进空间，后面会看到。不过在此之前，先把 SOC 组织得更合理一些（之后再缩小处理器）。

## 第 11 步：把存储器放入独立模块

在此前的设计中，所有内容（存储器和处理器）都放在 `SOC` 模块里。本步将学习如何把它们分开。

首先是 `Memory` 模块：

```verilog
module Memory (
   input             clk,
   input      [31:0] mem_addr,  // address to be read
   output reg [31:0] mem_rdata, // data read from memory
   input   	     mem_rstrb  // goes high when processor wants to read
);
   reg [31:0] MEM [0:255];

`include "riscv_assembly.v"
   integer L0_=8;
   initial begin
                  ADD(x1,x0,x0);
                  ADDI(x2,x0,31);
      Label(L0_); ADDI(x1,x1,1);
                  BNE(x1, x2, LabelRef(L0_));
                  EBREAK();
      endASM();
   end

   always @(posedge clk) begin
      if(mem_rstrb) begin
         mem_rdata <= MEM[mem_addr[31:2]];
      end
   end
endmodule
```

其接口中有一个连接时钟的 `clk` 信号。处理器每次要读取存储器时，都把待读地址放到 `mem_addr` 上，并把 `mem_rstrb` 置为 1；随后 `Memory` 模块通过 `mem_rdata` 返回读取的数据。

与之对应，`Processor` 模块也有 `mem_addr` 信号（这次是 `output`）、`mem_rdata` 信号（输入）和 `mem_rstrb` 信号（输出）：

```verilog
module Processor (
    input 	      clk,
    input 	      resetn,
    output     [31:0] mem_addr,
    input      [31:0] mem_rdata,
    output 	      mem_rstrb,
    output reg [31:0] x1
);
...
endmodule
```
（此外还有一个 `x1` 信号，保存寄存器 `x1` 的内容，可用于可视化调试。我们会把它连接到 LED。）

状态机增加了一个状态：
```verilog
   localparam FETCH_INSTR = 0;
   localparam WAIT_INSTR  = 1;
   localparam FETCH_REGS  = 2;
   localparam EXECUTE     = 3;

   case(state)
     FETCH_INSTR: begin
       state <= WAIT_INSTR;
     end
     WAIT_INSTR: begin
       instr <= mem_rdata;
       state <= FETCH_REGS;
     end
     FETCH_REGS: begin
       rs1 <= RegisterBank[rs1Id];
       rs2 <= RegisterBank[rs2Id];
       state <= EXECUTE;
     end
     EXECUTE: begin
        if(!isSYSTEM) begin
  	   PC <= nextPC;
	end
	state <= FETCH_INSTR;
      end
   endcase
```
_注：_后面会看到如何简化它，重新变回三个状态。

现在可以按如下方式连接 `mem_addr` 和 `mem_rstrb`：
```verilog
   assign mem_addr = PC;
   assign mem_rstrb = (state == FETCH_INSTR);
```

最后，在 `SOC` 中实例化并连接所有模块：
```verilog
module SOC (
    input  CLK,        // system clock
    input  RESET,      // reset button
    output [4:0] LEDS, // system LEDs
    input  RXD,        // UART receive
    output TXD         // UART transmit
);
   wire    clk;
   wire    resetn;
   Memory RAM(
      .clk(clk),
      .mem_addr(mem_addr),
      .mem_rdata(mem_rdata),
      .mem_rstrb(mem_rstrb)
   );

   wire [31:0] mem_addr;
   wire [31:0] mem_rdata;
   wire mem_rstrb;
   wire [31:0] x1;
   Processor CPU(
      .clk(clk),
      .resetn(resetn),
      .mem_addr(mem_addr),
      .mem_rdata(mem_rdata),
      .mem_rstrb(mem_rstrb),
      .x1(x1)
   );
   assign LEDS = x1[4:0];

   // Gearbox and reset circuitry.
   Clockworks #(
     .SLOW(19) // Divide clock frequency by 2^19
   ) CW (
     .CLK(CLK),
     .RESET(RESET),
     .clk(clk),
     .resetn(resetn)
   );

   assign TXD  = 1'b0; // not used for now
endmodule
```

现在可以在仿真器中运行 [step11.v](step11.v)。不出所料，它与上一步的行为相同（LED 从 0 数到 31，然后停止）。那在真实设备上呢？哇，情况反而更糟：需要 1341 个 LUT（而 IceStick 只有 1280 个）。所以让我们缩小代码，让它装得下！

## 第 12 步：尺寸优化——不可思议的缩小内核

_向经典电影《不可思议的收缩人》致敬_

有很多办法可以缩小这颗内核。先来看看 ALU。它能计算加法、减法和比较，那么能否复用减法结果来完成比较？当然可以，但为此需要计算一次 33 位减法，并测试符号位。Matthias Koch（@Mecrisp）向我讲解了这个技巧；swapforth/J1（另一颗可在 IceStick 上运行的小型 RISC 内核）也采用了它。33 位减法写法如下：
```verilog
   wire [32:0] aluMinus = {1'b0,aluIn1} - {1'b0,aluIn2};
```
如果想知道 Verilog 中 `A-B` 实际做了什么，它等价于 `A+~B+1`（先把 B 的所有位取反，再相加并加 1），这正是二进制补码减法的工作方式。例如，`4'b0000 - 4'b0001` 的结果是 `-1`，编码为 `4'b1111`。按公式计算就是：`4'b0000 + ~4'b0001 + 1` = `4'b0000 + 4'b1110 + 1` = `4'b1111`。因此我们保留下方表达式（虽然也可以使用上面更简单的形式，但了解幕后发生的事情很有意思）：
```verilog
   wire [32:0] aluMinus = {1'b1, ~aluIn2} + {1'b0,aluIn1} + 33'b1;
```
然后可以为三种测试创建连线（这样可节省三个 32 位加法器）：
```
   wire        EQ  = (aluMinus[31:0] == 0);
   wire        LTU = aluMinus[32];
   wire        LT  = (aluIn1[31] ^ aluIn2[31]) ? aluIn1[31] : aluMinus[32];
```

- 第一个信号 `EQ` 在 `aluIn1` 与 `aluIn2` 相等，也就是 `aluMinus == 0` 时拉高（无需测试第 33 位）；
- 第二个信号 `LTU` 对应无符号比较，其值由 33 位减法结果的符号位给出；
- 第三个信号分两种情况：若两数符号不同，则当 `aluIn1` 为负时拉高 `LT`；否则，`LT` 由 33 位减法结果的符号位给出。

当然，加法仍然需要一个加法器：
```verilog
   wire [31:0] aluPlus = aluIn1 + aluIn2;
```

随后按如下方式计算 `aluOut`：
```verilog
   reg [31:0]  aluOut;
   always @(*) begin
      case(funct3)
	3'b000: aluOut = (funct7[5] & instr[5]) ? aluMinus[31:0] : aluPlus;
	3'b001: aluOut = aluIn1 << shamt;;
	3'b010: aluOut = {31'b0, LT};
	3'b011: aluOut = {31'b0, LTU};
	3'b100: aluOut = (aluIn1 ^ aluIn2);
	3'b101: aluOut = funct7[5]? ($signed(aluIn1) >>> shamt) :
			 ($signed(aluIn1) >> shamt);
	3'b110: aluOut = (aluIn1 | aluIn2);
	3'b111: aluOut = (aluIn1 & aluIn2);
      endcase
   end
```

在 IceStick 上试试。成功！1167 个 LUT，装得下！不过这还不是停手的理由，仍有不少缩小空间的机会。看看 `takeBranch`：能否复用刚刚创建的 `EQ`、`LT` 和 `LTU` 信号？当然可以：

```verilog
   reg takeBranch;
   always @(*) begin
      case(funct3)
	3'b000: takeBranch = EQ;
	3'b001: takeBranch = !EQ;
	3'b100: takeBranch = LT;
	3'b101: takeBranch = !LT;
	3'b110: takeBranch = LTU;
	3'b111: takeBranch = !LTU;
	default: takeBranch = 1'b0;
      endcase
   end
```

为了让它正常工作，还要确保在处理分支时也把 `rs2` 接到 ALU 的第二个输入：

```verilog
   wire [31:0] aluIn2 = isALUreg | isBranch ? rs2 : Iimm;
```

真实设备上的结果如何？1094 个 LUT，还不错，不过继续优化……`JALR` 的跳转目标是 `rs1+Iimm`，我们此前专门为它创建了一个加法器。这有点蠢，因为 ALU 已经算出了这个值。好吧，复用它：

```verilog
   wire [31:0] nextPC = ((isBranch && takeBranch) || isJAL) ? PCplusImm  :
	                isJALR                              ? {aluPlus[31:1],1'b0}:
	                PCplus4;
```

现在怎么样？1030 个 LUT。还没结束：最消耗 LUT 的是移位器，而 ALU 中足足有三个（分别用于左移、逻辑右移和算术右移）。Matthias Koch（@mecrisp）还告诉了我另一招“魔法”：创建一个 33 位移位器，根据输入的第 31 位及其属于逻辑移位还是算术移位，把附加位设为 0 或 1，就能合并两个右移器。
```verilog
   wire [31:0] shifter =
               $signed({instr[30] & aluIn1[31], shifter_in}) >>> aluIn2[4:0];
```

更妙的是，Matthias 告诉我，其实只用一个移位器就够了：若为左移，则反转输入，再反转输出：
```verilog
   wire [31:0] shifter_in = (funct3 == 3'b001) ? flip32(aluIn1) : aluIn1;
   wire [31:0] leftshift = flip32(shifter);
```

于是 ALU 变成下面这样：
```verilog
   reg [31:0]  aluOut;
   always @(*) begin
      case(funct3)
	3'b000: aluOut = (funct7[5] & instr[5]) ? aluMinus[31:0] : aluPlus;
	3'b001: aluOut = leftshift;
	3'b010: aluOut = {31'b0, LT};
	3'b011: aluOut = {31'b0, LTU};
	3'b100: aluOut = (aluIn1 ^ aluIn2);
	3'b101: aluOut = shifter;
	3'b110: aluOut = (aluIn1 | aluIn2);
	3'b111: aluOut = (aluIn1 & aluIn2);
      endcase
   end
```

现在呢？朋友，只要 887 个 LUT！

_注 1：_事实上，还可以让移位器每个时钟周期只移动 1 位，从而节省更多空间。这样 ALU 会稍微复杂一些（变成多周期），但尺寸会小得多（Femtorv32-quark 就使用了这一技巧）。后面会介绍。

_注 2：_使用多周期 ALU 后，还可以只保留一个 33 位加法器，并把 `~aluIn2`、`aluIn1+(~aluIn2)` 和 `aluIn1+(~aluIn2)+1` 的计算拆开，用三个周期完成减法。

在那之前，还可以轻松地把地址计算使用的加法器提取出来：
```verilog
   wire [31:0] PCplusImm = PC + ( instr[3] ? Jimm[31:0] :
				  instr[4] ? Uimm[31:0] :
				             Bimm[31:0] );
   wire [31:0] PCplus4 = PC+4;
```

这样，`nextPC` 和 `writeBackData` 就能共享这两个加法器：
```verilog

   assign writeBackData = (isJAL || isJALR) ? (PCplus4) :
			  (isLUI) ? Uimm :
			  (isAUIPC) ? PCplusImm :
			  aluOut;

   assign writeBackEn = (state == EXECUTE && !isBranch);

   wire [31:0] nextPC = (isBranch && takeBranch || isJAL) ? PC+Imm  :
	                isJALR                   ? {aluPlus[31:1],1'b0} :
	                PCplus;
```

最终结果呢？839 个 LUT（又省了大约 50 个……）。还有继续节省 LUT 的空间（例如用多周期 ALU 完成移位，并减少地址计算使用的位数），不过先留到以后。现在设备上已经有足够空间继续后面的步骤了。

## 第 13 步：子程序（版本 1，使用普通 RISC-V 指令）

好了，现在已经有了一颗（尚不完整的）RISC-V 处理器和一个 SOC，而且二者都能装进设备。还记得吗？我们正在接近终点，只剩 8 条指令（5 种 Load、3 种 Store）。

| ALUreg | ALUimm | Jump  | Branch | LUI | AUIPC | Load  | Store | SYSTEM |
|--------|--------|-------|--------|-----|-------|-------|-------|--------|
| [*] 10 | [*] 9  | [*] 2 | [ *] 6 | [*] | [*]   | [ ] 5 | [ ] 3 | [*] 1  |

在攻克它们之前，先进一步学习 RISC-V 汇编和函数调用。此前我们一直使用变速箱降低 CPU 速度，以便观察它执行程序。能不能改为实现并调用一个 `wait` 函数呢？下面就来看看。

首先，移除 `Clockworks` 实例化中的 `#(.SLOW(nnn))` 参数：
```verilog
   Clockworks CW(
     .CLK(CLK),
     .RESET(RESET),
     .clk(clk),
     .resetn(resetn)
   );
```
这样将不再生成变速箱，而是把开发板的 `CLK` 信号直接连接到设计使用的内部 `clk` 信号。

接下来要解决两个问题：
- 如何编写一个等待一段时间的函数；
- 如何调用它。

_等一下，_你在谈函数调用，可我们还没有 `Load`/`Store` 指令。我们无法把返回地址压入栈中（因为还不能读写存储器，而栈位于存储器中！），那怎么可能实现呢？

使用 RISC-V 指令实现函数调用的方法有很多。为了确保所有人采用同一套约定，标准规定了一个**应用程序二进制接口**（ABI），用于定义如何调用函数、如何传递参数，以及每个寄存器的用途。详情请参阅[这份文档](https://github.com/riscv-non-isa/riscv-asm-manual/blob/master/riscv-asm.md)。

**调用函数：**从文档可知，调用函数时，返回地址存放在 `x1` 中。因此可以使用 `JAL(x1,offset)` 调用函数，其中 `offset` 是程序计数器与被调函数地址之间的有符号差值。只要偏移量能装入 20 位（Jimm 格式），这种方式就能工作。

_注：_对于距离更远的函数，可以组合使用 `AUIPC` 和 `JALR`，从而到达任意偏移量。

**从函数返回：**跳转到 `x1` 中保存的地址即可，可以使用 `JALR(x0,x1,0)` 完成。

**函数参数和返回值：**前 6 个函数参数通过 `x10`..`x16` 传递，返回值通过 `x10` 传递（会覆盖第一个函数参数）。

这很有意思：即使还没有 `Load`/`Store`，我们也能编写带函数的程序。不过，函数还不能调用其他函数，因为那需要把 `x1` 保存到栈中（严格来说，也可以把 `x1` 保存到另一个寄存器，但很快就会一团糟，所以不这样做）。

还有一件小事：我们刚刚了解到，在 ABI 中 `x1` 用于保存函数返回地址，而此前它一直连接到 LED。既然现在要遵守 ABI，就需要换一个寄存器。从现在开始，把 `x10` 连接到 LED。

好了，现在具备了编写又一个闪灯器版本所需的一切！选择一个 `slow_bit` 常量，编写一个计数到 `2^slow_bit` 的 `wait` 函数，再调用它来降低闪灯器速度：

```verilog
`ifdef BENCH
   localparam slow_bit=15;
`else
   localparam slow_bit=19;
`endif


`include "riscv_assembly.v"
   integer L0_   = 4;
   integer wait_ = 20;
   integer L1_   = 28;

   initial begin
      ADD(x10,x0,x0);
   Label(L0_);
      ADDI(x10,x10,1);
      JAL(x1,LabelRef(wait_)); // call(wait_)
      JAL(zero,LabelRef(L0_)); // jump(l0_)

      EBREAK(); // I keep it systematically
                // here in case I change the program.

   Label(wait_);
      ADDI(x11,x0,1);
      SLLI(x11,x11,slow_bit);
   Label(L1_);
      ADDI(x11,x11,-1);
      BNE(x11,x0,LabelRef(L1_));
      JALR(x0,x1,0);

      endASM();
   end

   always @(posedge clk) begin
      if(mem_rstrb) begin
         mem_rdata <= MEM[mem_addr[31:2]];
      end
   end
endmodule
```


分别在仿真和真实设备上尝试 [step13.v](step13.v)。

**动手试试**：制作《霹雳游侠》扫描灯，分别编写从左向右、从右向左的子程序，再加上等待子程序。_提示：_你需要把 `x1` 保存到另一个寄存器。

## 第 14 步：子程序（版本 2，使用 RISC-V ABI 和伪指令）

借助 ABI，我们有了一套编写程序的标准方式，但需要记住很多事情：
- 所有 RISC-V 寄存器本质上都一样，但 ABI 要求将某些寄存器用于特定任务（`x1` 保存返回地址，`x10`..`x16` 传递函数参数，等等）；
- 函数调用使用 `JAL`，或者组合 `AUIPC` 和 `JALR` 实现；函数返回则使用 `JALR` 实现。

CISC 处理器通常有专门的函数调用指令（`CALL`）和函数返回指令（`RET`），寄存器也常常各司其职（函数返回地址、栈指针、函数参数）。这样程序员需要记忆的内容更少，工作更轻松。RISC 处理器完全没有理由不提供同样的便利！我们可以假装这些寄存器各不相同，给它们起不同的名字（或别名）。[这里](https://github.com/riscv-non-isa/riscv-asm-manual/blob/master/riscv-asm.md#general-registers)列出了这些名称。

| ABI 名称 | 寄存器名 | 用途 |
|----------|----------|------|
| `zero` | `x0` | 读：0；写：忽略 |
| `ra` | `x1` | 返回地址 |
| `t0`...`t6` | ... | 临时寄存器 |
| `fp`,`s0`...`s11` | ... | 被保存寄存器；`fp`=`s0`：帧指针 |
| `a0`...`a7` | ... | 函数参数和返回值（`a0`） |
| `sp` | `x2` | 栈指针 |
| `gp` | `x3` | 全局指针 |

函数应当保持被保存寄存器（`s0`...`s11`）不变，或者负责保存并恢复它们。局部变量可以放在这些寄存器中。如果编写函数，应当把用到的这些寄存器压入栈，并在返回前弹出。

对于其他所有寄存器，都不能指望它们在函数调用前后保持不变。

全局指针 `gp` 可以充当一种“捷径”，用一条指令访问较远的存储器区域。等有了 `Load` 和 `Store` 后再介绍。

在 VERILOG 汇编器 [riscv_assembly.v](riscv_assembly.v) 中，只需为寄存器名称声明这些别名：
```verilog
   localparam zero = x0;
   localparam ra   = x1;
   localparam sp   = x2;
   localparam gp   = x3;
   ...
   localparam t4   = x29;
   localparam t5   = x30;
   localparam t6   = x31;
```

除了这些名称，还有一些用于常见任务的_伪指令_，例如：

| 伪指令 | 操作 |
|--------|------|
| `LI(rd,imm)` | 把一个 32 位数载入寄存器 |
| `CALL(offset)` | 调用函数 |
| `RET()` | 从函数返回 |
| `MV(rd,rs)` | 等价于 `ADD(rd,rs,zero)` |
| `NOP()` | 等价于 `ADD(zero,zero,zero)` |
| `J(offset)` | 等价于 `JAL(zero,offset)` |
| `BEQZ(rd1,offset)` | 等价于 `BEQ(rd1,x0,offset)` |
| `BNEZ(rd1,offset)` | 等价于 `BNE(rd1,x0,offset)` |
| `BGT(rd1,rd2,offset)` | 等价于 `BLT(rd2,rd1,offset)` |

如果常数在 [-2048, 2047] 范围内，`LI` 使用 `ADDI(rd,x0,imm)` 实现；否则，它会组合使用 `LUI` 和 `ADDI`（如果想了解工作原理，请参阅[这个 Stack Overflow 回答](https://stackoverflow.com/questions/50742420/risc-v-build-32-bit-constants-with-lui-and-addi)，其中涉及一些符号扩展的微妙细节）。

使用 ABI 寄存器名称和伪指令后，程序变成下面这样：

```verilog
   integer L0_   = 4;
   integer wait_ = 24;
   integer L1_   = 32;

   initial begin
      LI(a0,0);
   Label(L0_);
      ADDI(a0,a0,1);
      CALL(LabelRef(wait_));
      J(LabelRef(L0_));

      EBREAK();

   Label(wait_);
      LI(a1,1);
      SLLI(a1,a1,slow_bit);
   Label(L1_);
      ADDI(a1,a1,-1);
      BNEZ(a1,LabelRef(L1_));
      RET();

      endASM();
   end
```
看起来差别不算大，但在较长的程序中，它能通过体现程序员的意图来提高可读性（这是函数，那是跳转到标签，等等）。如果不用这些名称和伪指令，所有内容看起来都差不多，程序就更难阅读。

这件事相当有趣：RISC-V 标准拥有极其简单的指令集，但直接用它编程并不轻松。于是 ABI 假装这套指令集像 CISC 处理器一样更复杂，反而让程序员更省心。它还保证一个程序员编写的函数可以由另一个程序员所写、甚至采用不同语言编写的函数调用。后面会介绍如何使用 GNU 汇编器和 C 编译器为我们的 CPU 编译程序。不过在把玩软件和工具链之前，别忘了硬件中还有 8 条指令要实现（5 种 `Load` 和 3 种 `Store`）。

**动手试试**：自行设计一个两数相乘的子程序（或者从[别处](https://github.com/riscv-collab/riscv-gcc/blob/5964b5cd72721186ea2195a7be8d40cfe6554023/libgcc/config/riscv/muldi3.S)借鉴），在仿真中用多组输入进行测试，再到真实设备上测试。

## 第 15 步：加载

现在来看看如何实现加载指令。共有 5 条不同的指令：

| 指令 | 效果 |
|------|------|
| LW(rd,rs1,imm) | 把地址 (rs1+imm) 处的字载入 rd |
| LBU(rd,rs1,imm) | 把地址 (rs1+imm) 处的字节载入 rd |
| LHU(rd,rs1,imm) | 把地址 (rs1+imm) 处的半字载入 rd |
| LB(rd,rs1,imm) | 把地址 (rs1+imm) 处的字节载入 rd，再进行符号扩展 |
| LH(rd,rs1,imm) | 把地址 (rs1+imm) 处的半字载入 rd，再进行符号扩展 |

_注：_`LW` 的地址按字边界对齐（4 字节的倍数），`LH`、`LHU` 的地址按半字边界对齐（2 字节的倍数）。这是件好事，让我们的工作容易了不少……

但仍有一些工作要做！首先需要一段用于确定加载值的电路（称为 `LOAD_data`）。

可以看到，我们有加载字、半字和字节的指令，而加载半字和字节的指令又各有两个版本：
- `LBU`、`LHU` 把一个字节或半字加载到 `rd` 的最低位；
- `LB`、`LH` 把一个字节或半字加载到 `rd` 的最低位，再进行符号扩展。

例如，假设一个有符号字节的值为 `-1`，即 `8'b11111111`。使用 `LBU` 把它载入 32 位寄存器，结果为 `32'b0000000000000000000000011111111`；使用 `LB` 加载，结果则为 `32'b11111111111111111111111111111111`，也就是 32 位形式的 `-1`。

这样就形成了一个“二维”的情况数组：既要区分加载的是字节、半字还是字，又要区分是否进行符号扩展。好吧，其实还要更复杂一些。别忘了，存储器按字组织，因此加载字节时，需要知道读取的是一个字中的哪一个字节（4 选 1）；加载半字时，也要知道读取的是哪一个半字（2 选 1）。检查待读数据地址（`rs1 + Iimm`）的最低两位即可完成选择：

```verilog
   wire [31:0] loadstore_addr = rs1 + Iimm;
   wire [15:0] LOAD_halfword =
	       loadstore_addr[1] ? mem_rdata[31:16] : mem_rdata[15:0];

   wire  [7:0] LOAD_byte =
	       loadstore_addr[0] ? LOAD_halfword[15:8] : LOAD_halfword[7:0];
```

现在需要在 `mem_rdata`（`LW`）、`LOAD_halfword`（`LH`、`LHU`）和 `LOAD_byte`（`LB`、`LBU`）之间选择。查看 [RISC-V 参考手册](https://github.com/riscv/riscv-isa-manual/releases/download/Ratified-IMAFDQC/riscv-spec-20191213.pdf)第 130 页的表格可知，它由 `funct3` 的最低两位决定：

```verilog
   wire mem_byteAccess     = funct3[1:0] == 2'b00;
   wire mem_halfwordAccess = funct3[1:0] == 2'b01;

   wire [31:0] LOAD_data =
         mem_byteAccess ? LOAD_byte     :
     mem_halfwordAccess ? LOAD_halfword :
                          mem_rdata     ;
```

现在还要在这个表达式中加入符号扩展。写入 `rd` 最高位的值 `LOAD_sign` 同时取决于两个条件：指令是否进行符号扩展（`LB`、`LH`，其特征是 `funct3[2]=0`），以及加载值的最高位：

```verilog
   wire LOAD_sign =
	!funct3[2] & (mem_byteAccess ? LOAD_byte[7] : LOAD_halfword[15]);

   wire [31:0] LOAD_data =
         mem_byteAccess ? {{24{LOAD_sign}},     LOAD_byte} :
     mem_halfwordAccess ? {{16{LOAD_sign}}, LOAD_halfword} :
                          mem_rdata ;
```

呼……过程稍显痛苦，但最终并不算太复杂。我的初版设计复杂得多，是 Matthias Koch（@mecrisp）大幅简化后，才得到上面这个相当容易理解的设计。

不过还没有完全完成，现在需要修改状态机。它将增加 `LOAD` 和 `WAIT_DATA` 两个状态：

```verilog
   localparam FETCH_INSTR = 0;
   localparam WAIT_INSTR  = 1;
   localparam FETCH_REGS  = 2;
   localparam EXECUTE     = 3;
   localparam LOAD        = 4;
   localparam WAIT_DATA   = 5;
   reg [2:0] state = FETCH_INSTR;
```

_注 1：_其实可以使用更少的状态，但当前目标是先做出一个能工作且尽可能易于理解的版本。后面会介绍如何简化状态机。

_注 2：_别忘了检查 `state` 是否具有足够的位数！（现在是 `reg [2:0] state`，不能再像以前那样用 `reg [1:0] state`！！）然后按如下方式接入新状态：

```verilog
     ...
	   EXECUTE: begin
	      if(!isSYSTEM) begin
		 PC <= nextPC;
	      end
	      state <= isLoad ? LOAD : FETCH_INSTR;
	   end
	   LOAD: begin
	      state <= WAIT_DATA;
	   end
	   WAIT_DATA: begin
	      state <= FETCH_INSTR;
	   end

     ...
```

最后，按如下方式驱动 `mem_addr`（保存待读地址）和 `mem_rstrb`（处理器要读取数据时拉高）信号：

```verilog
   assign mem_addr = (state == WAIT_INSTR || state == FETCH_INSTR) ?
		     PC : loadstore_addr ;
   assign mem_rstrb = (state == FETCH_INSTR || state == LOAD);
```

现在用下面的程序测试新指令：
```verilog
   integer L0_   = 8;
   integer wait_ = 32;
   integer L1_   = 40;

   initial begin
      LI(s0,0);
      LI(s1,16);
   Label(L0_);
      LB(a0,s0,400); // LEDs are plugged on a0 (=x10)
      CALL(LabelRef(wait_));
      ADDI(s0,s0,1);
      BNE(s0,s1, LabelRef(L0_));
      EBREAK();

   Label(wait_);
      LI(t0,1);
      SLLI(t0,t0,slow_bit);
   Label(L1_);
      ADDI(t0,t0,-1);
      BNEZ(t0,LabelRef(L1_));
      RET();

      endASM();

      // Note: index 100 (word address)
      //     corresponds to
      // address 400 (byte address)
      MEM[100] = {8'h4, 8'h3, 8'h2, 8'h1};
      MEM[101] = {8'h8, 8'h7, 8'h6, 8'h5};
      MEM[102] = {8'hc, 8'hb, 8'ha, 8'h9};
      MEM[103] = {8'hff, 8'hf, 8'he, 8'hd};
   end
```
这个程序在地址 400 处的四个字中初始化了一些值，然后在循环中把它们加载到 `a0`。与之前一样，程序还包含一个延时循环（`wait` 函数），让你能够看清变化。

**动手试试**：分别在仿真和真实设备上运行程序。测试其他指令，并像第 3 步那样制作一串可编程彩灯。

**你到这里了！**只剩三条指令，马上就能完成！

| ALUreg | ALUimm | Jump  | Branch | LUI | AUIPC | Load  | Store | SYSTEM |
|--------|--------|-------|--------|-----|-------|-------|-------|--------|
| [*] 10 | [*] 9  | [*] 2 | [*] 6  | [*] | [*]   | [*] 5 | [ ] 3 | [*] 1  |

## 第 16 步：存储

我们快到终点了，但还要完成最后一些工作，实现以下三条指令：

| 指令 | 效果 |
|------|------|
| SW(rs2,rs1,imm) | 把 rs2 存入地址 rs1+imm |
| SB(rs2,rs1,imm) | 把 rs2 的最低 8 位存入地址 rs1+imm |
| SH(rs2,rs1,imm) | 把 rs2 的最低 16 位存入地址 rs1+imm |

为此需要完成三件事：
- 修改处理器与存储器之间的接口，使处理器能够写入存储器；
- 存储器按字寻址，每次写操作都会修改一个字。但 `SB` 和 `SH` 需要能够单独写入字节。因此除了要写入的字，还要计算这个字中哪些字节应当真正被修改（一个 4 位掩码）；
- 修改状态机。

`Memory` 模块修改如下：

``` verilog
module Memory (
   input             clk,
   input      [31:0] mem_addr,
   output reg [31:0] mem_rdata,
   input   	     mem_rstrb,
   input      [31:0] mem_wdata,
   input      [3:0]  mem_wmask
);

   reg [31:0] MEM [0:255];

   initial begin
      ...
   end

   wire [29:0] word_addr = mem_addr[31:2];
   always @(posedge clk) begin
      if(mem_rstrb) begin
         mem_rdata <= MEM[word_addr];
      end
      if(mem_wmask[0]) MEM[word_addr][ 7:0 ] <= mem_wdata[ 7:0 ];
      if(mem_wmask[1]) MEM[word_addr][15:8 ] <= mem_wdata[15:8 ];
      if(mem_wmask[2]) MEM[word_addr][23:16] <= mem_wdata[23:16];
      if(mem_wmask[3]) MEM[word_addr][31:24] <= mem_wdata[31:24];
   end
```


我们增加了两个输入信号：`mem_wdata` 是一个 32 位信号，保存要写入的值；`mem_wmask` 是一个 4 位信号，指明要写入哪些字节。

_注：_你可能会好奇它在实践中如何实现，尤其是对存储器的掩码写入如何在设备上综合。大多数 FPGA 的 BRAM 都通过厂商专用原语直接支持掩码写入。Yosys 有一个非常聪明的特殊步骤，称为“技术映射”（technology mapping）：它检测 VERILOG 源文件中的特定模式，并实例化最适合相应用途的厂商原语。

事实上，本教程此前已经用技术映射实现了寄存器堆：每个周期都要读取 `rs1` 和 `rs2` 两个寄存器。IceStick 的 BRAM 每个时钟周期只能读取一个值，因此 Yosys 会自动复制寄存器堆。每当有值写入 `rd` 时，它会同时写入两个寄存器堆：`bank1[rdId] <- writeBackValue; bank2[rdId] <- writeBackValue;`。这样，同一周期就能从各自的寄存器堆读取两个不同的寄存器：`rs1 <- bank1[rs1Id]; rs2 <- bank2[rs2Id];`。

借助 Yosys 的魔法，无需亲自处理这些细节；它会自动选择最佳映射方式（复制寄存器堆；若目标支持，则使用带两个读端口的单个寄存器堆；对于拥有大量 LUT 的大型 FPGA，甚至可以使用带地址译码器的触发器阵列）。本例的 IceStick 使用 Ice40HX1K，拥有 8 kB BRAM，分成 8 个各 1 kB 的块。其中两块用于（复制后的）寄存器堆，余下 6 kB BRAM 用于综合系统 RAM。

相应地更新 `Processor` 模块：
```verilog
module Processor (
    input 	      clk,
    input 	      resetn,
    output [31:0]     mem_addr,
    input [31:0]      mem_rdata,
    output 	      mem_rstrb,
    output [31:0]     mem_wdata,
    output [3:0]      mem_wmask,
    output reg [31:0] x10 = 0
);
```

（并在 `SOC` 中连接所有信号。）

下面看看如何计算要写入的字和掩码。写入地址仍为 `rs1 + imm`，但 `Load` 使用的立即数格式是 `Iimm`，`Store` 使用的是 `Simm`：
```
   wire [31:0] loadstore_addr = rs1 + (isStore ? Simm : Iimm);
```

接下来，要写入的数据取决于写入的是字节、半字还是字；对于字节和半字，还取决于地址的最低两位。有趣的是，我们无需测试写入的是字节、半字还是字，因为稍后介绍的写掩码会在写字节和半字时忽略最高位：
```
   assign mem_wdata[ 7: 0] = rs2[7:0];
   assign mem_wdata[15: 8] = loadstore_addr[0] ? rs2[7:0]  : rs2[15: 8];
   assign mem_wdata[23:16] = loadstore_addr[1] ? rs2[7:0]  : rs2[23:16];
   assign mem_wdata[31:24] = loadstore_addr[0] ? rs2[7:0]  :
			     loadstore_addr[1] ? rs2[15:8] : rs2[31:24];
```

最后是 4 位写掩码，用来指明 `mem_wdata` 中哪些字节应当真正写入存储器。其取值如下：

| 写掩码 | 指令 |
|--------|------|
| `4'b1111` | `SW` |
| `4'b0011` 或 `4'b1100` | `SH`，取决于 `loadstore_addr[1]` |
| `4'b0001`、`4'b0010`、`4'b0100` 或 `4'b1000` | `SB`，取决于 `loadstore_addr[1:0]` |

推导这个表达式略显痛苦。Matthias Koch 和我最终得到了下面的写法：

```verilog
   wire [3:0] STORE_wmask =
	      mem_byteAccess      ?
	            (loadstore_addr[1] ?
		          (loadstore_addr[0] ? 4'b1000 : 4'b0100) :
		          (loadstore_addr[0] ? 4'b0010 : 4'b0001)
                    ) :
	      mem_halfwordAccess ?
	            (loadstore_addr[1] ? 4'b1100 : 4'b0011) :
              4'b1111;
```

现在为状态机增加一些状态：
```verilog
   localparam FETCH_INSTR = 0;
   localparam WAIT_INSTR  = 1;
   localparam FETCH_REGS  = 2;
   localparam EXECUTE     = 3;
   localparam LOAD        = 4;
   localparam WAIT_DATA   = 5;
   localparam STORE       = 6;

   ...

   always @(posedge clk) begin
   ...
       case(state)
           ...
	   EXECUTE: begin
	      if(!isSYSTEM) begin
		 PC <= nextPC;
	      end
	      state <= isLoad  ? LOAD  :
		       isStore ? STORE :
		       FETCH_INSTR;
	   LOAD: begin
	      state <= WAIT_DATA;
	   end
	   WAIT_DATA: begin
	      state <= FETCH_INSTR;
	   end
	   STORE: begin
	      state <= FETCH_INSTR;
	   end
	 endcase
      end
   end
```

按如下方式驱动与存储器相接的信号：
```verilog
   assign mem_addr = (state == WAIT_INSTR || state == FETCH_INSTR) ?
		     PC : loadstore_addr ;
   assign mem_rstrb = (state == FETCH_INSTR || state == LOAD);
   assign mem_wmask = {4{(state == STORE)}} & STORE_wmask;
```

最后还有一个小细节：如果指令是 `Store`，不要回写寄存器堆！
```verilog
   assign writeBackEn = (state==EXECUTE && !isBranch && !isStore && !isLoad) ||
			(state==WAIT_DATA) ;
```
_注：_条件中用于防止在 `EXECUTE` 阶段写入 `rd` 的 `!isLoad` 项可以移除，因为紧接着在 `WAIT_DATA` 阶段，`rd` 就会被覆盖。保留它只是为了让仿真更容易理解。

**动手试试**：分别在仿真和真实设备上运行 [step16.v](step16.v)。它会把 16 个字节从地址 400 复制到地址 800，然后显示复制后的字节值。

**你到这里了！**恭喜！你已经完成了自己的第一颗 RV32I RISC-V 内核！

| ALUreg | ALUimm | Jump  | Branch | LUI | AUIPC | Load  | Store | SYSTEM |
|--------|--------|-------|--------|-----|-------|-------|-------|--------|
| [*] 10 | [*] 9  | [*] 2 | [*] 6  | [*] | [*]   | [*] 5 | [*] 3 | [*] 1  |

_可等一下，_为了实现 RISC-V 内核，我们确实付出了很多努力，但现在看到的东西还是像第 1 步那个傻乎乎的闪灯器！我想看点更厉害的！

为此，需要让设备通过比 5 个 LED 更丰富的方式与外部世界通信。

## 第 17 步：存储器映射设备——做点（远远）超越闪灯器的事情！

现在的思路是在 SOC 中加入设备。我们已经有 LED，并把它们连接到寄存器 `a0`（`x10`）。直接把设备接到寄存器上不算优雅。更好的方法是分配一个特殊的存储器地址：它并不对应真正的 RAM，而是对应一个连接到 LED 的寄存器。采用这种方式，只需给每个设备分配一个虚拟地址，就能加入任意数量的设备。SOC 中的地址译码硬件会把数据路由到正确的设备。你会看到，除了移除处理器中从 `x10` 引向 LED 的连线，只需对 SOC 作少量修改。

开始修改 SOC 前，首先要确定“存储器映射”（memory map），也就是地址空间的哪一部分对应什么。系统中有 6 kB RAM，所以实践中可以把 0 到 2^13-1 之间的地址（8 kB，为了方便取 2 的幂）视为 RAM。我决定为 RAM 保留更大的地址空间（因为其他 FPGA 可能拥有更多 BRAM），因此 RAM 专用地址空间设为 0 到 2^22-1，也就是 4 MB。

接着约定：如果地址的第 22 位为 1，这个地址就对应设备。然后还要规定如何在多个设备中进行选择。一种自然的想法是把第 0 到 21 位用作“设备索引”，但这样需要多个 22 位宽的比较器，会吃掉 IceStick 剩余 LUT 中相当大的一部分。Matthias Koch（@mecrisp）再一次提出了更好的方法：使用独热编码（1-hot encoding），也就是地址的第 `n` 位为 1 时，把数据路由到第 `n` 号设备。我们只考虑“字地址”（即忽略最低两位）。这样 SOC 最多只能连接 20 个不同设备，但仍远超实际需求。它的优点是大幅简化地址译码，使所有内容仍能装进 IceStick。

为了确定存储器请求应该路由到 RAM 还是设备，在 SOC 中插入以下电路：
```verilog
   wire [31:0] RAM_rdata;
   wire [29:0] mem_wordaddr = mem_addr[31:2];
   wire isIO  = mem_addr[22];
   wire isRAM = !isIO;
   wire mem_wstrb = |mem_wmask;
```

RAM 按如下方式连接：
```verilog
   Memory RAM(
      .clk(clk),
      .mem_addr(mem_addr),
      .mem_rdata(RAM_rdata),
      .mem_rstrb(isRAM & mem_rstrb),
      .mem_wdata(mem_wdata),
      .mem_wmask({4{isRAM}}&mem_wmask)
   );
```
（注意，`isRAM` 信号与写掩码执行了 AND 运算。）

现在可以加入连接 LED 的逻辑。在 SOC 模块接口中，把 LED 声明为 `reg`：
```verilog
module SOC (
    input 	     CLK,
    input 	     RESET,
    output reg [4:0] LEDS,
    input 	     RXD,
    output 	     TXD
);
```

再由一个简单的块进行驱动：
```verilog
   localparam IO_LEDS_bit = 0;

   always @(posedge clk) begin
      if(isIO & mem_wstrb & mem_wordaddr[IO_LEDS_bit]) begin
	 LEDS <= mem_wdata;
      end
   end
```

现在可以编写那个熟悉闪灯器的又一个版本：
```verilog
      LI(gp,32'h400000);
      LI(a0,0);
   Label(L1_);
      SW(a0,gp,4);
      CALL(LabelRef(wait_));
      ADDI(a0,a0,1);
      J(LabelRef(L1_));
```

首先把 IO 页的基地址（即 `2^22`）载入 `gp`。要写入 LED 值，就把 `a0` 存入 IO 页的 1 号字地址（也就是地址 4）。为了稍后连接多个设备时更方便，先编写几个辅助函数：

```verilog
   // Memory-mapped IO in IO page, 1-hot addressing in word address.
   localparam IO_LEDS_bit      = 0;  // W five leds

   // Converts an IO_xxx_bit constant into an offset in IO page.
   function [31:0] IO_BIT_TO_OFFSET;
      input [31:0] bit;
      begin
	 IO_BIT_TO_OFFSET = 1 << (bit + 2);
      end
   endfunction
```

然后可以按如下方式写入 LED：

```verilog
   SW(a0,gp,IO_BIT_TO_OFFSET(IO_LEDS_bit));
```

_好吧，就这？教程都走了 17（！）步，还是那个傻乎乎的闪灯器？_

你说得对，朋友。让我们加入 UART，使内核能够在虚拟终端中显示内容。IceStick（以及许多其他 FPGA 开发板）带有一颗专用芯片（如果你想知道，是 FTDI2232H），负责在古老朴素的 RS232 串行协议与 USB 之间转换。这对我们是个好消息，因为 RS232 协议很简单，实现起来远比 USB 容易。实际上，内核会通过两个引脚与外部世界通信：一个发送数据，称为 `TXD`；另一个接收数据，称为 `RXD`。FTDI 芯片会替你完成到 USB 协议的转换。

此外，没必要重复造轮子，已经有许多用 VERILOG 编写的 UART（Universal Asynchronous Receiver Transmitter，通用异步收发器，用于实现 RS232 协议）实现。目前我们只需要其中一半：让处理器通过它发送数据，以便在终端模拟器中显示文本。

Olof Kindgren 编写了一个[推文大小的 UART](https://twitter.com/OlofKindgren/status/1409634477135982598)，[这里](https://gist.github.com/olofk/e91fba2572396f55525f8814f05fb33d)还有可读性更好的版本。

把它插入 SOC 并连接起来：

```verilog
   // Memory-mapped IO in IO page, 1-hot addressing in word address.
   localparam IO_LEDS_bit      = 0;  // W five leds
   localparam IO_UART_DAT_bit  = 1;  // W data to send (8 bits)
   localparam IO_UART_CNTL_bit = 2;  // R status. bit 9: busy sending
   ...

   wire uart_valid = isIO & mem_wstrb & mem_wordaddr[IO_UART_DAT_bit];
   wire uart_ready;

   corescore_emitter_uart #(
      .clk_freq_hz(`BOARD_FREQ*1000000),
      .baud_rate(115200)
   ) UART(
      .i_clk(clk),
      .i_rst(!resetn),
      .i_data(mem_wdata[7:0]),
      .i_valid(uart_valid),
      .o_ready(uart_ready),
      .o_uart_tx(TXD)
   );

   wire [31:0] IO_rdata =
	       mem_wordaddr[IO_UART_CNTL_bit] ? { 22'b0, !uart_ready, 9'b0}
	                                      : 32'b0;

   assign mem_rdata = isRAM ? RAM_rdata :
	                      IO_rdata ;

```

UART 映射到存储器空间中的两个不同地址。第一个地址只能写入，用于发送一个字符；第二个地址只能读取，用于指示 UART 已就绪（第 9 位 = 0）还是正在忙于发送字符（第 9 位 = 1）。

现在，处理器与外部世界通信的手段终于不再局限于那可怜的五个 LED！让我们实现一个发送字符的函数：

```verilog
   Label(putc_);
      // Send character to UART
      SW(a0,gp,IO_BIT_TO_OFFSET(IO_UART_DAT_bit));
      // Read UART status, and loop until bit 9 (busy sending)
      // is zero.
      LI(t0,1<<9);
   Label(putc_L0_);
      LW(t1,gp,IO_BIT_TO_OFFSET(IO_UART_CNTL_bit));
      AND(t1,t1,t0);
      BNEZ(t1,LabelRef(putc_L0_));
      RET();
```

它把字符写入 IO 空间中映射的 UART 地址，然后在 UART 状态指示它正忙于发送字符时不断循环等待。

**动手试试**：在仿真中运行 [step17.v](step17.v)。

_等一下，_仿真时它怎么知道该如何显示内容？

这是因为我稍微作弊了一下，在 SOC 中加入了下面这段代码：
```verilog
`ifdef BENCH
   always @(posedge clk) begin
      if(uart_valid) begin
	 $write("%c", mem_wdata[7:0] );
	 $fflush(32'h8000_0001);
      end
   end
`endif
```
（传给 `$fflush()` 的神奇常量对应 `stdout`。必须这样做，否则要等到 `stdout` 输出缓冲区写满后，终端上才会显示内容。）采用这种方式时，仿真并没有测试 UART，而是完全绕过了它。我相信 Olof 的实现能正常工作；但如果要严谨一些，最好把某个模块接到仿真的 `TXD` 信号上，解码 RS232 协议并显示字符（后面会看到这种仿真的示例）。

**动手试试**：在真实设备上运行 [step17.v](step17.v)。

要显示 UART 发送的内容，请运行：
```
  $ ./terminal.sh
```
_说明：_
- 编辑 `terminal.sh`，在其中选择你喜欢的终端模拟器。你可能还要根据本地配置修改 `DEVICE=/dev/ttyUSB1`；
- 如果显示乱码，请尝试发送 break（在 picocom 中按 `<ctrl><A>`，再按 `<ctrl><\>`）；
- 默认波特率为 `115200`，足以用于基础测试和演示。后面的一些演示通过向 tty 发送 ANSI 序列来绘制“图形”，流畅运行时会更有趣。为此可以尝试更高的波特率（例如 `1000000`）。需要同时修改 `emitter_uart.v` 和 `terminal.sh`（适用于大多数 FPGA；GOWIN 开发板使用 BL702 而不是传统 FTDI 芯片，可能因此例外）。

## 第 18 步：计算曼德博集合

现在，我们已经有了一颗能工作的 RISC-V 处理器，以及一个带 UART、能向虚拟终端发送字符的 SOC。让我们用一个纯软件步骤稍微休息一下。本步将编写一个 RISC-V 汇编程序，计算粗略的 ASCII 艺术版曼德博集合。

我们的“图像”由 80×80 个字符组成。先写一个用“*”字符填满图像的程序。为此使用两个嵌套循环：Y 坐标存入 `s0`，X 坐标存入 `s1`，上限（80）存入 `s11`。程序如下：

```verilog
      LI(gp,32'h400000); // IO page
      LI(s1,0);
      LI(s11,80);

   Label(loop_y_);
      LI(s0,0);

   Label(loop_x_);
      LI(a0,"*");
      CALL(LabelRef(putc_));

      ADDI(s0,s0,1);
      BNE(s0,s11,LabelRef(loop_x_));

      LI(a0,13);
      CALL(LabelRef(putc_));
      LI(a0,10);
      CALL(LabelRef(putc_));

      ADDI(s1,s1,1);
      BNE(s1,s11,LabelRef(loop_y_));

      EBREAK();
```
（并从上一个示例复制 `putc` 函数。）

**定点数：**现在要计算曼德博集合，需要处理实数。遗憾的是，我们这颗极其简单的 RISC-V 内核无法直接处理浮点数。C 编译器的支持库 `libgcc` 中有一些支持浮点数的函数，但要到后面才会介绍如何使用。目前的思路是使用定点数计算曼德博集合：在一个整数中，用若干位表示小数部分（本例为 10 位），其余若干位表示整数部分（本例为 22 位）。换句话说，如果要表示实数 `x`，就在寄存器中保存 `x*2^10` 的整数部分。这与浮点数类似，只不过我们的“指数”始终为 10。程序中将使用以下常量：

```verilog
   `define mandel_shift 10
   `define mandel_mul (1 << `mandel_shift)
```

计算两个数的和或差时不需要改变什么，因为参与加法（或减法）的两个数都有相同的 `2^10` 因子。但乘法不同：计算 `x*y` 时，实际算的是 `x*2^10*y*2^10`，结果为 `(x*y)*2^20`，而我们想要的是 `(x*y)*2^10`，因此还要除以 `2^10`（右移 `10` 位）。

很好，可怎样计算两个寄存器中整数的乘积？处理器没有 `MUL` 指令！其实可以添加 `MUL`（它属于 RV32M 指令集），后面会介绍，但它装不进小小的 IceStick！那该怎么办？可以实现一个函数：从 `a0` 和 `a1` 接收两个数，计算乘积，再通过 `a0` 返回。C 编译器支持库 `libgcc` 中就有这样一个函数；为我们这种不带 `MUL` 指令的小型 RV32I RISC-V 处理器编译 C 时，使用的正是它。函数源代码在[这里](https://github.com/riscv-collab/riscv-gcc/blob/5964b5cd72721186ea2195a7be8d40cfe6554023/libgcc/config/riscv/muldi3.S)。

让我们把它移植到 VERILOG RISC-V 汇编器中（遗憾的是，两者语法略有不同；后面会介绍如何直接使用 gcc 和 gas）：

```verilog
      // Mutiplication routine,
      // Input in a0 and a1
      // Result in a0
   Label(mulsi3_);
      MV(a2,a0);
      LI(a0,0);
   Label(mulsi3_L0_);
      ANDI(a3,a1,1);
      BEQZ(a3,LabelRef(mulsi3_L1_));
      ADD(a0,a0,a2);
   Label(mulsi3_L1_);
      SRLI(a1,a1,1);
      SLLI(a2,a2,1);
      BNEZ(a1,LabelRef(mulsi3_L0_));
      RET();
```
（别忘了在 `initial` 块之前声明新标签。）

在显示曼德博集合之前，先用一个更简单的图形测试定点计算思路。设想要把 `[-2.0,2.0]x[-2.0,2.0]` 的正方形映射到 30×30 字符的显示区域，并显示一个以 `(0,0)` 为圆心、半径为 `2` 的圆盘。首先要计算定点坐标 `x,y`，分别存入 `s2` 和 `s3`；然后计算 `x^2+y^2`，方法是调用两次 `mulsi3` 子程序（别忘了把结果右移 10 位）。最后，把结果与 `4 << 10` 比较：4 是_半径的平方_，左移 10 位则是因为使用定点表示。根据比较结果判断点位于圆盘内还是圆盘外，并用不同字符显示。相应程序如下：

```verilog
   `define mandel_shift 10
   `define mandel_mul (1 << `mandel_shift)
   `define xmin (-2*`mandel_mul)
   `define xmax ( 2*`mandel_mul)
   `define ymin (-2*`mandel_mul)
   `define ymax ( 2*`mandel_mul)
   `define dx ((`xmax-`xmin)/30)
   `define dy ((`ymax-`ymin)/30)
   `define norm_max (4 << `mandel_shift)

   integer    loop_y_      = 28;
   integer    loop_x_      = 36;
   integer    in_disk_     = 92;

   initial begin
      LI(gp,32'h400000); // IO page

      LI(s1,0);
      LI(s3,`xmin);
      LI(s11,30);
      LI(s10,`norm_max);

   Label(loop_y_);
      LI(s0,0);
      LI(s2,`ymin);

   Label(loop_x_);

      MV(a0,s2);
      MV(a1,s2);
      CALL(LabelRef(mulsi3_));
      SRLI(s4,a0,`mandel_shift); // s4 = x*x
      MV(a0,s3);
      MV(a1,s3);
      CALL(LabelRef(mulsi3_));
      SRLI(s5,a0,`mandel_shift); // s5 = y*y
      ADD(s6,s4,s5);             // s6 = x*x+y*y
      LI(a0,"*");
      BLT(s6,s10,LabelRef(in_disk_)); // if x*x+y*y < 4
      LI(a0," ");
  Label(in_disk_);
      CALL(LabelRef(putc_));

      ADDI(s0,s0,1);
      ADDI(s2,s2,`dx);
      BNE(s0,s11,LabelRef(loop_x_));

      LI(a0,13);
      CALL(LabelRef(putc_));
      LI(a0,10);
      CALL(LabelRef(putc_));

      ADDI(s1,s1,1);
      ADDI(s3,s3,`dy);
      BNE(s1,s11,LabelRef(loop_y_));

      EBREAK();
```

输出如下：
```
          ***********
        ***************
       ******************
     *********************
    ***********************
    ************************
   *************************
  ***************************
  ***************************
 *****************************
 *****************************
 *****************************
 *****************************
 *****************************
 *****************************
 *****************************
 *****************************
 *****************************
 *****************************
 *****************************
  ***************************
  ***************************
   *************************
   *************************
    ***********************
     *********************
      *******************
        ***************
          ***********
```

要计算曼德博集合，需要迭代执行以下操作：
```
   Z <- 0; iter <- 0
   do
      Z <- Z^2 + C
      iter <- iter + 1
   while |Z| < 2
```
其中 `Z` 和 `C` 是复数，`C = x + iy` 对应当前像素。根据复数乘法规则（`i*i = -1`），可以计算 `Z^2 = (Zr + i*Zi)^2 = Zr^2-Zi^2 + 2*i*Zr*Zi`。计算这些迭代的循环写法如下：
```verilog
   Label(loop_Z_);
      MV(a0,s4); // Zrr  <- (Zr*Zr) >> mandel_shift
      MV(a1,s4);
      CALL(LabelRef(mulsi3_));
      SRLI(s6,a0,`mandel_shift);
      MV(a0,s4); // Zri <- (Zr*Zi) >> (mandel_shift-1)
      MV(a1,s5);
      CALL(LabelRef(mulsi3_));
      SRAI(s7,a0,`mandel_shift-1);
      MV(a0,s5); // Zii <- (Zi*Zi) >> (mandel_shift)
      MV(a1,s5);
      CALL(LabelRef(mulsi3_));
      SRLI(s8,a0,`mandel_shift);
      SUB(s4,s6,s8); // Zr <- Zrr - Zii + Cr
      ADD(s4,s4,s2);
      ADD(s5,s7,s3); // Zi <- 2Zri + Cr

      ADD(s6,s6,s8); // if norm > norm max, exit loop
      LI(s7,`norm_max);
      BGT(s6,s7,LabelRef(exit_Z_));

      ADDI(s10,s10,-1);  // iter--, loop if non-zero
      BNEZ(s10,LabelRef(loop_Z_));

   Label(exit_Z_);
```

最后，根据退出循环时 `iter`（`s10`）的值显示不同字符：
```
   Label(exit_Z_);
      LI(a0,colormap_);
      ADD(a0,a0,s10);
      LBU(a0,a0,0);
      CALL(LabelRef(putc_));
```
其中，“色表”是一组模拟不同“强度”的字符，从最暗排列到最亮：
```
   Label(colormap_);
      DATAB(" ",".",",",":");
      DATAB(";","o","x","%");
      DATAB("#","@", 0 , 0 );
```

![](https://github.com/BrunoLevy/learn-fpga/blob/master/FemtoRV/TUTORIALS/Images/mandelbrot_terminal.gif)

**动手试试**：分别在仿真和真实设备上运行 [step18.v](step18.v)。修改程序，绘制自己的图形（例如尝试用“色表”绘制“同心圆”）。

## 第 19 步：使用 Verilator 加速仿真

正如第 18 步中看到的，仿真比设计在真实设备上运行慢得多。不过还有一种名为 `verilator` 的工具，可以把 VERILOG 设计转换成 C++。编译生成的 C++ 后，就能得到远快于 icarus/iverilog 的仿真。先安装 verilator：
```
  $ apt-get install verilator
```

在把设计转换成 C++ 之前，要先创建一个“bench”，也就是一段为设计生成信号并声明 C++ `main()` 函数的 C++ 代码。`main` 函数的主要作用是声明一个 `VSOC` 类对象（由 `SOC` 模块生成），并不断翻转它的 `CLK` 信号。每次改变 `CLK` 后，都要调用 `eval()` 函数使变化生效。`sim_main.cpp` 文件如下：

```c++
#include "VSOC.h"
#include "verilated.h"
#include <iostream>

int main(int argc, char** argv, char** env) {
   VSOC top;
   top.CLK = 0;
   while(!Verilated::gotFinish()) {
      top.CLK = !top.CLK;
      top.eval();
   }
   return 0;
}
```

此外，[sim_main.cpp](sim_main.cpp) 中还有一些代码，用于检测 LED 何时发生变化并显示其状态。

使用以下命令把设计转换成 C++：
```
  $ verilator -DBENCH -DBOARD_FREQ=12 -Wno-fatal --top-module SOC -cc -exe sim_main.cpp step18.v
```

然后编译 C++ 并运行生成的程序：
```
  $ cd obj_dir
  $ make -f VSOC.mk
  $ ./VSOC
```

可以看到，它比 icarus/iverilog 快得多！对于小型设计，差别可能不算特别大；但相信我，当你开发一颗带 FPU 的 RV32IMFC 内核时，高效仿真实在太重要了！

为方便起见，这里提供了 `run_verilator.sh` 脚本，可以这样调用：
```
  $ run_verilator.sh step18.v
```

## 第 20 步：使用 GNU 工具链编译程序——汇编

走到这一步，你可能仍觉得我们的 RISC-V 设计只是个教育玩具，离“真家伙”还差得远。事实上，从这一步开始，你会感觉到自己做出的东西与任何其他 RISC-V 处理器一样真实！处理器的价值取决于它能运行的软件；如果这个小家伙能运行任何为（RV32I）RISC-V 处理器编写的软件，那它就是一颗 RV32I RISC-V 处理器。

_等一下，_此前编写软件一直使用 VERILOG 汇编器。它不就是个玩具，和真正的工具不一样吗？

实际上，VERILOG 汇编器生成的机器码与任何其他 RISC-V 汇编器完全相同。我们完全可以改用任意 RISC-V 汇编器，把生成的机器码载入设计，再运行它！

为此，VERILOG 提供了 `$readmemh()` 命令，可以从外部文件加载存储器初始化数据。在 [step20.v](step20.v) 中，其用法如下：

```verilog
   initial begin
       $readmemh("firmware.hex",MEM);
   end
```

其中 `firmware.hex` 是一个 ASCII 文件，以十六进制形式保存 `MEM` 的初始内容。

所以，要使用外部汇编器，只需弄清以下几件事：
- 如何使用 GNU 工具编译 RISC-V 汇编代码；
- 如何把我们所创建设备的信息告诉 GNU 工具（RAM 起始地址、RAM 容量）；
- 如何把 GNU 工具的输出转换成 `$readmemh()` 能够理解的文件。

先从 [blinker.S](FIRMWARE/blinker.S) 中的一个简单闪灯器开始：

```
# Simple blinker

.equ IO_BASE, 0x400000
.equ IO_LEDS, 4

.section .text

.globl start

start:
        li   gp,IO_BASE
	li   sp,0x1800
.L0:
	li   t0, 5
	sw   t0, IO_LEDS(gp)
	call wait
	li   t0, 10
	sw   t0, IO_LEDS(gp)
	call wait
	j .L0

wait:
        li t0,1
	slli t0, t0, 17
.L1:
        addi t0,t0,-1
	bnez t0, .L1
	ret
```

可以看到，它与此前使用 VERILOG 汇编器编写的代码非常相似。这个程序包含三个不同部分：
- **主程序**；
- **实用函数**，这里是 `wait` 函数；
- **启动代码**，也就是初始化 `gp` 和 `sp`。

因此把文件拆成三部分：
- [FIRMWARE/blinker.S](FIRMWARE/blinker.S)，包含 `main` 函数；
- [FIRMWARE/wait.S](FIRMWARE/wait.S)，包含 `wait` 函数；
- [FIRMWARE/start.S](FIRMWARE/start.S)，包含启动代码，并在最后调用 `main`。

要进行编译，需要在机器上安装 RISC-V 工具链（编译器、汇编器、链接器）。我们的 makefile 可以替你完成：

```
  $ cd learn-fpga/FemtoRV
  $ make ICESTICK.firmware_config
```
_注：_即使你使用更大的开发板，也始终执行 `ICESTICK.firmware_config`。它会把 makefile 配置为构建 `RV32I`（这正是我们的处理器所支持的配置）。

该命令会下载一些文件，并解压到 `learn-fpga/FemtoRV/FIRMWARE/TOOLCHAIN`。
把 `riscv64-unknown-elf-gcc..../bin/` 目录加入 PATH。

现在编译程序：
```
  $ cd learn-fpga/FemtoRV/TUTORIALS/FROM_BLINKER_TO_RISCV/FIRMWARE
  $ riscv64-unknown-elf-as -march=rv32i -mabi=ilp32 -mno-relax start.S -o start.o
  $ riscv64-unknown-elf-as -march=rv32i -mabi=ilp32 -mno-relax blinker.S -o blinker.o
  $ riscv64-unknown-elf-as -march=rv32i -mabi=ilp32 -mno-relax wait.S -o wait.o
```
这里指定了与处理器所支持指令对应的架构（`rv32i`），以及与函数调用方式对应的 ABI（`ilp32`）。`no-relax` 选项与我们用来访问 IO 页的 `gp` 寄存器有关，它可以防止汇编器将 `gp` 用于其他用途。

这些命令会生成目标文件（`.o`）。接下来调用链接器，由这些目标文件生成可执行文件。链接器将决定代码和数据应放在存储器的什么位置。为此，需要在链接脚本 [FIRMWARE/bram.ld](FIRMWARE/bram.ld) 中说明设备的存储器组织方式：

```
MEMORY
{
   BRAM (RWX) : ORIGIN = 0x0000, LENGTH = 0x1800  /* 6kB RAM */
}
SECTIONS
{
    everything :
    {
	. = ALIGN(4);
	start.o (.text)
        *(.*)
    } >BRAM
}
```

链接脚本包含一段 `MEMORY` 描述。本例只有一个 6 kB 的存储器段，称为 `BRAM`，从地址 `0x0000` 开始。随后是 `SECTIONS`，用于说明哪些内容放在哪里（也就是哪个段放入哪块存储器）。本例极其简单：所有内容都进入 BRAM。我们还规定 `start.o` 的内容必须最先放入存储器。调用链接器的命令如下：

```
  $ riscv64-unknown-elf-ld blinker.o wait.o -o blinker.bram.elf -T bram.ld -m elf32lriscv -nostdlib -norelax
```

它会生成一个 ELF 可执行文件（ELF 是 Executable and Linkable Format，可执行与可链接格式的缩写），与 Linux 系统中的二进制文件格式相同。选项 `-T bram.ld` 指定使用我们的链接脚本；`-m elf32lriscv` 表示生成 32 位可执行文件。目前不使用 C 标准库（`-nostdlib`），并保留 `gp` 自用（`-norelax`）。命令行的待链接目标文件列表中不必写 `start.o`，因为链接脚本 `bram.ld` 已经包含了它。

还没有完全结束。现在需要从 ELF 可执行文件中提取相关信息，生成一个以十六进制保存全部机器码的文件，供 VERILOG 的 `$readmemh()` 函数读取。为此，我编写了 `firmware_words` 工具；它能理解 ELF 文件格式，提取我们需要的部分，并以 ASCII 十六进制形式写出：

```
  $ make blinker.bram.hex
```

_注：_可以直接运行 `make xxxx.bram.hex`，它会自动调用汇编器、链接器和 ELF 转换工具。

现在可以分别在仿真和真实设备上运行示例：
```
  $ cd ..
  $ ./run_verilator.sh step20.v
  $ BOARDS/run_xxx.sh step20.v
```

现在事情变得容易了，可以编写更复杂的程序。下面看看如何编写著名的“Hello, world”程序。我们需要一个 `putstring` 子程序，在 tty 上显示字符串。它从 `a0` 接收待显示字符串首字符的地址。只需遍历字符串中的所有字符，遇到空字符时退出循环，并为每个字符调用 `putchar`：
```
# Warning, buggy code ahead !
putstring:
	mv t2,a0
.L2:    lbu a0,0(t2)
	beqz a0,.L3
	call putchar
	addi t2,t2,1
	j .L2
.L3:    ret
```
看到注释了吗？它表示上面的代码有错误，你能找出来吗？

提示：`putstring` 是一个会调用其他函数的函数。这种情况下是不是要做一些特殊处理？

还记得 `call` 和 `ret` 的作用吗？没错，`call` 先把 `PC+4` 存入 `ra`，再跳转到函数；`ret` 则跳到 `ra` 中的地址。假设有人调用了 `putstring` 函数。进入函数时，`ra` 保存着 `putstring` 执行到 `ret` 时应跳回的地址。但在 `putstring` 内部，我们又调用了 `putchar`；该调用会用紧跟在调用之后的地址覆盖 `ra`，让 `putchar` 返回时能够跳到那里。问题是，`putstring` 最后也会跳到这个地址，这可不是我们想要的。为避免这种情况，需要在 `putstring` 开头保存 `ra`，并在末尾恢复它。使用栈即可完成：

```
putstring:
	addi sp,sp,-4 # save ra on the stack
	sw ra,0(sp)   # (need to do that for functions that call functions)
	mv t2,a0
.L2:    lbu a0,0(t2)
	beqz a0,.L3
	call putchar
	addi t2,t2,1
	j .L2
.L3:    lw ra,0(sp)  # restore ra
	addi sp,sp,4 # resptore sp
	ret
```

函数可以这样使用：
```
   la   a0, hello
   call putstring

   ...

hello:
	.asciz "Hello, world !\n"
```

`la`（load address，加载地址）伪指令把字符串地址载入 `a0`。字符串用普通标签和 `.asciz` 指令声明，后者会生成以零结尾的字符串。

**动手试试**：编译 `hello.S`（`cd FIRMWARE; make hello.bram.hex`），并分别在仿真和真实设备上测试。也试试 `mandelbrot.S`。可以看到，[FIRMWARE/mandelbrot.S](FIRMWARE/mandelbrot.S) 中并没有 `__mulsi` 函数。查看 [FIRMWARE/Makefile](FIRMWARE/Makefile) 会发现，可执行文件链接了包含该函数的正确版本 `libgcc.a`（面向 RV32I）。

现在，你开始能够感受到这颗处理器是个真家伙了：运行曼德博示例时，_别人_编写的代码正在_你的_处理器上执行。还能不能更进一步，运行标准工具生成的代码？

## 第 21 步：使用 GNU 工具链编译程序——C

下面看看如何为处理器编写 C 代码。此时我们已经能够生成目标文件（`.o`），并用链接器把它们组合成 ELF 可执行文件。链接脚本确保所有内容都被放到存储器中的正确位置；随后处理器就能执行这些代码：首先执行放在地址 0 的 `start.S` 内容，再由它调用 `main` 函数。此前的程序全部使用汇编编写。第 13、14 步介绍的 ABI（应用程序二进制接口）妙就妙在：只要遵循 ABI，就可以组合不同工具生成的目标文件（`.o`），C 编译器生成的文件当然也不例外。

取自 picorv 示例的 [FIRMWARE/sieve.c](FIRMWARE/sieve.c) 很适合用来测试。它很有意思，会对整数执行乘法、除法和取模。RV32I 内核并未实现这些运算，但编译器可以借助 `libgcc.a` 中的函数提供支持；由于我们会链接 `libgcc.a`，所以它们能够正常工作。不过，该程序还使用 `printf()` 显示结果，而这个函数声明在 `libc.a` 中。原则上可以直接使用，但 `printf()` 支持的格式太多，代码过于庞大，装不进 6 kB RAM。因此，我们在 [FIRMWARE/print.c](FIRMWARE/print.c) 中提供了一个小得多、简单得多的版本（同样取自 picorv），并将其加入要与可执行文件链接的目标文件中。

![](mandel_and_riscvlogo.png)

另外还有两个示例。其一是曼德博程序的 C 版本 [FIRMWARE/mandel_C.c](FIRMWARE/mandel_C.c)，它使用 [ANSI 颜色](https://stackoverflow.com/questions/4842424/list-of-ansi-color-escape-sequences)在终端中显示低分辨率“图形”。另一个是 [FIRMWARE/riscv_logo.c](FIRMWARE/riscv_logo.c)，用于显示旋转的 RISC-V 标志（颇有 90 年代 demoscene 风格！）。

**动手试试**：编译 `sieve.c`（`cd FIRMWARE; make sieve.bram.hex`），分别在仿真（`./run_verilator.sh step20.v`）和真实设备（`BOARDS/run_xxx.sh step20.v; ./terminal.sh`）上测试。也试试其他程序，再编写自己的程序（如果没有灵感，可以尝试元胞自动机、生命游戏等）。

注：Verilator 框架可以在仿真中直接加载 ELF 可执行文件，无需重新生成 `firmware.hex`。可以生成所有演示程序：`cd FIRMWARE; make hello.bram.elf mandelbrot.bram.elf mandel_C.bram.elf riscv_logo.bram.elf;cd ..`，然后使用 `./run_verilator.sh step20.v FIRMWARE/mandel_C.bram.elf` 或 `./obj_dir/FIRMWARE/mandel_C.bram.elf` 运行想要的程序。

现在可以看出，你的处理器不只是玩具，而是一颗真正的 RISC-V 处理器，可以运行标准工具生成的程序！

_注：_IceStick 只有 `6kB` RAM，因此只能容纳很小的程序。如果编译后的程序超过 `6kB`，就会报错。更棘手的情况是程序几乎填满整个 BRAM：此时栈几乎没有空间，会覆盖其他内容，使 CPU 进入无效状态，很可能直接卡死。遇到这种情况时很难理解和调试，因此只要生成代码占用超过 BRAM 的 95%，`firmware_words` 就会显示醒目的警告消息。

## 第 22 步：存储数据——能否拥有超过 6 kB 的存储器？

_以及处理器中的一些优化_

![](IceStick_SPIFLASH.jpg)

IceStick 只有 8 块各 1 kB 的 BRAM，其中两块要用于寄存器，因此程序只能使用剩下的 6 kB RAM。这足以容纳曼德博之类的小程序或小型图形演示，但很快就会碰到上限。IceStick 上有一颗小芯片（见图），带有 4 MB FLASH 存储器；其他开发板也有类似芯片。综合后的设计就存放在这块 FLASH 中。启动时，FPGA 会从该芯片加载配置。好消息是，FPGA 配置只占几千字节，因此还留下大量空间可用于存放自己的数据。不过，要与这颗芯片通信，还需要创建一些额外硬件。

从图中可以看到，这颗芯片只有 8 个引脚。仅靠 8 个引脚，怎样寻址 4 MB 数据？实际上，它采用_串行协议_（SPI）。访问数据时，通过一个引脚逐位发送待读地址，然后芯片再通过另一个引脚逐位返回数据。若想进一步了解，可以阅读我关于 SPI flash 的[笔记](https://github.com/BrunoLevy/learn-fpga/blob/master/FemtoRV/TUTORIALS/spi_flash.md)；VERILOG 实现在 [spi_flash.v](spi_flash.v) 中。它会根据使用的引脚数量以及引脚是否双向，支持不同协议。

`MappedSPIFlash` 模块具有以下接口：
```verilog
module MappedSPIFlash(
    input wire 	       clk,
    input wire 	       rstrb,
    input wire [19:0]  word_address,

    output wire [31:0] rdata,
    output wire        rbusy,

    output wire        CLK,
    output reg         CS_N,
    inout  wire [1:0]  IO
);
```

| 信号 | 说明 |
|------|------|
| clk | 系统时钟 |
| rstrb | 读选通；处理器要读取一个字时拉高 |
| word_address | 待读字的地址 |
| rdata | 从存储器读取的数据 |
| rbusy | 正忙于接收数据时置位 |
| CLK | SPI flash 芯片的时钟引脚 |
| CS_N | SPI flash 芯片的片选引脚，低电平有效 |
| IO | 用于发送和接收数据的两个双向引脚 |

现在要修改 SOC，让某些地址对应 SPI flash。首先需要决定如何把它映射到处理器的存储器空间。思路是使用存储器地址的第 23 位选择 SPI flash，第 22 位则用于 IO（LED、UART）。此外，访问 IO 时还要检查第 23 位是否为零；若第 23 位和第 22 位都为零，则访问 BRAM。这样，存储器空间根据第 23、22 位分成四个“象限”，我们使用其中三个。

用以下信号区分不同的存储器区域：
```verilog
   wire isSPIFlash  = mem_addr[23];
   wire isIO        = mem_addr[23:22] == 2'b01;
   wire isRAM       = mem_addr[23:22] == 2'b00;
```

`MappedSPIFlash` 模块按如下方式连接：
```verilog
   wire SPIFlash_rdata;
   wire SPIFlash_rbusy;
   MappedSPIFlash SPIFlash(
      .clk(clk),
      .word_address(mem_wordaddr),
      .rdata(SPIFlash_rdata),
      .rstrb(isSPIFlash & mem_rstrb),
      .rbusy(SPIFlash_rbusy),
      .CLK(SPIFLASH_CLK),
      .CS_N(SPIFLASH_CS_N),
      .IO(SPIFLASH_IO)
   );
```
（引脚 `SPIFLASH_CLK`、`SPIFLASH_CS_N`、`SPIFLASH_IO[0]` 和 `SPIFLASH_IO[1]` 在 `BOARDS` 子目录中的约束文件里声明。）

发送给处理器的数据经过一个三选一多路复用器：
```verilog
   assign mem_rdata = isRAM      ? RAM_rdata :
                      isSPIFlash ? SPIFlash_rdata :
	                           IO_rdata ;
```

好了，现在处理器只要访问地址中第 23 位为 1 的存储器，就能自动触发 SPI flash 读取。但它怎么知道数据已经就绪？（别忘了，数据是逐位到达的。）当 `MappedSPIFlash` 正忙于接收数据时，`SPIFlash_rbusy` 会拉高，因此处理器状态机必须考虑这个信号。为处理器增加一个新的输入信号 `mem_rbusy`，并按如下方式修改状态机：
```verilog
   ...
   WAIT_DATA: begin
      if(!mem_rbusy) begin
	 state <= FETCH_INSTR;
      end
   end
   ...
```

然后在 SOC 中，把该信号连接到 `SPIFlash_rbusy`：
```verilog
   wire mem_rbusy;
   ...
   Processor CPU(
     ...
     .mem_rbusy(mem_rbusy),
     ...
   );
   ...
   assign mem_rbusy = SPIFlash_rbusy;
```

顺便说一下，既然正在重构状态机，还可以做一些改进。还记得状态机的这一部分吗？是不是可以更快？
```verilog
   WAIT_INSTR: begin
      instr <= mem_rdata;
      state <= FETCH_REGS;
   end
   FETCH_REGS: begin
      rs1 <= RegisterBank[rs1Id];
      rs2 <= RegisterBank[rs2Id];
      state <= EXECUTE;
   end
```

没错，`rs1Id` 和 `rs2Id` 分别只是从 `instr` 引出的 5 根线，因此可以直接从 `mem_rdata` 中取得它们，并在 `WAIT_INSTR` 状态读取寄存器：
```verilog
   WAIT_INSTR: begin
      instr <= mem_rdata;
      rs1 <= RegisterBank[mem_rdata[19:15]];
      rs2 <= RegisterBank[mem_rdata[24:20]];
      state <= EXECUTE;
   end
```
这样每条指令都能节省一个周期，轻松获益！

还有一件事：为什么需要单独的 `LOAD` 和 `STORE` 状态？难道不能在 `EXECUTE` 状态发起存储器传输吗？当然可以，因此要相应修改写掩码和读选通信号：
```verilog
   assign mem_rstrb = (state == FETCH_INSTR || (state == EXECUTE & isLoad));
   assign mem_wmask = {4{(state == EXECUTE) & isStore}} & STORE_wmask;
```

这样状态机就只剩 4 个状态！
```verilog
   localparam FETCH_INSTR = 0;
   localparam WAIT_INSTR  = 1;
   localparam EXECUTE     = 2;
   localparam WAIT_DATA   = 3;
   reg [1:0] state = FETCH_INSTR;
   always @(posedge clk) begin
      if(!resetn) begin
	 PC    <= 0;
	 state <= FETCH_INSTR;
      end else begin
	 if(writeBackEn && rdId != 0) begin
	    RegisterBank[rdId] <= writeBackData;
	 end
	 case(state)
	   FETCH_INSTR: begin
	      state <= WAIT_INSTR;
	   end
	   WAIT_INSTR: begin
	      instr <= mem_rdata;
	      rs1 <= RegisterBank[mem_rdata[19:15]];
	      rs2 <= RegisterBank[mem_rdata[24:20]];
	      state <= EXECUTE;
	   end
	   EXECUTE: begin
	      if(!isSYSTEM) begin
		 PC <= nextPC;
	      end
	      state <= isLoad  ? WAIT_DATA : FETCH_INSTR;
	   end
	   WAIT_DATA: begin
	      if(!mem_rbusy) begin
		 state <= FETCH_INSTR;
	      end
	   end
	 endcase
      end
   end
```

还有其他一些可优化之处。首先，你可能已经注意到，RV32I 指令的最低两位始终是 `2'b11`，因此无需加载它们：
```verilog
   reg [31:2] instr;
   ...
   instr <= mem_rdata[31:2];
   ...
   wire isALUreg  =  (instr[6:2] == 5'b01100);
   ...
```

另外，所有地址计算都使用 32 位，而地址空间实际上只有 24 位，因此可以在这里节省大量资源：
```verilog
   localparam ADDR_WIDTH=24;
   wire [ADDR_WIDTH-1:0] PCplusImm = PC + ( instr[3] ? Jimm[31:0] :
				  instr[4] ? Uimm[31:0] :
				             Bimm[31:0] );
   wire [ADDR_WIDTH-1:0] PCplus4 = PC+4;

   wire [ADDR_WIDTH-1:0] nextPC = ((isBranch && takeBranch) || isJAL) ? PCplusImm   :
	                                  isJALR   ? {aluPlus[31:1],1'b0} :
	                                             PCplus4;

   wire [ADDR_WIDTH-1:0] loadstore_addr = rs1 + (isStore ? Simm : Iimm);
```

最新的 Verilog 文件位于 [step22.v](step22.v)。现在用下面这个[程序](FIRMWARE/read_spiflash.c)检查处理器能否访问 SPI flash：
```C
#include "io.h"
#define SPI_FLASH_BASE ((char*)(1 << 23))
int main()  {
   for(int i=0; i<16; ++i) {
      IO_OUT(IO_LEDS,i);
      int lo = (int)SPI_FLASH_BASE[2*i  ];
      int hi = (int)SPI_FLASH_BASE[2*i+1];
      print_hex_digits((hi << 8) | lo,4); // print four hexadecimal digits
      printf(" ");
   }
   printf("\n");
}
```

SPI flash 被映射到存储器空间中，使用第 23 位为 1 的地址（我们称为 `SPI_FLASH_BASE` 的首地址就是 `1 << 23`）。随后逐个访问字节，并把它们组合成 16 位字显示出来。由于 RISC-V 采用小端序，每个字在存储器中的第一个字节是最低有效字节。[FIRMWARE/print.c](FIRMWARE/print.c) 中的 `print_hex_digits()` 函数负责完成显示（第二个参数指定每个数字要打印多少个十六进制字符）。

现在按如下方式编译程序、综合设计并发送到设备：

```
 $ cd FIRMWARE
 $ make read_spiflash.bram.hex
 $ cd ..
 $ BOARDS/run_icestick.sh step22.v
 $ ./terminal.sh
```

……然后什么也看不到。为什么？因为程序在终端启动前就执行完了，所以没能看到输出。不过可以按下那个看不见的复位按钮（[第 2 步](README.md#第-2-步更慢的闪灯器)曾提到）来复位处理器。每按一次“按钮”，终端就会显示 SPI flash 中存储的前 16 个字。在 IceStick 上，输出大致如下：
```
00FF FF00 AA7E 7E99 0051 0501 0092 6220 4B01 0072 8290 0000 0011 0101 0000 0000
```

知道这些值来自哪里吗？想想 FPGA 开发板上为什么会有 SPI flash 芯片：设计就存放在那里。FPGA 启动时，会从 SPI flash 加载设计。这个设计对应 `yosys/nextpnr/icepack` 流水线末尾生成的 `SOC.bin` 文件：
- `yosys` 把 Verilog 转换成“电路”，也称“网表”；
- `nextpnr` 把电路中的门映射到 FPGA 的逻辑单元；
- 最后，`icepack` 把结果转换成 FPGA 能直接理解的“二进制流”。

查看二进制流的前 16 个字：

```
  $ od -x -N 32 SOC.bin
```

将看到类似下面的内容：
```
0000000 00ff ff00 aa7e 7e99 0051 0501 0092 6220
0000020 4b01 0072 8290 0000 0011 0101 0000 0000
0000040
```

这与刚才从 SPI flash 芯片读取并显示在终端中的内容一致。也就是说，CPU 可以从 SPI flash 读取自己在 FPGA 中的表示，就像生物学家测序自己的 DNA！虽然这种递归感既美妙又耐人寻味，但实用价值大概很有限。不过再仔细看看：`SOC.bin` 文件并不大：

```
$ ls -al SOC.bin
-rw-rw-r-- 1 blevy blevy 32220 Jan  7 07:31 SOC.bin
```

它只有约 `32KB`，而 SPI flash 芯片容量为 `4MB`，可用空间非常充裕！唯一要注意的是不能覆盖 FPGA 配置，也就是说，存储数据的起点必须超过 `SOC.bin` 的大小。因此，我们使用 `1MB` 偏移量存放数据（你可能觉得从 `32KB` 到 `1MB` 之间浪费了大量空间，但教程后续步骤会把这片空间用于其他用途）。

**动手试试**：创建文本文件 `hello.txt`，把它写入 FPGA 的 `1MB` 偏移处（方法见下文），再编写程序显示所存文件。为了知道何时停止，可以约定一个终止字符，或者预先编码文件长度。

对于 ICE40 开发板（IceStick、IceBreaker 等），使用：
```
 $ iceprog -o 1M hello.txt
```

对于 ECP5 开发板（ULX3S），使用：
```
 $ cp hello.txt hello.img
 $ ujprog -j flash -f 1048576 hello.img
```
（使用从 [https://github.com/kost/fujprog](https://github.com/kost/fujprog) 编译的最新版 `ujprog`。）


![](ST_NICCC_tty.png)

好了，现在可以利用新增的存储空间做些更有意思的事情。我们要在终端上显示一段动画。这是一个来自 90 年代的演示，会把多边形数据流式传送给软件多边形渲染器。多边形数据是一个 640 kB 的二进制文件，位于 `learn_fpga/FemtoRV/FIRMWARE/EXAMPLES/DATA/scene1.dat`（有关文件格式的更多信息，请查看同一目录中的其他文件）。首先从 1 MB 偏移处把文件写入 SPI flash。对于基于 ICE40 的开发板（IceStick、IceBreaker），使用：

```
 $ iceprog -o 1M learn_fpga/FemtoRV/FIRMWARE/EXAMPLES/DATA/scene1.dat
```

对于 ECP5 开发板（ULX3S），使用：
```
 $ cp learn_fpga/FemtoRV/FIRMWARE/EXAMPLES/DATA/scene1.dat scene1.img
 $ ujprog -j flash -f 1048576 scene1.img
```
（使用从 [https://github.com/kost/fujprog](https://github.com/kost/fujprog) 编译的最新版 `ujprog`。）

现在可以编译程序：
```
 $ cd FIRMWARE
 $ make ST_NICCC.bram.hex
 $ cd ..
```
并把设计和程序发送到设备：
```
 $ BOARDS/run_xxx.sh step22.v
 $ ./terminal.sh
```
**动手试试**：在 SPI Flash 中存储一幅图像（使用便于读取的格式），并编写程序显示它。可以用 `printf("\033[48;2;%d;%d;%dm ",R,G,B);` 发送一个像素（其中 `R`、`G`、`B` 是 0 到 255 之间的数），并在每条扫描线末尾调用 `printf("\033[48;2;0;0;0m\n");`。

## 第 23 步：从 SPI Flash 运行程序——初步尝试

完成上一步后，我们已经能从 SPI flash 加载数据，数据存储空间也相当充裕，但代码和变量仍要共享区区 6 kB，实在不多！如果能用 SPI flash 存储代码并直接从那里执行，就太棒了。6 kB 已经能容纳不错的演示程序；试想一下，如果代码可以使用 2 MB，而全部 6 kB 都留给变量，又能做出什么！

要从 SPI flash 加载代码，只需在 `mem_rbusy` 变为零之前一直停留在 `WAIT_INSTR` 状态，也就是在把 `state` 改为 `EXECUTE` 前测试 `mem_rbusy`：

```verilog
   WAIT_INSTR: begin
      instr <= mem_rdata[31:2];
      rs1 <= RegisterBank[mem_rdata[19:15]];
      rs2 <= RegisterBank[mem_rdata[24:20]];
      if(!mem_rbusy) begin
	 state <= EXECUTE;
      end
   end
```

然后使用以下程序初始化 BRAM；该程序会跳转到地址 `0x00820000`：

```verilog
   initial begin
      LI(a0,32'h00820000);
      JR(a0);
   end
```

该地址等于 SPI flash 映射到 CPU 地址空间中的地址（`0x00800000` = 1 << 23）加上 128 kB（`0x20000`）偏移量。之所以必须保留 128 kB 偏移，是因为别忘了：SPI Flash 还与 FPGA 共享，用于存储它的配置！

硬件部分基本完成。现在看看能否从那里执行代码。为此需要新的链接脚本 [FIRMWARE/spiflash0.ld](FIRMWARE/spiflash0.ld)：

```
MEMORY {
   FLASH (RX)  : ORIGIN = 0x00820000, LENGTH = 0x100000 /* 4 MB in flash */
}
SECTIONS {
    everything : {
	. = ALIGN(4);
	start.o (.text)
        *(.*)
    } >FLASH
}
```

它与之前的链接脚本类似，不过这次告诉链接器把所有内容放入 flash 存储器（暂时如此；稍后再讨论全局变量如何处理）。使用一个不会写入全局变量的程序来测试，例如 [FIRMWARE/hello.S](FIRMWARE/hello.S)。用新链接脚本进行链接：
```
  $ riscv64-unknown-elf-ld -T spiflash0.ld -m elf32lriscv -nostdlib -norelax hello.o putchar.o -o hello.spiflash0.elf
```

不过这条命令输入起来很繁琐，因此 Makefile 已将它自动化：
```
  $ make hello.spiflash0.elf
```

现在把 ELF 可执行文件转换成平坦二进制文件：
```
  $ riscv64-unknown-elf-objcopy hello.spiflash0.elf hello.spiflash0.bin -O binary
```

也可以使用 Makefile：
```
  $ make hello.spiflash0.bin
```

再把它写入 SPI flash 的 128 kB 偏移处：
```
  $ iceprog -o 128k hello.spiflash0.bin
```

也可以使用 Makefile：
```
  $ make hello.spiflash0.prog
```

然后运行：
```
  $ ./terminal.sh
```

## 第 24 步：从 SPI Flash 运行程序——更完善的链接脚本

开始前，先对内核作一个小改动：按下复位按钮时，CPU 会跳到地址 0，而该地址最初保存着一条跳转到 flash 存储器的指令。但程序执行后，RAM 很可能已被挪作他用，不再保留这条跳转指令。为解决这个问题，可以让 CPU 每次在复位信号变低时直接跳到 flash 存储器：

```verilog
   if(!resetn) begin
     PC    <= 32'h00820000;
     state <= WAIT_DATA;
   end
```

请注意，这里把状态设为 `WAIT_DATA`，让处理器先等待 `mem_rbusy` 变低，再执行其他操作。

好了，现在有了大量 flash 存储器，可以把代码安装在那里并直接运行。只读变量也可以放进去，比如上一个示例中的字符串 `.asciz "Hello, world !\n"`。局部变量怎么办？它们分配在栈上，而栈位于现有的 6 kB RAM 中，所以没有问题。系统怎么知道栈在哪里？别忘了，我们编写过 [FIRMWARE/start.S](FIRMWARE/start.S)，它把 `sp` 初始化到 RAM 末尾（`0x1800`），这样就足够了。

但下面这样的程序又该如何工作？
```C
  int x = 3;
  void main() {
     x = x + 1;
     printf("%d\n",x);
  }
```

全局变量 `x` 有一个初始值，需要存放在某处，所以要把它放进 flash 存储器；但之后又会修改它，因此运行时必须放在 RAM。怎样兼顾这两点？实际上，需要一套机制：把所有已初始化全局变量的初始值存入 flash，并在启动时复制到 RAM。为此要准备新的链接脚本（指定变量和其初始值分别放在哪里），以及新的 `start.S`（把初始值复制给变量）。下面看看如何实现。

编译 C 代码时，编译器会插入指令，说明不同内容应放入哪些“节”（section）。可以从某个 C 程序生成汇编并查看：
```
$ cd FIRMWARE
$ make ST_NICCC.o
$ readelf -S ST_NICCC.o
```

它会显示目标文件中存在的各个节。

| 节 | 说明 |
|----|------|
| text | 可执行代码 |
| bss, sbss | 未初始化数据 |
| data, sdata | 可读写数据 |
| rodata | 只读数据 |

未初始化数据使用 `bss` 作为节名有其历史原因，可追溯到 20 世纪 60 年代（BSS，即 Block Started by Symbol，是 IBM 704 汇编器的一条伪指令）。未初始化和已初始化数据节各有两种形式：`sbss` 和 `sdata` 分别用于较小的未初始化数据和已初始化数据。

`readelf` 输出中还有 `type` 字段。`PROGBIT` 表示需要从文件加载一些数据（对应 `text`、`data` 和 `rodata` 段），`NOBITS` 表示无需加载数据（对应 `bss`）。`Addr` 指示该节将映射到存储器中的位置（对 `.o` 文件而言始终为 0，但对链接后的 ELF 可执行文件很有用，可以用 `readelf` 查看）。`Offs` 字段指示节数据在 `.o` 文件中的偏移量，`Size` 字段则给出该节的字节数。

因此，要编写一个链接脚本，说明以下事项：
- `text` 节放入 flash 存储器；
- `bss` 节放入 BRAM；
- `data` 节放入 BRAM，但其初始值存放在 flash 存储器中。

`text` 和 `bss` 已经知道如何处理。对于 `data`，链接脚本可以指定 LMA（Load Memory Address，加载存储地址），说明初始值应存放在哪里。链接脚本大致如下：

```
  MEMORY {
      FLASH (rx)  : ORIGIN = 0x00820000, LENGTH = 0x100000
      RAM   (rwx) : ORIGIN = 0x00000000, LENGTH = 0x1800
  }
  SECTIONS {

    .data: AT(address_in_spi_flash) {
      *(.data*)
      *(.sdata*)
    } > RAM

    .text : {
      start_spiflash1.o(.text)
      *(.text*)
      *(.rodata*)
      *(.srodata*)
    } >FLASH

    .bss : {
      *(.bss*)
      *(.sbss*)
    } >RAM
  }
```

每个节都说明如何把从目标文件读取的节映射到可执行文件中的节（`.data`、`.text` 和 `.bss`），以及如何把这些节映射到 flash 存储器和 BRAM。每个节内部都有一些模式匹配规则，指出目标文件中的哪些节属于该范围。对于 `.text`，要确保第一个节是 `start_spiflash1.o` 的 text 节，因为处理器复位时会跳到那里。另请注意，只读数据（`.rodata` 和 `.srodata`）也放入 flash。

对于 `.data`，`AT` 关键字指定 LMA（加载存储地址），链接器会把初始值放在该处（SPI flash 中的地址）；而每当程序引用 `data` 或 `sdata` 节中的符号时，链接器都会使用它在 RAM 中的地址。

但还有一个问题：系统怎么知道应该把初始化数据从 flash 复制到 BRAM？怎么知道具体地址？又如何把未初始化数据（BSS）清零？事实上，这些都需要在启动代码 `start_spiflash1.S` 中手工完成：

```asm
.equ IO_BASE, 0x400000

.text
.global _start
.type _start, @function

_start:
.option push
.option norelax
     li  gp,IO_BASE
.option pop

     li   sp,0x1800

# zero-init bss section:
     la a0, _sbss
     la a1, _ebss
     bge a0, a1, end_init_bss
loop_init_bss:
     sw zero, 0(a0)
     addi a0, a0, 4
     blt a0, a1, loop_init_bss
end_init_bss:

# copy data section from SPI Flash to BRAM:
     la a0, _sidata
     la a1, _sdata
     la a2, _edata
     bge a1, a2, end_init_data
loop_init_data:
     lw a3, 0(a0)
     sw a3, 0(a1)
     addi a0, a0, 4
     addi a1, a1, 4
     blt a1, a2, loop_init_data
end_init_data:

     call main
     ebreak
```

- 首先初始化栈指针和全局指针 `gp`（本例中将其设为 IO 页地址）；
- 第一个循环清除 `_sbss` 到 `_ebss` 之间的存储器；
- 第二个循环把数据从 `_sidata` 复制到 `_sdata` ... `_edata`；
- 最后调用 `main`。

……但等一下，怎么知道 `_sbss`、`_ebss`、`_sidata`、`_sdata`、`_edata` 的值？

链接脚本可以替我们生成这些值。`.data` 节如下所示：

```
    .data : AT ( _sidata ) {
        . = ALIGN(4);
        _sdata = .;
        *(.data*)
        *(.sdata*)
        . = ALIGN(4);
        _edata = .;
    } > RAM
```

其中 `.` 表示当前地址。另外，`. = ALIGN(4);` 这样的语句可以确保地址始终按 4 字节边界对齐，因为 `start_spiflash1.S` 中的初始化循环依赖这一点。

`.text` 节的声明如下：

```
    .text : {
        . = ALIGN(4);
        start_spiflash1.o(.text)
        *(.text*)
        . = ALIGN(4);
        *(.rodata*)
        *(.srodata*)
        _etext = .;
        _sidata = _etext;
    } >FLASH
```

请注意，它恰好在 text 节末尾声明 `_sidata`，这样 `.data` 节就可以把初始化数据放在那里。

好了，用一个示例试试看：
```
  $ cd FIRMWARE
  $ make mandel_C.spiflash1.prog
  $ cd ..
  $ ./terminal.sh
```

成功了，但_等一下_，它明显比以前慢。能猜到原因吗？

FLASH 是*串行*存储器，也就是说，地址要逐位发送，结果也逐位返回（准确地说，本例中两者都是每次传输两位）。它比一个周期就能取得 32 位值的 BRAM 慢得多。能做些什么吗？当然可以！把一些关键函数放入 BRAM 如何？为此按如下方式修改链接脚本（最终结果见 [FIRMWARE/spiflash2.ld](FIRMWARE/spiflash2.ld)）：

```
    .data_and_fastcode : AT ( _sidata ) {
        . = ALIGN(4);
        _sdata = .;

	/* Initialized data */
        *(.data*)
        *(.sdata*)

	/* integer mul and div */
	*/libgcc.a:muldi3.o(.text)
	*/libgcc.a:div.o(.text)

	putchar.o(.text)
	print.o(.text)

	/* functions with attribute((section(".fastcode"))) */
	*(.fastcode*)

        . = ALIGN(4);
        _edata = .;
    } > RAM
```

这样就指定了某些特定函数（libgcc 中的整数乘除法函数以及 IO 函数）应放入高速 RAM。只需这么做！链接器会把这些函数的代码放入与已初始化变量的初始化数据相同的节中；运行时启动代码 `start_spiflash1.S` 会在启动时把它们连同初始化数据一起复制到 RAM。太棒了！

用示例试试看：

```
  $ cd FIRMWARE
  $ make mandel_C.spiflash2.prog
  $ cd ..
  $ ./terminal.sh
```

啊，这就好多了！

另请注意 `*(.fastcode*)` 这一行：把自己的函数标记为位于 `fastcode` 节，就能将其放入 BRAM。在 C 中可以这样写：

```C
 void my_function(my args ...) __attribute((section(".fastcode")));
 void my_function(my args ...) {
      ...
 }
```

**动手试试**：运行 `ST_NICCC` 演示（`make ST_NICCC.spiflash2.prog`）。然后在 `ST_NICCC.c` 中取消定义 `RV32_FASTCODE` 那一行的注释，并重新运行。

![](tinyraytracer_tty.png)

现在可以在设备上运行更大的程序：
- [FIRMWARE/pi.c](FIRMWARE/pi.c)（Fabrice Beillard 编写，用于计算圆周率的小数位）；
- [FIRMWARE/tinyraytracer.c](FIRMWARE/tinyraytracer.c)（Dmitry Sokolov 编写，实现光线追踪）。

二者都使用浮点数。对于我们这样的 RV32I 内核，浮点数运算要调用 `libgcc` 中实现的子程序。因此，可执行文件会更大（`pi` 为 17 kB，`tinyraytracer` 为 25 kB），过去不可能在 6 kB RAM 中运行。SPI FLASH 提供的额外存储器为设备带来了更多可能！

至此，设备不仅能运行标准工具（gcc）编译的代码，还能运行独立开发的现有代码（`libgcc` 中的数学子程序）。让现成的二进制代码运行在自己亲手创建的处理器上，实在令人兴奋！

## 下一篇教程

[流水线](PIPELINE.md)

## 各步骤文件

- [第 1 步](step1.v)：闪灯器——速度太快，什么也看不清
- [第 2 步](step2.v)：带 Clockworks 的闪灯器
- [第 3 步](step3.v)：从 ROM 加载图案的闪灯器
- [第 4 步](step4.v)：指令译码器
- [第 5 步](step5.v)：寄存器堆和状态机
- [第 6 步](step6.v)：ALU
- [第 7 步](step7.v)：使用 VERILOG 汇编器
- [第 8 步](step8.v)：跳转
- [第 9 步](step9.v)：分支
- [第 10 步](step10.v)：LUI 和 AUIPC
- [第 11 步](step11.v)：独立模块中的存储器
- [第 12 步](step12.v)：尺寸优化——不可思议的缩小内核！
- [第 13 步](step13.v)：子程序 1（标准 RISC-V 指令集）
- [第 14 步](step14.v)：子程序 2（使用 RISC-V 伪指令）
- [第 15 步](step15.v)：加载
- [第 16 步](step16.v)：存储
- [第 17 步](step17.v)：存储器映射设备
- [第 18 步](step18.v)：曼德博集合
- 第 19 步：使用 Verilator 加速仿真
- [第 20 步](step20.v)：使用 GNU 工具链编译汇编程序
- 第 21 步：使用 GNU 工具链编译 C 程序
- [第 22 步](step22.v)：更多存储器！使用 SPI Flash
- [第 23 步](step23.v)：从 SPI Flash 运行程序——初步尝试
- [第 24 步](step24.v)：从 SPI Flash 运行程序——更完善的链接脚本

_编写中_

- 第 25 步：更多设备（LED 点阵、OLED 屏幕……）

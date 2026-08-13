# 从闪灯器到 RISC-V（第二篇）——流水线

在[上一篇教程](README.md)中，我们学习了如何在 FPGA 上创建一颗功能完整的 RISC-V 处理器。这颗处理器效率不算高，每条指令需要 3 到 4 个周期。现代处理器借助各种技术，效率要高得多，甚至可以每个周期执行多条指令。本篇将介绍如何把这颗极其简单的处理器改造成效率更高的流水线处理器。

本篇需要一块至少拥有 128 kB BRAM 的 FPGA（例如 ULX3S），当然也可以完全在仿真中运行。

## 第 1 步：分离指令存储器和数据存储器

上一篇中的处理器 [step24.v](step24.v) 使用“统一存储器”，通过同一组连线访问程序和数据。流水线处理器的内部结构有所不同：程序存储器与数据存储器彼此独立。实际上，这些存储器通常是缓存，由连接外部世界的统一存储器总线填充。目前我还不知道缓存如何工作（留给后续步骤），所以为了简化问题，先在处理器中设置“程序 ROM”和“数据 RAM”（各 64 kB），并直接使用 `.hex` 文件初始化（稍后会介绍如何从 `ELF` 可执行文件生成这些 `.hex` 文件）：

```verilog
   reg [31:0] PROGROM [0:16383];
   reg [31:0] DATARAM [0:16383];   
   initial begin
      $readmemh("PROGROM.hex",PROGROM);
      $readmemh("DATARAM.hex",DATARAM);      
   end
```

- `PROGROM` 用于存放指令；
- `DATARAM` 用于存放变量。`LOAD` 和 `STORE` 指令能够读写该存储器，但不能访问程序存储器。

此前的存储器总线改为内部连线：

```verilog
   wire [31:0] mem_addr;
   wire [31:0] mem_rdata;
   wire [31:0] mem_wdata;
   wire [3:0]  mem_wmask;
```

与此前相比，`mem_rstrobe` 和 `mem_rbusy` 已经消失：内部存储器总会在下一周期给出 `mem_addr` 所指地址的数据。

为了与外部世界通信，处理器仍然为映射后的 IO 页保留一条存储器总线，用于和 `UART` 及其他设备交互：

```verilog
module Processor (
    ...
    output [31:0] IO_mem_addr,  
    input [31:0]  IO_mem_rdata, 
    output [31:0] IO_mem_wdata, 
    output 	  IO_mem_wr     
);
```

现在需要把所有内容路由到内部存储器总线。我们保留与之前相同的 IO 页（这样就能复用相同代码），由存储器地址的第 22 位指示：

```verilog
   wire isIO  = mem_addr[22];
   wire isRAM = !isIO;
```

数据 RAM 每个周期都会读取，并可选择写入：

```verilog
   wire [13:0] mem_word_addr = mem_addr[15:2];
   reg [31:0] dataram_rdata;
   wire [3:0] dataram_wmask = mem_wmask & {4{isRAM}};
   always @(posedge clk) begin
      dataram_rdata <= DATARAM[mem_word_addr];
      if(dataram_wmask[0]) DATARAM[mem_word_addr][ 7:0 ] <= mem_wdata[ 7:0 ];
      if(dataram_wmask[1]) DATARAM[mem_word_addr][15:8 ] <= mem_wdata[15:8 ];
      if(dataram_wmask[2]) DATARAM[mem_word_addr][23:16] <= mem_wdata[23:16];
      if(dataram_wmask[3]) DATARAM[mem_word_addr][31:24] <= mem_wdata[31:24];
   end
```

然后接入外部 IO 总线：

```verilog
   assign mem_rdata = isRAM ? dataram_rdata : IO_mem_rdata;
   assign IO_mem_addr  = mem_addr;
   assign IO_mem_wdata = mem_wdata;
   assign IO_mem_wr    = isIO & mem_wmask[0];
```

最后还要做两项简单修改：
- 在 `FETCH_INSTR` 状态从 `PROGROM` 取指：`instr <= PROGROM[PC[15:2]];`；
- 不再需要 `mem_rbusy` 信号（别忘了，`DATARAM` 和 `PROGROM` 都固定在一个周期内完成访问），因此状态机得以简化。代价是无法再像以前那样直接从 SPI flash 执行程序；等以后用_指令缓存_替换 `PROGROM` 后，便能再次做到。

更新后的 VERILOG 源码在这里：[pipeline1.v](pipeline1.v)

还需要为新内核编写软件。软件采用两个 ASCII 十六进制文件 `PROGROM.hex` 和 `DATARAM.hex` 的形式，分别保存程序存储器的内容和数据存储器的初始内容。首先创建新的链接脚本，描述存储器映射：

```
MEMORY {
   PROGROM (RX) : ORIGIN = 0x00000, LENGTH = 0x10000  /* 64kB ROM */
   DATARAM (RW) : ORIGIN = 0x10000, LENGTH = 0x10000  /* 64kB RAM */   
}
```
ROM 占据起始的 64 kB，后面紧接着是 RAM。

然后描述各个节，规定 text 段进入 `PROGROM`，其余内容进入 `DATARAM`：
```
SECTIONS {

    .text : {
        . = ALIGN(4);
	start_pipeline.o (.text)
        *(.text*)
    } > PROGROM

    .data : {
	. = ALIGN(4);
        *(.data*)          
        *(.sdata*)
        *(.rodata*) 
        *(.srodata*)
        *(.bss*)
        *(.sbss*)
	
        *(COMMON)
        *(.eh_frame)  
        *(.eh_frame_hdr)
        *(.init_array*)         
        *(.gcc_except_table*)  
    } > DATARAM
}
```

text 段以 `start_pipeline.S` 的内容开始：
```asm
.equ IO_BASE, 0x400000  
.section .text
.globl start
start:
        li   gp,IO_BASE
	li   sp,0x20000
	call main
	ebreak
	
```

使用这个链接脚本，可以生成一个 ELF 二进制文件：全部代码位于存储器起始 64 kB，数据位于随后的 64 kB。例如，下面是编译 Fabrice Bellard 所写圆周率小数位计算程序的方法：

```
$ cd FIRMWARE
$ riscv64-unknown-elf-gcc -Os -fno-pic -march=rv32i -mabi=ilp32 -fno-stack-protector -w -Wl,--no-relax   -c pi.c
$ riscv64-unknown-elf-as -march=rv32i -mabi=ilp32   start_pipeline.S -o start_pipeline.o 
$ riscv64-unknown-elf-as -march=rv32i -mabi=ilp32   putchar.S -o putchar.o 
$ riscv64-unknown-elf-as -march=rv32i -mabi=ilp32   wait.S -o wait.o 
$ riscv64-unknown-elf-gcc -Os -fno-pic -march=rv32i -mabi=ilp32 -fno-stack-protector -w -Wl,--no-relax   -c print.c
$ riscv64-unknown-elf-gcc -Os -fno-pic -march=rv32i -mabi=ilp32 -fno-stack-protector -w -Wl,--no-relax   -c memcpy.c
$ riscv64-unknown-elf-gcc -Os -fno-pic -march=rv32i -mabi=ilp32 -fno-stack-protector -w -Wl,--no-relax   -c errno.c
$ riscv64-unknown-elf-ld -T pipeline.ld -m elf32lriscv -nostdlib -norelax pi.o putchar.o wait.o print.o memcpy.o errno.o -lm libgcc.a -o pi.pipeline.elf
```

`Makefile` 可以替你完成这些步骤：
```
$ make pi.pipeline.elf
```

可以查看存储器映射：
```
$ readelf -a pi.pipeline.elf | more
```

你会看到 `.text` 和 `.data` 正好位于预期位置：
```
Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        00000000 000074 003fd8 00  AX  0   0  4
  [ 2] .data             PROGBITS        00010000 004050 0002c0 00  WA  0   0  8
```

要生成 `PROGROM.hex` 和 `DATARAM.hex`，可以使用 `firmware_words` 工具。它提供 `-from_addr` 和 `-to_addr` 参数，用于选择要写入 `.hex` 文件的存储器区间：

```
$ firmware_words pi.pipeline.elf -ram 0x20000 -max_addr 0x20000 -out pi.PROGROM.hex -from_addr 0 -to_addr 0xFFFF
$ firmware_words pi.pipeline.elf -ram 0x20000 -max_addr 0x20000 -out pi.DATARAM.hex -from_addr 0x10000 -to_addr 0x1FFFF
```

同样，`Makefile` 会替你完成这些工作，并把 `.hex` 文件复制到所需位置：

```
$ make clean
$ make pi.pipeline.hex
```
_（必须先执行 `make clean`，否则已有的 `.elf` 文件会让它混淆。）_

现在可以在仿真中运行 `pi`：

```
$ cd ..
$ ./run_verilator.sh pipeline1.v
```

也可以尝试其他程序（例如 `tinyraytracer`）：
```
$ cd FIRMWARE
$ make tinyraytracer.pipeline.hex
$ cd ..
$ ./run_verilator.sh pipeline1.v
```

如果有 ULX3S，把它连接到 USB 端口，然后运行：
```
$ BOARDS/run_ulx3s.sh pipeline1.v
$ ./terminal.sh
```

## 第 2 步：性能计数器

流水线的目的是提升性能，因此需要一种衡量性能的方法。RISC-V ISA 提供了一组特殊寄存器，称为 CSR（Control and Status Register，控制与状态寄存器）。CSR 有很多，用于控制中断、保护级别、虚拟存储器等。这里仅实现其中两个（或者说四个）：
- `CYCLE`：统计时钟周期数；
- `INSTRET`：统计已退役指令数。

有了它们，只需用 `CYCLE` 除以 `INSTRET`，即可轻松计算 CPI（cycles per instruction，每条指令的周期数）。

可以使用以下指令读取 `CYCLE` 和 `INSTRET`：
- `RDCYCLE`
- `RDCYCLEH`
- `RDINSTRET`
- `RDINSTRETH`

每个计数器各有两条读取指令，因为计数器宽 64 位。`RDCYCLE` 把最低 32 位读入 `rd`，`RDCYCLEH` 把最高 32 位读入 `rd`；`INSTRET` 也是如此。

实际上，这四条都是伪指令，全部由同一条指令 `CSRRS` 实现，编码如下：


| instr[31:20] | instr[19:15] | instr[14:12] | instr[11:7] | instr[6:0] |
|--------------|--------------|--------------|-------------|------------|
|   CSR Id     |     rs1      |  funct3      |     rd      |   SYSTEM   |
|              |   5'b00000   |  3'b010      |             | 7'b1110011 |


`CSRRS` 指令以原子方式把 CSR 的当前内容读入 `rd`，并根据 `rs1` 设置 CSR 中的位。如果 `rs1` 是 `x0`，它就只会把 CSR 复制到 `rd`（我们的四条 `RDXXX` 伪指令正是这种情况）。

- 指令字的最高 12 位编码相关 CSR 的 ID；
- 本例中的 `rs1` 始终为 `x0`；
- 对 `CSRRS` 而言，`funct3` 为 `3'b010`（还有其他操作 CSR 的指令，此处不实现）；
- `rd` 与往常一样使用指令字中的相同位；
- 操作码为 `SYSTEM`，与已经使用的 `EBREAK` 相同。我们通过 `funct3` 识别 `EBREAK`（对 `EBREAK` 而言，其值为 `3'b000`）。

需要识别的四个 CSR ID 如下：

| CSR     | Id  | Id (binary)  |
|---------|-----|--------------|
| CYCLE   | C00 | 110000000000 |
| CYCLEH  | C80 | 110010000000 |
| INSTRET | C02 | 110000000010 |
| INSTETH | C82 | 110010000010 |

在设计中声明两个新的 64 位寄存器。`cycle` 在每个时钟周期递增：

```verilog
   reg [63:0] cycle;   
   reg [63:0] instret;
   
   always @(posedge clk) begin
      cycle <= cycle + 1;
   end
```

每次从程序存储器取出一条指令时，`instret` 递增：

```verilog
     ...
     WAIT_INSTR: begin
        rs1 <= RegisterBank[instr[19:15]];
        rs2 <= RegisterBank[instr[24:20]];
        state <= EXECUTE;
        instret <= instret + 1;
     end
     ...
```     

指令译码器需要区分同样使用 `SYSTEM` 操作码的 `EBREAK` 和 `CSRRS`：

```verilog
   wire isEBREAK     = isSYSTEM & (funct3 == 3'b000);
   wire isCSRRS      = isSYSTEM & (funct3 == 3'b010);
```

然后按如下方式声明读出的 CSR 值：

```verilog
   wire [31:0] CSR_data =
	       ( instr[27] & instr[21]) ? instret[63:32]:
	       (!instr[27] & instr[21]) ? instret[31:0] :
	             instr[27]          ? cycle[63:32]  :
 	                                  cycle[31:0]   ;
```

（这里只检查 `instr` 中能够区分已实现四个 CSR 的位。）

最后，当译码出的指令为 `CSRRS` 时，把它路由到 `writeBackData`：

```verilog
   assign writeBackData = (isJAL || isJALR) ? PCplus4   :
			      isLUI         ? Uimm      :
			      isAUIPC       ? PCplusImm :
			      isLoad        ? LOAD_data :
			      isCSRRS       ? CSR_data  :
			                      aluOut;
```

最终的 VERILOG 设计在这里：[pipeline2.v](pipeline2.v)。

现在需要创建一些实用函数，让 C 程序能够轻松读取计数器。它们实现在 [FIRMWARE/perf.S](FIRMWARE/perf.S) 中。先看看 `rdcycle()`：

```asm
rdcycle:
.L0:  
   rdcycleh a1
   rdcycle a0
   rdcycleh t0
   bne a1,t0,.L0
   ret
```

需要了解两件事：
- RISC-V RV32 ABI 通过 `a1` 和 `a0` 返回 64 位值，其中最高 32 位位于 `a1`；
- 读取 64 位计数器需要两条指令，因此在读取过程中，最低 32 位可能发生回绕。按照 RISC-V 程序员手册的说明，可以读取两次最高位并进行比较；若不相等，就循环重试，直到二者一致。

请注意，该函数遵守 ABI，因此可以从 C 代码调用。我们在 [perf.h](perf.h) 中这样声明它：

```C
#include <stdint.h>
extern uint64_t rdcycle();
```

（`rdinstret()` 也是如此。）

现在用一个简单的 [FIRMWARE/test_rdcycle.c](FIRMWARE/test_rdcycle.c) 程序测试处理器性能：

```C
#include "perf.h"

int main() {
   for(int i=0; i<100; ++i) {
      uint64_t cycles = rdcycle();
      uint64_t instret = rdinstret();      
      printf("i=%d    cycles=%d     instret=%d\n", i, (int)cycles, (int)instret);
   }
   uint64_t cycles = rdcycle();
   uint64_t instret = rdinstret();      
   printf("cycles=%d     instret=%d    100CPI=%d\n", (int)cycles, (int)instret, (int)(100*cycles/instret));
   
}
```

现在可以编译程序、生成 ROM 和 RAM 初始内容，并启动仿真：

```
$ cd FIRMWARE
$ make test_rdcycle.pipeline.hex
$ cd ..
$ ./run_verilator.sh pipeline2.v
```

_注 1：_如果你好奇，`Makefile` 已经把 `perf.S` 加入编译和链接文件列表。

_注 2：_我们的 `printf()` 函数无法显示浮点值，因此显示的是 `100*CPI`，而非 `CPI`。

由此可知，对于这个简单循环，朴素 CPU 设计的 CPI 约为 `3.14`。接下来看看在 `tinyraytracer` 这样更贴近实际的程序中表现如何。[FIRMWARE/raystones.c](FIRMWARE/raystones.c) 中增加了计算 CPI 和“raystones”性能指标的代码。

看看结果：

```
$ cd FIRMWARE
$ make raystones.pipeline.hex
$ cd ..
$ ./run_verilator.sh pipeline2.v
```

程序会计算并显示一个简单的光线追踪场景，同时给出内核的 CPI 和“raystones”得分。

也可以在真实设备（ULX3S）上运行：
```
$ BOARDS/run_ulx3s.sh pipeline1.v
$ ./terminal.sh
```

该程序是 Dmitry Sokolov 的 [tinyraytracer](https://github.com/ssloy/tinyraytracer) 的 C 版本。光线追踪很适合用来对内核进行基准测试，因为它会大量使用浮点运算；这些运算可以由软件实现，也可以由 FPU 实现。

除了 CPI（每条指令的周期数），程序还会计算 CPU 的“raystones”得分。

“raystones”（`pixels/s/MHz` 或 `pixels/Mticks`）是衡量内核在光线追踪这一真实场景中浮点性能的有趣指标。我们的内核得到以下结果：

| CPI   | RAYSTONES |
|-------|-----------|
| 3.034 | 2.614     |

内核运行时的 CPI 略高于 3。大多数指令需要 3 个周期，只有加载指令需要 4 个。光线追踪偏重计算；对于更偏重数据访问的程序，平均 CPI 会更接近 4。

下表给出了几种常见内核的 raystones 性能：

 | 内核                 | 指令集     | raystones |
 |----------------------|------------|-----------|
 | serv                 | rv32i      |   0.111   |
 |                      |            |           |
 | picorv32-minimal     | rv32i      |   1.45    |
 | picorv32-standard    | rv32im     |   2.352   |
 |                      |            |           |
 | femtorv-quark        | rv32i      |   1.99    |
 | femtorv-electron     | rv32im     |   3.373   |
 | femtorv-gracilis     | rv32imc    |   3.516   |
 | femtorv-petitbateau  | rv32imfc   |  45.159   |
 |                      |            |           |
 | vexriscv imac        | rv32imac   |   7.987   |
 | vexriscv_smp         | rv32imafd  | 124.121   |

当前内核比 `femtorv-quark` 快，因为它带有能在一个周期内完成移位的桶形移位器；但它比支持 `rv32im` 的 `femtorv-electron` 慢，后者乘法只需 1 个周期，除法需要 32 个周期。

流水线能带来多大提升？理想情况下，流水线处理器可以达到 1 CPI，不过这还没有计入为解决某些情况（依赖和分支）而必须产生的停顿，后面会详细讨论。比较 `femtorv-gracilis`（`rv32imc`）与采用流水线的 `vexriscv imac` 可以发现，`vexriscv` 的速度超过前者两倍。

因此，我们的目标是把当前处理器改成流水线版本：与 `vexriscv` 类似，但更加简单。目前 ALU 只支持 `rv32i` 指令集，并能在一个周期内完成运算；此外，程序存储器与数据存储器彼此独立，全部位于 BRAM 中，存储器读写均在一个周期内完成。稍后再介绍如何实现缓存。

_关于 RAYSTONES 的说明：_所有 RAYSTONES 统计都应在 [FIRMWARE/raystones.c](FIRMWARE/raystones.c) 中取消 `#define NO_GRAPHIC` 那一行的注释后获得。这是因为 [FIRMWARE/putchar.S](FIRMWARE/putchar.S) 中的 `putchar()` 函数会先发送字符，再等待 UART 空闲。等待循环的迭代次数会随 CPU 频率与 UART 波特率之比产生*巨大*变化。

## 第 3 步：顺序执行的五级流水线

流水线处理器与使用状态机的多周期处理器类似，但每个状态都有自己的电路，并与其他状态并发运行，再把输出传给下一个状态（这里称为“级”，而不是“状态”）。由于需要处理一些棘手情况，而且我暂时还不完全清楚具体做法，因此打算一步一步来：先写出一颗拥有全部流水级的内核，但仍用状态机驱动，每次只执行一级，而不让各级并发运行；随后再研究各级并发运行时需要修改什么。

遵循所有优秀的处理器设计教材（Patterson & Hennessy、Harris & Harris），我们从一个极其经典的五级设计开始：

| 缩写 | 全称 | 说明 |
|------|------|------|
| I**F** | Instruction fetch（取指） | 从程序存储器读取指令 |
| I**D** | Instruction decode（译码） | 译码指令和立即数 |
| **E**X | Execute（执行） | 计算 ALU、测试条件和地址 |
| **M**EM | read or write memory（访存） | 加载和存储 |
| **W**B | Write back（回写） | 把结果写入寄存器堆 |

_（下文使用单字母级名 F、D、E、M、W，而不是更经典的 IF、ID、EX、MEM、WB。因为稍后要为流水线寄存器命名，其名称前缀由写入它的级和读取它的级所对应的字母组成。）_

每一级都从一组寄存器读取输入，并把输出写入另一组寄存器。这些寄存器称为“流水线寄存器”，与 RISC-V ISA 的标准寄存器（称为“架构寄存器”）不同，只用于内部记录状态。

具体来说，每一级都有自己的程序计数器副本、当前指令副本，以及从它们派生的其他字段（实际上，最后几级可以不再保存程序计数器和指令字，但目前先假定全部保留）。

更准确地说，每一级都需要拥有正常工作所需一切信息的独立副本。为什么？

- 请记住，即使目前各级仍按顺序启动，它们最终也要并发运行（下一步就是如此）。这意味着每一级会处理一条*不同*的指令，以及对应的*不同* PC；
- 除了提高吞吐量，还有另一个原因：把执行过程拆成多个寄存器到寄存器的阶段，通常能缩短关键路径，从而支持更高频率。

因此在这一步中，我们先创建一颗多周期 CPU，每个状态对应上表中的一级。状态机会依次遍历所有状态：

```verilog
   localparam F_bit = 0; localparam F_state = 1 << F_bit;
   localparam D_bit = 1; localparam D_state = 1 << D_bit;
   localparam E_bit = 2; localparam E_state = 1 << E_bit;
   localparam M_bit = 3; localparam M_state = 1 << M_bit;
   localparam W_bit = 4; localparam W_state = 1 << W_bit;
   reg [4:0] 	  state;
   wire           halt;
   always @(posedge clk) begin
      if(!resetn) begin
	 state  <= F_state;
      end else if(!halt) begin
	 state <= {state[3:0],state[4]};
      end
   end
```

下一步会移除这个状态机，但目前让各级顺序执行更合适，这样暂时无需操心一些棘手问题。以后终究无法回避它们，不过至少对我来说，每次只引入一个难点，并在每一步运行 `RAYSTONES` 基准测试、确认 CPU 仍能工作，会更容易一些。

现在来回顾即将遵守的**游戏规则**：
- 假设各级名为 `A`、`B`、`C`、`D`、`E`（比 `F`、`D`、`E`、`M`、`W` 更容易记，尽管很快就会改用后者）。每一级从一组寄存器读取输入，再把输出写入另一组寄存器。某一级（例如 `B`）的输出同时也是下一级（例如 `C`）的输入。为方便起见，这些既是 `B` 输出、又是 `C` 输入的寄存器统一使用 `BC_` 前缀。因此，`B` 从带 `AB_` 前缀的寄存器读取输入，把输出写入带 `BC_` 前缀的寄存器；随后 `C` 从 `BC_` 寄存器读取输入，再把输出写入 `CD_` 寄存器，依此类推。于是 `B` 级的 Verilog 代码如下：

```verilog
   always @(posedge clk) begin
     if(state[B_bit]) begin
         BC_xxx <= AB_aaa;
	 BC_yyy <= AB_bbb;
	 ...
     end
   end
```

- 但还不止这些，每一级都可以包含一些组合逻辑。例如在 `E` 级中，ALU 从 `DE_rs1` 和 `DE_rs2` 两个寄存器读取输入，并把结果输出到 `EM_Eresult`。这类组合函数可以使用中间信号来编写；中间信号始终为 wire，并使用所在级的名称作为前缀。在 `B` 级中，这些信号的输入要么是带 `AB_` 前缀的寄存器，要么是其他带 `B_` 前缀的信号。

- 某些级还会读写存储器（程序 ROM、数据 RAM 和寄存器堆）。在 `B` 级写存储器时，地址和待写数据可以来自带 `AB_` 前缀的寄存器，也可以来自带 `B_` 前缀的 wire。读取存储器时，输出必须始终写入带 `BC_` 前缀的寄存器，而且除了读取数据，不能再对结果做任何处理。_这是因为每块存储器本身已有寄存器化读端口；遵守这条规则，综合器才能把 `BC_` 目标映射到该寄存器化读端口。_

总结这些规则：

- **规则 1：**`B` 级从带 `AB_` 前缀的寄存器获取输入，把输出写入带 `BC_` 前缀的寄存器；

- **规则 2：**`B` 级可以有中间 wire，名称使用 `B_` 前缀。它们的输入要么是带 `AB_` 前缀的寄存器，要么是带 `B_` 前缀的 wire；

- **规则 3：**如果 `B` 级读取存储器，结果必须存入带 `BC_` 前缀的寄存器，而且不能再对结果做任何处理。

换一种说法：在 `B` 级中，`<=` 左侧只能是带 `BC_` 前缀的寄存器；`<=` 或 `=` 右侧只能是带 `AB_` 前缀的寄存器、带 `B_` 前缀的 wire，或者它们的某种组合……

不过，规则有两个**例外**（不然就太简单了！）：

- 发生跳转或分支被采用时，需要修改由取指 `F` 级管理的程序计数器。因此设置一个 32 位信号 `jumpOrBranchAddress` 和一个 1 位信号 `jumpOrBranch`；每当程序计数器需要更新为 `jumpOrBranchAddress` 时，就拉高后者；

- 大多数指令会把结果回写到寄存器堆。因此设置一个保存待写值的 32 位信号 `wbData`、一个指明目标寄存器的 5 位信号 `wbRdId`，以及一个 1 位信号 `wbEnable`；每当需要把 `wbData` 写入寄存器 `wbRdId` 时，就拉高 `wbEnable`。

如果想象一条指令从左向右穿过所有级（取指 `F`、译码 `D`、执行 `E`、访存 `M`、回写 `W`），就会发现上述两组信号从右向左传播。从某种意义上说，它们在“逆时间而行”。这正是下一步所有难题的根源。不过目前各级仍按顺序逐个激活，所以暂时没有困难。

还有一点：我很确定自己不可能一次就写对，因此调试非常重要。为此，我用 VERILOG 编写了一个简单的 [RISC-V 反汇编器](riscv_disassembly.v)。仿真时可以用它显示流水线所有级中的指令，对追踪错误非常有用。

为了简单起见，也为了能使用反汇编器，我们将在所有级中都保存 PC 和当前指令（当然，每一级中的指令各不相同）。

查看取指级之前，我*强烈*建议对寄存器和信号命名保持极其严格：流水线架构中许多东西名称相同，如果命名不规范，很容易把一切都弄混！

### 取指

看看取指 `F` 级是什么样子：

```verilog
   reg  [31:0] F_PC;
   reg  [31:0] PROGROM[0:16383]; 

   initial begin
      $readmemh("PROGROM.hex",PROGROM);
   end

   always @(posedge clk) begin
      if(!resetn) begin
	 F_PC    <= 0;
      end else if(state[F_bit]) begin
	 FD_instr <= PROGROM[F_PC[15:2]];
	 FD_PC    <= F_PC;
	 F_PC     <= F_PC+4;
      end else if(jumpOrBranch) begin
	 F_PC  <= jumpOrBranchAddress;	 
      end      
   end
   
/******************************************************************************/
   reg [31:0] FD_PC;   
   reg [31:0] FD_instr;
/******************************************************************************/

```

`F` 级有些特殊，因为它是第一级。它没有输入，但拥有程序计数器 `F_PC` 和指令 ROM。它把 PC 和加载的指令输出给译码 `D` 级。请注意，它还要处理来自后续级的“规则例外”信号 `jumpOrBranch` 和 `jumpOrBranchAddress`。

### 译码

译码 `D` 级负责识别操作码、从指令中提取立即数，并从寄存器堆读取操作数。为了尽可能简单，目前不在译码级统一译码，而是在每次需要指令信息时重新译码。听起来有点疯狂，但现阶段目标是理解流水线设计如何工作，稍后再研究如何优化。另一个原因是，我们希望用反汇编器显示每一级中的指令，因此每一级都必须保留指令和 PC；我更愿意让系统中的信息始终采用一种形式，否则可能成为错误来源。于是译码级实现如下：

```verilog
   reg [31:0] RegisterBank [0:31];
   
   always @(posedge clk) begin
      if(state[D_bit]) begin
	 DE_PC    <= FD_PC;
	 DE_instr <= FD_instr;
	 DE_rs1 <= RegisterBank[rs1Id(FD_instr)];
	 DE_rs2 <= RegisterBank[rs2Id(FD_instr)];
      end
   end

   always @(posedge clk) begin
      if(wbEnable) begin
	 RegisterBank[wbRdId] <= wbData;
      end
   end
   
/******************************************************************************/
   reg [31:0] DE_PC;
   reg [31:0] DE_instr;
   reg [31:0] DE_rs1;
   reg [31:0] DE_rs2;
/******************************************************************************/
```

它通过 `DE_PC` 和 `DE_instr` 把 PC 与指令传给下一级（执行级），并从寄存器堆读取两个操作数，存入 `DE_rs1` 和 `DE_rs2`。为了提高源码可读性，我们编写了一些辅助函数，用于从指令中提取不同字段和立即数：

```verilog
   function isALUreg; input [31:0] I; isALUreg=(I[6:0]==7'b0110011); endfunction
   function isALUimm; input [31:0] I; isALUimm=(I[6:0]==7'b0010011); endfunction
   function isBranch; input [31:0] I; isBranch=(I[6:0]==7'b1100011); endfunction
   ...
   function [4:0] rs1Id; input [31:0] I; rs1Id = I[19:15];      endfunction
   function [4:0] rs2Id; input [31:0] I; rs2Id = I[24:20];      endfunction
   function [4:0] rdId;  input [31:0] I; rdId  = I[11:7];       endfunction
   ...
   function [2:0] funct3; input [31:0] I; funct3 = I[14:12]; endfunction
   function [6:0] funct7; input [31:0] I; funct7 = I[31:25]; endfunction
   ...
   function [31:0] Uimm; input [31:0] I; Uimm={I[31:12],{12{1'b0}}}; endfunction
   ...
```

（后续各级也会使用这些函数。）_请注意，这种做法很蠢，因为它会创建多个“指令译码器”。后面会介绍如何优化；目前暂时保留，因为它更简单，也能让反汇编器显示所有级中的指令。_

还要注意“规则例外”信号 `wbEnable`、`wbRdId` 和 `wbData`，以及它们如何用于回写寄存器堆。

### 执行

执行 `E` 级的工作方式与第一篇完全相同。它包含一个大型组合函数 `E_aluOut`（保存当前运算的 32 位结果），以及 `E_takeBranch`（保存分支条件测试的 1 位结果），这里不再重复。由此可以派生出以下信号：

```verilog
   wire E_JumpOrBranch = (
         isJAL(DE_instr)  || 
         isJALR(DE_instr) || 
         (isBranch(DE_instr) && E_takeBranch)
   );

   wire [31:0] E_JumpOrBranchAddr =
	isBranch(DE_instr) ? DE_PC + Bimm(DE_instr) :
	isJAL(DE_instr)    ? DE_PC + Jimm(DE_instr) :
	/* JALR */           {E_aluPlus[31:1],1'b0} ;

   wire [31:0] E_result = 
	(isJAL(DE_instr) | isJALR(DE_instr)) ? DE_PC+4                :
	isLUI(DE_instr)                      ? Uimm(DE_instr)         :
	isAUIPC(DE_instr)                    ? DE_PC + Uimm(DE_instr) : 
        E_aluOut                                                      ;
```

最后，执行级计算运算结果 `Eresult` 以及 Load/Store 指令的地址 `addr`，并把它们连同 PC、指令和第二个源寄存器的值一起传给下一级：

```verilog
   always @(posedge clk) begin
      if(state[E_bit]) begin
	 EM_PC      <= DE_PC;
	 EM_instr   <= DE_instr;
	 EM_rs2     <= DE_rs2;
	 EM_Eresult <= E_result;
	 EM_addr    <= isStore(DE_instr) ? DE_rs1 + Simm(DE_instr) : 
                                           DE_rs1 + Iimm(DE_instr) ;
      end
   end
   
/******************************************************************************/
   reg [31:0] EM_PC;
   reg [31:0] EM_instr;
   reg [31:0] EM_rs2;
   reg [31:0] EM_Eresult;
   reg [31:0] EM_addr;
/******************************************************************************/
```

然后按如下方式连接取指级的两个“规则例外”信号：

```verilog
   assign jumpOrBranchAddress = E_JumpOrBranchAddr;
   assign jumpOrBranch        = E_JumpOrBranch & state[M_bit];
```

### 访存

访存级的工作方式与第一篇大致相同，包括那些略显复杂的操作数对齐和加载符号扩展逻辑。不过有个微妙之处：还记得吗？在某一级（这里是 `M`）从存储器读取内容时，除了把存储器内容写入结果寄存器（这里使用 `MW` 前缀）之外，不能做其他任何事情。因此无法在 `M` 级完成对齐和符号扩展。为此，我们把存储器字内容 `MW_Mdata` 传给下一级（回写级），由后者完成对齐和符号扩展。不过 Store 指令的对齐仍在 `M` 级进行，因为存储器操作的*输入*不受这条限制。

### 回写

除了上一段提到的对齐和符号扩展，回写 `W` 级还有几个多路复用器，用于生成剩余的“规则例外”信号，把寄存器数据回写到译码级：

```verilog
   assign wbData = 
	       isLoad(MW_instr)  ? (W_isIO ? MW_IOresult : W_Mresult) :
	       isCSRRS(MW_instr) ? MW_CSRresult :
	       MW_Eresult;

   assign wbEnable =
        !isBranch(MW_instr) && !isStore(MW_instr) && (rdId(MW_instr) != 0);

   assign wbRdId = rdId(MW_instr);
```

基本就是这样！结果实现在 [pipeline3.v](pipeline3.v) 中。运行 RAYSTONES：

```
$ cd FIRMWARE
$ make raystones.pipeline.hex
$ cd ..
$ ./run_verilator.sh pipeline3.v
```

将得到以下结果（同时列出 `pipeline2.v` 的结果作为参考）：


| 版本 | 说明 | CPI | RAYSTONES |
|------|------|-----|-----------|
|pipeline2.v | 3—4 状态多周期 | 3.034 | 2.614 |
|pipeline3.v | “顺序流水线” | 5 | 1.589 |

哦，太棒了，我们拼命工作，反而得到了一颗**性能更差**的内核？等等，这还不是真正的流水线；下一步才会开始“换挡”。不过事实上，我们已经有所收获。尝试综合两颗内核：

```
   BOARDS/run_ulx3s.sh pipeline2.v
   BOARDS/run_ulx3s.sh pipeline3.v   
```
然后查看 fmax（报告的最大时钟频率）：

| Version    | fmax   |
|------------|--------|
|pipeline2.v | 50 MHz |
|pipeline3.v | 80 MHz |

为什么会这样？实际上，我们把处理器的工作拆成了 5 个简单的级，每一级都按规则实现，除了……用于*跳转和分支*以及*寄存器回写*的那些特殊连线。这些规则确保各级彼此独立（特殊连线除外），而且只完成简单工作。因此，即使算上特殊连线，关键路径也比之前每条指令执行 3 或 4 个周期的版本更短。

**总结：**我们从第一篇中采用 3 或 4 状态架构的内核出发，按以下思路重新设计：
- 5 个状态：取指、译码、执行、访存、回写；
- 每个状态都会依次执行；
- 每个状态只依赖前一个状态，并把结果写给下一个状态；
- 目前每个状态都有自己的程序计数器和指令副本；
- 有两个例外：跳转/分支，以及寄存器回写。

这一步完成的内容并不复杂。

现在让所有级并行运行，看看如何向 1 CPI 靠近。只需修改几处内容，不过这些修改更加微妙。

## 第 4 步：真正的流水线

从上一步的内核开始，给它做一点“手术”：
- 首先移除状态机；
- 然后移除所有类似 `if(state[F_bit])`、`if(state[D_bit])`、`if(state[E_bit])` 的语句；
- 还要移除 `IO_mem_wr` 和 `M_wmask` 中对 `state[M_bit]` 的引用；
- 接下来开始思考哪里会出错，以及我们破坏了什么……

### 控制冒险

需要解决两类问题。第一类与跳转和分支有关。来看一个简单程序：

```
$ cd FIRMWARE
$ make helloC.pipeline.hex
```
这还会生成包含汇编代码的 `helloC.pipeline.elf.list`，开头如下：

```asm
00000000 <start>:
   0:	004001b7          	lui	x3,0x400
   4:	00020137          	lui	x2,0x20
   8:	008000ef          	jal	x1,10 <main>
   c:	00100073          	ebreak

00000010 <main>:
  10:	ff010113          	addi	x2,x2,-16 # 1fff0 <val.1514+0xffb8>
  14:	00112623          	sw	x1,12(x2)
  18:	004007b7          	lui	x15,0x400
  1c:	0ff00713          	li	x14,255
  20:	00010537          	lui	x10,0x10
  ...
```

看看流水线会做什么：

| clk  | F              | D            | E            | M            | W            |
|------|----------------|--------------|--------------|--------------|--------------|
|  1   | lui x3,0x400   | nop          | nop          | nop          | nop          |
|  2   | lui x2,0x20    | lui x3,0x400 | nop          | nop          | nop          |
|  3   | jal x1,0x10    | lui x2,0x20  | lui x3,0x400 | nop          | nop          | 
|  4   | ebreak         | jal x1,0x10  | lui x2,0x20  | lui x3,0x400 | nop          |
|  5   | addi x2,x2,-16 | ebreak       | jal x1,0x10  | lui x2,0x20  | lui x3,0x400 |

- 在时钟 3，`jal` 指令进入流水线的 `F` 级；
- 在时钟 4，它进入译码级；
- 在时钟 5，它进入执行级，这意味着我们希望在时钟 6 跳到目标地址 `0x10`。但此时已经有两条指令进入流水线。如果用 `PC` 表示 `jal` 指令的地址，那么 `PC+4` 处的指令已在 `D` 级，`PC+8` 处的指令已在 `F` 级，而它们本不该出现在那里。虽然本例中 `addi x2,x2,-16` 恰好就是跳转目标 `0x10` 处的指令，但这只是特殊情况；如果 `<main>` 更远，情况就不同了……因此，在时钟 5 执行 `jal x1,0x10` 时，它会通过 `jumpOrBranch` 和 `jumpOrBranchAddress` 把 `0x10` 发送给 `F` 级。如果不做特殊处理，就会得到以下状态：
  
| clk  | F              | D              | E            | M            | W            |
|------|----------------|----------------|--------------|--------------|--------------|
|  6   | addi x2,x2,-16 | addi x2,x2,-16 | ebreak       | jal x1,0x10  | lui x2,0x20  | 

- `E` 级中的 `ebreak` 不该存在，因为它位于 PC+4，而我们已经跳转；
- `D` 级中的 `addi x2,x2,-16` 不该存在，因为它位于 PC+8，而我们已经跳转；
- `F` 级中第二条 `addi x2,x2,-16` 是正确的，因为它位于跳转目标地址。

由于晚了两个周期才知道下一个 PC 的值而形成的这种情况，称为*控制冒险*（control hazard）。

解决思路很简单：把不该出现的指令替换为 NOP（即 `add x0, x0, x0`）：

| clk  | F              | D              | E            | M            | W            |
|------|----------------|----------------|--------------|--------------|--------------|
|  6   | addi x2,x2,-16 | nop            | nop          | jal x1,0x10  | lui x2,0x20  | 

这称为“冲刷”（flush）`D` 和 `E` 级。

就这么简单！如何实现？声明 `D_flush` 和 `E_flush` 两根 wire。当 `D` 级（或 `E` 级）的指令需要在下一周期被替换为 `nop` 时，就拉高对应信号。也就是说，当 `E` 级中的指令是跳转或被采用的分支时拉高。取指级和译码级代码更新如下：

```verilog
   localparam NOP = 32'b0000000_00000_00000_000_00000_0110011;
   wire D_flush;
   wire E_flush;

   ...
   /* F */
   always @(posedge clk) begin
      FD_instr <= PROGROM[F_PC[15:2]]; 
      FD_PC <= F_PC;
      F_PC <= F_PC+4;

      if(jumpOrBranch) begin
	 F_PC     <= jumpOrBranchAddress;
      end

      if(D_flush | !resetn) begin
         FD_instr <= NOP;
      end
      
      if(!resetn) begin
	 F_PC <= 0;
      end
   end
   ...
   /* D */
   always @(posedge clk) begin
      DE_PC    <= FD_PC;
      DE_instr <= E_flush ? NOP : FD_instr;
      DE_rs1 <= RegisterBank[rs1Id(FD_instr)];
      DE_rs2 <= RegisterBank[rs2Id(FD_instr)];
      if(wbEnable) begin
	 RegisterBank[wbRdId] <= wbData;
      end
   end
```

把 `D_flush` 和 `E_flush` 连接到 `E_JumpOrBranch`：

```verilog
   assign D_flush = E_JumpOrBranch;
   assign E_flush = E_JumpOrBranch;
```

……但上面的 VERILOG 代码有个问题，你能发现吗？

还记得规则吗？每次访问存储器时，只能把它的值写入寄存器，不能做其他事情。而这里却在某些条件下把寄存器值替换成 `NOP`。仿真时这没有影响，但综合设计时会造成灾难！综合器会尝试把 64 kB PROGROM 替换为*组合逻辑*存储器，也就是一组带有*巨大*地址译码器的触发器。它或许能装进 ULX3S，但布线始终无法收敛；我让它跑了一整夜也不行……

因此换一种做法：增加寄存器 `FD_nop`，每当需要清除 `D` 级指令时就把它置为 1：

```verilog
   /* F */
   always @(posedge clk) begin
      FD_instr <= PROGROM[F_PC[15:2]]; 
      FD_PC    <= F_PC;
      F_PC     <= F_PC+4;
      if(jumpOrBranch) begin
	 F_PC     <= jumpOrBranchAddress;
      end

      FD_nop <= D_flush | !resetn;
      
      if(!resetn) begin
	 F_PC <= 0;
      end
   end
   ...
   reg FD_nop;
```

译码级则更新如下：
```verilog
   /* D */
   always @(posedge clk) begin
      ...
      DE_instr <= (E_flush | FD_nop) ? NOP : FD_instr;
      ...
   end
```

（每次涉及 `FD_instr` 时，都别忘了测试 `FD_nop`。）

好了，跳转和分支处理完毕。由于跳转和被采用的分支会向流水线插入两个 NOP，它们需要 3 个周期：跳转或分支指令本身 1 个周期，再加上两个 NOP 的 2 个周期。

不过还没有完全结束。所有级并行执行后，还会遇到另一类问题，你能找到吗？

还记得前面说过，问题来自规则的例外。第一类问题与从 `E` 级传向 `F` 级的 `jumpOrBranch` 和 `jumpOrBranchAddress` 连线有关。

此外还有从 `W` 级传向 `D` 级寄存器堆的 `wbEnable`、`wbData` 和 `wbRdId`。这些连线会引发另一类问题。

### 数据冒险

继续分析同一个程序：
```asm
00000000 <start>:
   0:	004001b7          	lui	x3,0x400
   4:	00020137          	lui	x2,0x20
   8:	008000ef          	jal	x1,10 <main>
   c:	00100073          	ebreak

00000010 <main>:
  10:	ff010113          	addi	x2,x2,-16 # 1fff0 <val.1514+0xffb8>
  14:	00112623          	sw	x1,12(x2)
  18:	004007b7          	lui	x15,0x400
  1c:	0ff00713          	li	x14,255
  20:	00010537          	lui	x10,0x10
  ...
```
现在它会这样运行：

| clk  | F              | D              | E              | M              | W              |
|------|----------------|----------------|----------------|----------------|----------------|
|  1   | lui x3,0x400   | nop            | nop            | nop            | nop            |
|  2   | lui x2,0x20    | lui x3,0x400   | nop            | nop            | nop            |
|  3   | jal x1,0x10    | lui x2,0x20    | lui x3,0x400   | nop            | nop            | 
|  4   | ebreak         | jal x1,0x10    | lui x2,0x20    | lui x3,0x400   | nop            |
|  5   | addi x2,x2,-16 | ebreak         | jal x1,0x10    | lui x2,0x20    | lui x3,0x400   |
|  6   | addi x2,x2,-16 | nop            | nop            | jal x1,0x10    | lui x2,0x20    |  
|  7   | sw   x1,12(x2) | addi x2,x2,-16 | nop            | nop            | jal x1,0x10    |
|  8   | lui  x15,0x400 | sw   x1,12(x2) | addi x2,x2,-16 | nop            | nop            |
|  9   | li   x14,255   | lui  x15,0x400 | sw   x1,12(x2) | addi x2,x2,-16 | nop            |
|  10  | lui  x10,0x10  | li   x14,255   | lui  x15,0x400 | sw   x1,12(x2) | addi x2,x2,-16 | 


看出问题了吗？时钟 8 译码的 `sw x1,12(x2)` 指令使用寄存器 `x2`，而前一条指令 `addi x2,x2,-16` 会设置 `x2`。但 `x2` 要等到时钟 10 末尾、该指令离开 `W` 级时才会更新。因此在时钟 8，`sw x1,12(x2)` 仍处于 `D` 级，读取到的是错误的 `x2` 值。这称为*数据冒险*（data hazard）。

如何解决？思路很简单：让 `sw x1,12(x2)` 留在 `D` 级，直到 `addi x2,x2,-16` 把结果写入 `x2`（即离开 `W` 级），同时向 `E` 级插入 NOP：

| clk  | F              | D              | E              | M              | W              |
|------|----------------|----------------|----------------|----------------|----------------|
|  8   | lui  x15,0x400 | sw   x1,12(x2) | addi x2,x2,-16 | nop            | nop            |
|  9   | lui  x15,0x400 | sw   x1,12(x2) | nop            | addi x2,x2,-16 | nop            |
|  10  | lui  x15,0x400 | sw   x1,12(x2) | nop            | nop            | addi x2,x2,-16 | 
|  11  | lui  x15,0x400 | sw   x1,12(x2) | nop            | nop            | nop            |
|  12  | li   x14,255   | lui  x15,0x400 | sw   x1,12(x2) | nop            | nop            | 
|  13  | lui  x10,0x10  | li   x14,255   | lui  x15,0x400 | sw   x1,12(x2) | nop            | 

这意味着，只要 `D` 级中的指令使用了 `E`、`M` 或 `W` 级指令将要写入的寄存器，就需要：
- 把这条指令留在 `D` 级，即让 `D` 级“停顿”（stall）；
- 显然，`F` 级也应停顿，就像发生了一场小堵车；
- 冲刷 `E` 级。

换一种说法：每当某一级停顿，它之前的所有级也必须停顿，同时冲刷下一级。这里 `D` 级停顿，因此之前的级（只有 `F`）也停顿，并冲刷下一级 `E`。

如何实现？

只需声明 `F_stall` 和 `D_stall` 两根新 wire，并在 `F` 和 `D` 中加以考虑：

```verilog

   ...
   /* F */
   always @(posedge clk) begin
      if(!F_stall) begin
	 FD_instr <= PROGROM[F_PC[15:2]]; 
	 FD_PC    <= F_PC;
	 F_PC     <= F_PC+4;
      end

      if(jumpOrBranch) begin
	 F_PC     <= jumpOrBranchAddress;
      end

      FD_nop <= D_flush | !resetn;
      
      if(!resetn) begin
	 F_PC <= 0;
      end
   end
   ...
   /* D */
   always @(posedge clk) begin
      if(!D_stall) begin
	 DE_PC    <= FD_PC;
	 DE_instr <= (E_flush | FD_nop) ? NOP : FD_instr;
      end
      
      if(E_flush) begin
	 DE_instr <= NOP;
      end
      
      DE_rs1 <= RegisterBank[rs1Id(FD_instr)];
      DE_rs2 <= RegisterBank[rs2Id(FD_instr)];
      
      if(wbEnable) begin
	 RegisterBank[wbRdId] <= wbData;
      end
   end
```

换句话说，如果某一级停顿，就不更新输出；除非它同时被冲刷（冲刷的优先级高于停顿）。

冲刷和停顿信号按如下方式生成：

```verilog
   wire rs1Hazard = !FD_nop && readsRs1(FD_instr) && rs1Id(FD_instr) != 0 && (
               (writesRd(DE_instr) && rs1Id(FD_instr) == rdId(DE_instr)) ||
               (writesRd(EM_instr) && rs1Id(FD_instr) == rdId(EM_instr)) ||
	       (writesRd(MW_instr) && rs1Id(FD_instr) == rdId(MW_instr)) ) ;

   wire rs2Hazard = !FD_nop && readsRs2(FD_instr) && rs2Id(FD_instr) != 0 && (
               (writesRd(DE_instr) && rs2Id(FD_instr) == rdId(DE_instr)) ||
               (writesRd(EM_instr) && rs2Id(FD_instr) == rdId(EM_instr)) ||
	       (writesRd(MW_instr) && rs2Id(FD_instr) == rdId(MW_instr)) ) ;
   
   wire dataHazard = rs1Hazard || rs2Hazard;
   
   assign F_stall = dataHazard;
   assign D_stall = dataHazard;
   
   assign D_flush = E_JumpOrBranch;
   assign E_flush = E_JumpOrBranch | dataHazard;
```
（别忘了测试 `FD_nop`。之所以需要它，是因为 `FD_instr` 是存储器的输出寄存器；冲刷 `F` 时，不能直接把 `NOP` 写入 `FD_instr`。）

`rs1Hazard` 和 `rs2Hazard` 信号使用了新的 `writesRd()`、`readsRs1()`、`readsRs2()` 函数，定义如下：

```verilog
   function writesRd;
      input [31:0] I;
      writesRd = !isStore(I) && !isBranch(I);
   endfunction

   function readsRs1;
      input [31:0] I;
      readsRs1 = !(isJAL(I) || isAUIPC(I) || isLUI(I));
   endfunction

   function readsRs2;
      input [31:0] I;
      readsRs2 = isALUreg(I) || isBranch(I) || isStore(I);
   endfunction
```

查看下表可以发现：除 `Store` 和 `Branch` 外，所有指令都写 `rd`；除 `JAL`、`AUIPC` 和 `LUI` 外，所有指令都读 `rs1`；只有 `ALUreg`、`Branch` 和 `Store` 读取 `rs2`：

| 指令 | 操作 |
|------|------|
| ALUreg      | `rd <- rs1 OP rs2`             |
| ALUimm      | `rd <- rs1 OP Iimm`            |
| Branch      | `if(rs1 OP rs2) PC<-PC+Bimm`   |
| JALR        | `rd <- PC+4; PC<-rs1+Iimm`     |
| JAL         | `rd <- PC+4; PC<-PC+Jimm`      |
| AUIPC       | `rd <- PC + Uimm`              | 
| LUI         | `rd <- Uimm`                   |
| Load        | `rd <- mem[rs1+Iimm]`          |
| Store       | `mem[rs1+Simm] <- rs2`         |
| SYSTEM | 特殊操作 |

现在已经有了让流水线停顿的机制，也可以用它停止执行，例如在执行 `EBREAK` 时：

```verilog
   wire halt = resetn & isEBREAK(DE_instr);

   assign F_stall = dataHazard | halt;
   assign D_stall = dataHazard | halt;
```

**总结：**我们已经修复两类问题：一是控制冒险，由 `JumpOrBranch`、`JumpOrBranchAddress` 太晚到达 `F` 级程序计数器引起；二是数据冒险，由 `wbEnable`、`wbData` 和 `wbRdId` 太晚到达 `D` 级寄存器堆引起。最终处理器实现在 [pipeline4.v](pipeline4.v) 中。与上一步的“顺序流水线” [pipeline3.v](pipeline3.v) 相比，修改其实很少：
- 移除状态机；
- 增加“流水线控制”信号 `F_stall`、`D_stall`、`D_flush`、`E_flush`；
- 增加一小段组合电路，生成流水线控制信号：
   - 发生控制冒险时，冲刷 `F` 和 `D`；
   - 发生数据冒险时，让 `F`、`D` 停顿，并冲刷 `E`。

动手试试：在 [pipeline4.v](pipeline4.v) 中取消 `//define VERBOSE` 所在行的注释，然后运行：
```
   $ cd FIRMWARE
   $ make helloC.pipeline.hex
   $ cd ..
   $ ./run_verilator.sh pipeline4.v 
```
并检查 `log.txt`：
- 它会给出每一级的程序计数器和指令；
- `F` 还会标出 `jumpOrBranch` 何时有效，以及 `PC` 何时从 `jumpOrBranchAddress` 收到新值；
- `D` 会用 `[  ]` 标出 `rs1` 和 `rs2` 中的数据冒险；
- `E` 会显示 `rs1` 和 `rs2` 的值；
- `W` 会显示回写寄存器堆的值。

这个工具给我的调试带来了*巨大*帮助（我可没有第一次就把所有东西写对！！！）。

现在该用 RAYSTONES 测试新处理器了。

```
$ cd FIRMWARE
$ make raystones.pipeline.hex
$ cd ..
$ ./run_verilator.sh pipeline4.v &> log.txt
```

现在表现如何？

| 版本 | 说明 | CPI | RAYSTONES |
|------|------|-----|-----------|
|pipeline2.v | 3—4 状态多周期 | 3.034 | 2.614 |
|pipeline3.v | “顺序流水线” | 5 | 1.589 |
|pipeline4.v | 停顿/冲刷 | 2.193 | 3.734 |

好多了！速度超过上一步两倍，也比最初的 3—4 状态内核略快。不过流水线中生成了很多 NOP，它们也称为“气泡”（bubble）。能不能少吹一些气泡？

## 第 5 步：在同一周期读写寄存器堆

优秀的处理器设计教材（Patterson and Hennessy、Harris and Harris）都说，`W` 写入寄存器堆的数据可以在*同一周期*被 `D` 读取。那我们为什么要一直停顿到指令离开 `W`？

事实上，当前设计假设寄存器堆访问延迟为 1 个周期；寄存器堆映射到 BRAM 时确实如此。这意味着 `W` 写入寄存器堆的数据要到 1 个周期后才可用，因此必须检查 `W` 中的指令是否会导致数据冒险。

寄存器堆也可以实现为触发器阵列，从而没有这 1 个周期的延迟，这称为*组合式*访问。稍后会介绍具体做法；目前先保留延迟为 1 个周期的寄存器堆，看看如何为它增加连线：

```verilog
/* D */
always @(posedge clk) begin
  ...
  if(wbEnable && rdId(MW_instr) == rs1Id(FD_instr)) begin
      DE_rs1 <= wbData;
  end else begin
      DE_rs1 <= RegisterBank[rs1Id(FD_instr)];
  end

  if(wbEnable && rdId(MW_instr) == rs2Id(FD_instr)) begin
      DE_rs2 <= wbData;
  end else begin
      DE_rs2 <= RegisterBank[rs2Id(FD_instr)];
  end
  ...
end  
```

思路很简单：如果 `W` 写入的寄存器正是 `D` 要访问的寄存器，就直接把数据发送给 `D`，无需访问寄存器堆。这称为把数据“转发”（forward）到 `D`，新增连线也称为“旁路”（bypass）。

现在可以更新数据冒险规则，移除对 `W` 级指令的测试：
```
   wire rs1Hazard = !FD_nop && readsRs1(FD_instr) && rs1Id(FD_instr) != 0 && (
               (writesRd(DE_instr) && rs1Id(FD_instr) == rdId(DE_instr)) ||
	       (writesRd(EM_instr) && rs1Id(FD_instr) == rdId(EM_instr)) );
   

   wire rs2Hazard = !FD_nop && readsRs2(FD_instr) && rs2Id(FD_instr) != 0 && (
               (writesRd(DE_instr) && rs2Id(FD_instr) == rdId(DE_instr)) ||
	       (writesRd(EM_instr) && rs2Id(FD_instr) == rdId(EM_instr)) );
```

源码位于 [pipeline5.v](pipeline5.v)。

下面看看如何以“正确方式”实现：让寄存器堆采用组合式访问，从而真正做到在同一周期读写，就像优秀教材（Patterson & Hennessy、Harris & Harris）描述的那样：

```verilog
   reg [31:0] RegisterBank [0:31];
   always @(posedge clk) begin
      if(!D_stall) begin
	 DE_PC    <= FD_PC;
	 DE_instr <= (E_flush | FD_nop) ? NOP : FD_instr;
      end
      
      if(E_flush) begin
	 DE_instr <= NOP;
      end

      if(wbEnable) begin
	 RegisterBank[wbRdId] <= wbData;
      end
   end
   
/******************************************************************************/
   reg [31:0] DE_PC;
   reg [31:0] DE_instr;
   wire [31:0] DE_rs1 = RegisterBank[rs1Id(DE_instr)];
   wire [31:0] DE_rs2 = RegisterBank[rs2Id(DE_instr)];
/******************************************************************************/
```

可以看到，`DE_rs1` 和 `DE_rs2` 现在是直接连接寄存器堆的 wire；当然，读取它们的指令和旁路也都已移除。完整源码位于 [pipeline5_bis.v](pipeline5_bis.v)。

现在表现如何？

| 版本 | 说明 | CPI | RAYSTONES |
|------|------|-----|-----------|
|pipeline2.v | 3—4 状态多周期 | 3.034 | 2.614 |
|pipeline3.v | “顺序流水线” | 5 | 1.589 |
|pipeline4.v | 停顿/冲刷 | 2.193 | 3.734 |
|pipeline5.v | 停顿/冲刷，组合式寄存器堆 | 1.889 | 4.330 |

提升不算特别惊人，但可以看到，我们正在（缓慢地）向 1 CPI 靠近。

## 第 6 步：寄存器转发

上一步中，我们“模拟”了寄存器堆的组合式访问，让它能在同一周期完成读写，使 `D` 级指令“看到” `W` 级指令回写的值。我们也看到，这不是最佳办法，因为使用组合式访问的寄存器堆更加简单。不过，在设计这套“模拟”组合访问时，我们学到了一件事：如果所需数据已经存在于流水线中的其他位置，与其停顿并制造气泡、等待它写入寄存器堆，不如直接把数据发送到需要的地方，这样根本不必等待！

要转发给 `E` 的数据可能位于两个位置：
- 位于 `M`，即 `EM_Eresult`；
- 位于 `W`，即 `wbData`。

因此可以增加旁路，在 `E` 级开头把结果“转发”给 `rs1` 和 `rs2`：

```verilog
   wire E_M_fwd_rs1 = rdId(EM_instr) != 0 && writesRd(EM_instr) && 
	              (rdId(EM_instr) == rs1Id(DE_instr));
   
   wire E_W_fwd_rs1 = rdId(MW_instr) != 0 && writesRd(MW_instr) && 
	              (rdId(MW_instr) == rs1Id(DE_instr));

   wire E_M_fwd_rs2 = rdId(EM_instr) != 0 && writesRd(EM_instr) && 
	              (rdId(EM_instr) == rs2Id(DE_instr));
   
   wire E_W_fwd_rs2 = rdId(MW_instr) != 0 && writesRd(MW_instr) && 
	              (rdId(MW_instr) == rs2Id(DE_instr));
   
   wire [31:0] E_rs1 = E_M_fwd_rs1 ? EM_Eresult :
	               E_W_fwd_rs1 ? wbData     :
	               DE_rs1;
	       
   wire [31:0] E_rs2 = E_M_fwd_rs2 ? EM_Eresult :
	               E_W_fwd_rs2 ? wbData     :
	               DE_rs2;
```

然后在 `E` 级的其他位置，把 `EM_rs1`（以及 `EM_rs2`）分别替换为 `E_rs1`（以及 `E_rs2`）。一个都不能漏，否则一切都会坏掉！（每个各有三处。）

现在只剩一种数据冒险：加载指令后紧跟一条使用其结果的指令。如果隔了一条指令才使用加载结果，该结果就能从 `W` 正确转发。

流水线控制逻辑变得更简单：
```verilog
   wire rs1Hazard = readsRs1(FD_instr) && (rs1Id(FD_instr) == rdId(DE_instr)) ;
   wire rs2Hazard = readsRs2(FD_instr) && (rs2Id(FD_instr) == rdId(DE_instr)) ;
   wire dataHazard = !FD_nop && (isLoad(DE_instr)||isCSRRS(DE_instr)) && (rs1Hazard || rs2Hazard);
   
   assign F_stall = dataHazard | halt;
   assign D_stall = dataHazard | halt;
   assign D_flush = E_JumpOrBranch;
   assign E_flush = E_JumpOrBranch | dataHazard;
```
（`F_stall`、`D_stall`、`D_flush` 和 `E_flush` 的定义没有变化。）

说明：
- 正常情况下，应在 `dataHazard` 中测试 `rdId(DE_instr)` 是否不为零。不过，加载和读取 CSR 通常不会把目标设为零！即使真的加载到零，也只会产生一个原本可以避免的气泡；
- 也可以在每次出现 Load 时无条件制造一个气泡：`wire dataHazard = !FD_nop && (isLoad(DE_instr)||isCSRRS(DE_instr))`。这种写法同样正确，但会生成更多气泡。它也可能有价值，因为会简化流水线控制逻辑，并可能提高 fmax。还可以采用其他策略，例如当关键路径位于寄存器 ID 比较时，只比较其中一部分位。让 `dataHazard` 比必要情况更频繁地置位并不会导致错误，只会增加 CPI。

新版本位于 [pipeline6.v](pipeline6.v)。该跑基准测试了！

| 版本 | 说明 | CPI | RAYSTONES |
|------|------|-----|-----------|
|pipeline2.v | 3—4 状态多周期 | 3.034 | 2.614 |
|pipeline3.v | “顺序流水线” | 5 | 1.589 |
|pipeline4.v | 停顿/冲刷 | 2.193 | 3.734 |
|pipeline5.v | 停顿/冲刷，组合式寄存器堆 | 1.889 | 4.330 |
|pipeline6.v | 停顿/冲刷 + 寄存器转发 | 1.426 | 5.714 |

很好，新流水线 CPU 的速度已经超过最初 3—4 状态多周期 CPU 的两倍！

## 第 7 步：初尝分支预测

目前，跳转和分支总会引入两个气泡。看看 `JAL` 的情况：为 `JAL` 引入两个气泡有点愚蠢，因为我们知道下一个 `PC` 应当是 `PC+Jimm`。因此可以在 `D` 级计算 `PC+Jimm`，并直接传给 `F` 级。这样 `F` 级变为：

```verilog
   reg  [31:0] PC;
   wire [31:0] F_PC = D_JumpOrBranchNow ? D_JumpOrBranchAddr : PC;
   
   always @(posedge clk) begin
      
      if(!F_stall) begin
	 FD_instr <= PROGROM[F_PC[15:2]]; 
	 FD_PC    <= F_PC;
	 PC       <= F_PC+4;
      end
      
      if(E_JumpOrBranch) begin 
	 PC     <= E_JumpOrBranchAddr;
      end

      FD_nop <= D_flush | !resetn;
      
      if(!resetn) begin
	 PC <= 0;
      end
   end
```

这样，`F` 级既能从 `D_JumpOrBranchAddress` 得到当前 PC，又能从 `E_JumpOrBranchAddress` 得到下一个 PC。既然已经为 `JAL` 引入这套机制，还可以更进一步：后向分支（目标小于 `PC`）通常更容易被采用，前向分支（目标大于 `PC`）则较少被采用。只需测试 `Jimm` 的符号位，就能判断分支是向前还是向后。因此可以规定：`D` 对后向分支总是发送分支目标，即“预测采用分支”；对前向分支则从不发送，即“预测不采用分支”：

```verilog

   wire D_predictBranch = FD_instr[31];

   wire D_JumpOrBranchNow = !FD_nop && (
           isJAL(FD_instr) || 
           (isBranch(FD_instr) && D_predictBranch))
        );
   
   wire [31:0] D_JumpOrBranchAddr =  
               FD_PC + (isJAL(FD_instr) ? Jimm(FD_instr) : Bimm(FD_instr)); 
```

这种分支预测策略称为“静态”预测，因为它不保存状态；具体名称是 BTFNT（Backwards Taken, Forwards Not Taken，后向采用、前向不采用）。稍后会介绍更复杂的分支预测策略。

通过新的 `DE_predictBranch` 寄存器把预测传给 `E`：
```verilog
/* D */
   always @(posedge clk) begin
      ...
      if(!D_stall) begin
         ...
	 DE_predictBranch <= D_predictBranch;
	 ...
      end
      ...
   end      
```

如果分支未被采用，或者指令是 `JALR`，`E` 就需要发送“修正”：

```verilog
   wire E_JumpOrBranch = (
         isJALR(DE_instr) || 
         (isBranch(DE_instr) && (E_takeBranch^DE_predictBranch))
   );

   wire [31:0] E_JumpOrBranchAddr =
	isBranch(DE_instr) ? 
                     (DE_PC + (DE_predictBranch ? 4 : Bimm(DE_instr))) :
	/* JALR */   {E_aluPlus[31:1],1'b0} ;
```

当预测与实际决策不一致（`E_takeBranch^DE_predictBranch`）时，就需要修正。如果预测分支会被采用，但实际不应采用，则发送 `PC+4`；如果预测不采用，而实际应采用，则发送 `PC+Bimm`。

一旦通过 `E_JumpOrBranchAddr` 并拉高 `E_JumpOrBranch` 发送“修正”，流水线控制就会生成两个气泡。流水线控制逻辑无需改变，仍如下所示：

```verilog
   assign F_stall = dataHazard | halt;
   assign D_stall = dataHazard | halt;
   
   assign D_flush = E_JumpOrBranch;
   assign E_flush = E_JumpOrBranch | dataHazard;
```

- 注 1：`D_flush` 和 `E_flush` 仍然连接到未寄存的 `E_JumpOrBranch` 信号，因为是否冲刷 `E` 的决策必须在同一周期作出；
- 注 2：`E_JumpOrBranch` 拉高的频率比以前低得多。`JAL` 现在只需一个周期，而且预测正确时，约一半分支也只需一个周期；
- 注 3：`JALR` 仍会生成两个气泡，因为 JALR 依赖 `rs1`；该值要到 `E` 级开头才可用，而且可能来自寄存器转发；
- 注 4：当前设计的关键路径很长。`PC` 前的多路复用器由来自 ALU 的 `E_JumpOrBranch` 驱动，而 ALU 的两个输入又连接到寄存器转发逻辑。可以把 `E_JumpOrBranch` 和 `E_JumpOrBranchAddress` 存入 `EM` 流水线寄存器：

```verilog
   /* E */
   always @(posedge clk) begin
      ...
      EM_JumpOrBranchNow  <= E_JumpOrBranch;
      EM_JumpOrBranchAddr <= E_JumpOrBranchAddr;
   end		  
```

这样它们会晚一个周期才可用，但只要把它们多路复用到 `F_PC` 而不是 `PC`，就没有问题：

```verilog

   /* F */
   
   wire [31:0] F_PC = 
	       D_JumpOrBranchNow  ? D_JumpOrBranchAddr  :
	       EM_JumpOrBranchNow ? EM_JumpOrBranchAddr :
	                            PC;
   
   always @(posedge clk) begin
      if(!F_stall) begin
	 FD_instr <= PROGROM[F_PC[15:2]]; 
	 FD_PC    <= F_PC;
	 PC       <= F_PC+4;
      end
      
      FD_nop <= D_flush | !resetn;
      
      if(!resetn) begin
	 PC <= 0;
      end
   end
```

新版本位于 [pipeline7.v](pipeline7.v)。效果如何？

| 版本 | 说明 | CPI | RAYSTONES |
|------|------|-----|-----------|
|pipeline2.v | 3—4 状态多周期 | 3.034 | 2.614 |
|pipeline3.v | “顺序流水线” | 5 | 1.589 |
|pipeline4.v | 停顿/冲刷 | 2.193 | 3.734 |
|pipeline5.v | 停顿/冲刷，组合式寄存器堆 | 1.889 | 4.330 |
|pipeline6.v | 停顿/冲刷 + 寄存器转发 | 1.426 | 5.714 |
|pipeline7.v | 基础分支预测 | 1.226 | 6.077 |

有趣的是，这颗内核比实现 RV32IM 的 femtorv-electron（3.373 raystones）快得多！

## 第 8 步：动态分支预测

下面看看如何用动态分支预测进一步改进（感谢 Bruce Hoult 建议使用 gshare，稍后详述）。

现在可以从更一般的角度看待前面完成的工作：
- 在 `D` 级，预测分支是否会被采用，并根据预测结果把下一条预测指令的地址交给 `F` 级；
- `D` 级还把预测传给 `E` 级。随后 `E` 级将预测与分支实际结果比较。若二者不一致，就把修正后的地址发送给 `F` 级，并冲刷流水线中的两条指令。

这套机制完全通用，并由 `D_predictBranch` 信号参数化。目前该信号连接到 `BImm` 的符号位，即 `instr[31]`，只根据当前指令字作出决策。我们可以更聪明一些：为什么不设计一套能从先前决策中“学习”的机制？这意味着要保存一份动态更新的状态，因此称为“动态”分支预测。

要了解动态分支预测，推荐以下资料：
- [资料 1](https://danluu.com/branch-prediction/)
- [资料 2](https://people.engr.ncsu.edu/efg/521/f02/common/lectures/notes/lec16.pdf)
- [Onur Mutlu 在苏黎世联邦理工学院的课程](https://www.youtube.com/watch?v=hl4eiN8ZMJg)（感谢 Luke Wren 推荐）。
- [ALF](https://team.inria.fr/alf/members/andre-seznec/branch-prediction-research/)

看看它在内核中的实际含义。思路是保存每条分支最近一次的结果，即采用或不采用。显然不可能为所有地址都保存结果，但可以使用 PC 的低位对它们进行哈希。我们声明一张 `BHT`（Branch History Table，分支历史表），本例包含 4096 个条目：

```verilog
   localparam BP_ADDR_BITS=12;
   localparam BHT_SIZE=1<<BHT_INDEX_BITS;
   reg BHT[BHT_SIZE-1:0];
```

再声明一个根据 PC 寻址 `BHT` 的函数会很方便。当然，要忽略最低两位，因为指令按 32 位边界对齐，它们始终为 `00`。

```verilog
   function [BP_ADDR_BITS-1:0] BHT_index;
      input [31:0] PC;
      BHT_index = PC[BP_ADDR_BITS+1:2];
   endfunction
```

现在可以定义 `D_predictBranch` 信号，并通过流水线寄存器把它传给 `E`：

```verilog
   /* D */
   wire D_predictBranch = BHT[BHT_index(FD_PC)][1];
   ...
   always @(posedge clk) begin
      ...
      if(!D_stall) begin
         ...
	 DE_predictBranch <= D_predictBranch;
	 DE_BHTindex <= BHT_index(FD_PC);
      end
      ...
   end
```

随后 `E` 级要完成两件事：
- 如果预测与实际决策不一致，发送修正并生成两个气泡；
- 使用实际决策更新分支历史表。

```verilog
  /* E */
  wire E_JumpOrBranch = (
         isJALR(DE_instr) || 
         (isBranch(DE_instr) && (E_takeBranch^DE_predictBranch))
  );
  ...  
  always @(posedge clk) begin
     ...
     if(isBranch(DE_instr)) begin
	BHT[DE_BHTindex] <= E_takeBranch;
     end
     ...
  end     
```

我还加入了一些 VERILOG 指令，用于测量预测命中率，以便跟踪进展。显然，不同程序的结果可能*相差很大*。因此，我们既在 RAYSTONES 测试中测量，也在更流行的 DHRYSTONES 测试中测量（在 `FIRMWARE` 中运行 `make dhrystone.pipeline.hex`）：

| 预测策略 | raystones | dhrystones |
|----------|-----------|------------|
| 始终预测采用 | 68.6% | 89.2% |
| 后向采用、前向不采用 | 63% | 93.4% |
| 1 位 BHT | 74% | 92.7% |

很有意思：更聪明的策略并不一定总有收益！“BTFNT”策略在 DHRYSTONE 中有所提升，但在 RAYSTONE 中却不如最简单的“预测采用”策略。对于 1 位分支历史表，情况正好相反。

如果阅读了上面的参考资料，就会看到它们建议保存 2 位状态，也就是使用饱和计数器。此时 `D_predictBranch` 信号取最高位：

```verilog
   reg [1:0] BHT[BHT_SIZE-1:0];
   ...
   wire D_predictBranch = BHT[BHT_index(FD_PC)][1]; 
```

然后 `E` 级使用一个递增或递减 2 位饱和计数器的函数，按如下方式更新 `BHT`：

```verilog

   /* E */
   function [1:0] incdec_sat;
      input [1:0] prev;
      input dir;
      incdec_sat = 
 	   {dir, prev} == 3'b000 ? 2'b00 :
           {dir, prev} == 3'b001 ? 2'b00 :
	   {dir, prev} == 3'b010 ? 2'b01 :
	   {dir, prev} == 3'b011 ? 2'b10 :		
	   {dir, prev} == 3'b100 ? 2'b01 :
	   {dir, prev} == 3'b101 ? 2'b10 :
	   {dir, prev} == 3'b110 ? 2'b11 :
	                           2'b11 ;
   endfunction;

   ...
   always @(posedge clk) begin
      if(isBranch(DE_instr)) begin
	 BHT[DE_BHTindex] <= incdec_sat(BHT[DE_BHTindex], E_takeBranch);
      end
   end   
```

效果如何？不错！两个测试程序的预测率都提高了：

| 预测策略 | raystones | dhrystones |
|----------|-----------|------------|
| 始终预测采用 | 68.6% | 89.2% |
| 后向采用、前向不采用 | 63% | 93.4% |
| 1 位 BHT | 74% | 92.7% |
| 2 位 BHT | 76.8% | 95.97% |

还可以注入更多分支历史信息来提高预测率。可以创建一个 FIFO，记住最近少量分支（本例为 9 个）的结果：

```verilog
   localparam BP_HISTO_BITS=9;
   reg [BP_HISTO_BITS-1:0] branch_history;

   /* E */
   always @(posedge clk) begin
      ...
      if(isBranch(DE_instr)) begin
	 branch_history <= {E_takeBranch,branch_history[BP_HISTO_BITS-1:1]};
	 BHT[DE_BHTindex] <= incdec_sat(BHT[DE_BHTindex], E_takeBranch);
      end
   end
```

然后把这段历史与 PC 低位执行 XOR，用于索引 BHT：

```verilog
   function [BHT_INDEX_BITS-1:0] BHT_index;
      input [31:0] PC;
   /* verilator lint_off WIDTH */
      BHT_index = PC[BP_ADDR_BITS+1:2] ^ 
                  (branch_history << (BP_ADDR_BITS - BP_HISTO_BITS));
   /* verilator lint_on WIDTH */      
   endfunction
```

新版本位于 [pipeline8.v](pipeline8.v)。

这种策略称为“gshare”，几乎没有成本，却能显著提高分支预测准确率，如下表所示：

| 预测策略 | raystones | dhrystones |
|----------|-----------|------------|
| 始终预测采用 | 68.6% | 89.2% |
| 后向采用、前向不采用 | 63% | 93.4% |
| 1 位 BHT | 74% | 92.7% |
| 2 位 BHT | 76.8% | 95.97% |
| gshare | 82% | 96.3% |

它对 RAYSTONES 的提升比对 DHRYSTONES 更明显。原因可能是 DHRYSTONES 中有许多带重复模式的长循环，BHT 中的 2 位计数器大多数时候已经足够；而 RAYSTONES 会执行浮点计算（本例由软件实现），分支模式更加不规则，全局历史能捕获这些模式。

## 第 9 步：返回地址栈

优化用于实现函数调用的 `JALR` 指令，是一种相当容易的性能提升方式。思路是在处理器中加入一个小栈，称为返回地址栈（Return Address Stack，简称 RAS），典型深度为 4。每次检测到函数调用，就把函数返回地址压入栈顶。函数使用 `JALR` 返回时，返回地址已经位于栈顶，取出后弹栈即可。RAS 只需声明为 4 个 32 位寄存器：

```verilog
/* D */
   reg [31:0] RAS_0;
   reg [31:0] RAS_1;
   reg [31:0] RAS_2;
   reg [31:0] RAS_3;
```

把 `D_JumpOrBranchNow` 和 `D_JumpOrBranchAddress` 两个信号重命名为 `D_predictPC` 和 `D_PCprediction`（我觉得这样更清楚）。

```verilog
/* D */

   wire D_predictPC = !FD_nop && (
      D_isJAL || D_isJALR || (D_isBranch && D_predictBranch) 
   );
   wire [31:0] D_PCprediction = 
                D_isJALR ? RAS_0 : 
	        (FD_PC + (D_isJAL ? D_Jimm : D_Bimm));
```

把预测返回地址传给 `E`，让 `E` 能够将其与实际返回地址比较：

```verilog
/* D */
always @(posedge clk) begin
   ...
   DE_predictRA <= RAS_0;
end
```

现在，执行级要完成两件不同的事。首先判断 PC 预测是否正确；如果不正确，就拉高 `E_correctPC`，并把 `E_PCcorrection` 发送给 `F`（它们以前分别称为 `E_jumpOrBranch` 和 `E_jumpOrBranchAddress`）：

```verilog
/* E */
  wire [31:0] E_JALRaddr = {E_aluPlus[31:1],1'b0};
  wire E_correctPC = (
     (DE_isJALR    && (DE_predictRA != E_JALRaddr)   ) || 
     (DE_isBranch  && (E_takeBranch^DE_predictBranch))
  );
  wire [31:0] E_PCcorrection = DE_isBranch ? DE_PCplus4orBimm : E_JALRaddr;
```

还需要更新 RAS：

```verilog
/* E */
 always @(posedge clk) begin
    ...
    if(!FD_nop && !D_flush) begin
       if(D_isJAL && (D_rdId==1 || D_rdId==5)) begin
          RAS_3 <= RAS_2;
          RAS_2 <= RAS_1;
          RAS_1 <= RAS_0;
          RAS_0 <= FD_PC + 4;
        end 
        if(D_isJALR && D_rdId==0 && (D_rs1Id == 1 || D_rs1Id==5)) begin
           RAS_0 <= RAS_1;
           RAS_1 <= RAS_2;
           RAS_2 <= RAS_3;
        end
    end
    ...
  end    
```

我们假设 `JAL x1,addr` 或 `JAL x5,addr` 实现函数调用，而 `JALR x0,x1,imm` 或 `JALR x0,x5,imm` 实现函数返回。为什么还有 `x5`？实际上，`x1` 和 `x5` 都可以用作链接寄存器；RISC-V 规范第 17 页表 2.1 对此有所说明（感谢 Bruce Hoult）。

新版本实现在 [pipeline9.v](pipeline9.v) 中。可以通过配置启用或禁用不同功能：

| 标志 | 说明 |
|------|------|
| `CONFIG_PC_PREDICT` | 启用 `D` -> `F` 路径 |
| `CONFIG_RAS` | 返回地址栈（需要 `CONFIG_PC_PREDICT`） |
| `CONFIG_GSHARE` | gshare 分支预测（需要 `CONFIG_PC_PREDICT`） |
| `CONFIG_INITIALIZE` | 若设置，则把寄存器初始化为 0 |
| | 某些综合工具需要此项 |

现在可以测试返回地址栈的影响：

| 预测策略 | raystones | CPI | dhrystones（DMIPS/MHz） | CPI |
|----------|-----------|-----|-------------------------|-----|
| gshare | 7.185 | 1.121 | 1.562 | 1.116 |
| gshare+RAS | 7.374 | 1.092 | 1.606 | 1.086 |

## 用 VERILOG 编写的调试器

![](verilog_riscv_debugger.png)

此外，该版本内置了一个用 VERILOG 编写的“调试器”，仅适用于 Verilator。设置 `CONFIG_DEBUG` 即可启用。它允许逐时钟运行内核并检查不同流水级，还会显示寄存器转发和流水线控制信号。

调试器只有两条命令：
- 按 `<return>` 前进到下一个时钟周期；
- 按 `g` 前进到下一个断点。

可以在 VERILOG 源码中通过拉高 `breakpoint` 信号来声明断点，参见 [pipeline9.v](pipeline9.v) 第 840 行。

例如，可以在执行给定地址处的指令时中断：

```verilog
  wire breakpoint = (DE_PC   == 32'h000000); // break on address reached
```

或者在读写给定地址的数据时中断：

```verilog
   wire breakpoint = (EM_addr == 32'h......);
```

在 LED 输出时中断：
```verilog
   wire breakpoint = (EM_addr == 32'h400004);
```

在字符输出时中断：
```verilog
   wire breakpoint = (EM_addr == 32'h400008);
```

下一步要实现 RV32M。它使用多周期 ALU，需要稍微复杂一些的流水线控制，因此有这个调试器会很方便！

## 第 10 步：RV32M

借助流水线、寄存器转发、分支预测和返回地址栈，我们达到了 7.374 raystones。现在看看能否通过支持更多指令，也就是整数乘法、除法和求余，让处理器更快。这些指令属于 `RV32M` 扩展。内核支持它们后，只需面向该架构重新编译代码；编译器会生成 `MUL` 指令，而不再调用 gcc 库中的 `__mulsi3`。需要支持 8 条新指令：

| 指令 | 操作 | 说明 | funct3 |
|------|------|------|--------|
| MUL rd,rs1,rs2 | rd <- rs1 * rs2 | 有符号，取最低 32 位 | 000 |
| MULH rd,rs1,rs2 | rd <- rs1 * rs2 | 有符号，取最高 32 位 | 001 |
| MULHSU rd,rs1,rs2 | rd <- rs1 * rs2 | 有符号×无符号，取最高 32 位 | 010 |
| MULHU rd,rs1,rs2 | rd <- rs1 * rs2 | 无符号，取最高 32 位 | 011 |
| DIV rd,rs1,rs2 | rd <- rs1 / rs2 | 有符号版本 | 100 |
| DIVU rd,rs1,rs2 | rd <- rs1 / rs2 | 无符号 | 101 |
| REM rd,rs1,rs2 | rd <- rs1 % rs2 | 有符号 | 110 |
| REMU rd,rs1,rs2 | rd <- rs1 % rs2 | 无符号 | 111 |

这里包含乘法、除法和求余，各有有符号和无符号版本。对于乘法，32 位乘以 32 位会得到 64 位结果，因此需要两条指令：`MUL` 取最低位，`MULH` 取最高位。`MULH` 又有三种变体：有符号×有符号、有符号×无符号、无符号×无符号。

首先要译码这些新指令。它们的 `opcode` 全部为 `7'b0110011`，也就是 `ALUreg`。区别在于，所有 RV32M 指令的 `funct7` 都是 `0000001`，具体指令再由 `funct3` 编码。正好有 8 条新指令和 8 种 `funct3` 取值！请注意，`funct3[2]` 指示该指令是乘法，还是除法/求余。

既然已经知道如何译码 RV32M 指令，接下来就要创建计算乘法、除法和余数的电路。

对于乘法，大多数 FPGA 都内置了能在一个周期内完成计算的模块。它们称为 DSP（Digital Signal Processing，数字信号处理）模块，因为这正是其擅长的任务。只要向 Yosys 传递正确的标志，在 VERILOG 中写 `A <= B * C;` 即可使用。幸好有这些模块，否则就要逐位计算乘积，每次乘法需要 32 个周期；在分支预测良好时，与软件乘法 `__mulsi` 相比并没有多少优势。

我们将计算 33 位×33 位的乘积。为什么是 33 位？Matthias Koch 想到可以根据运算类型和操作数符号添加一位符号位：

```verilog
   /* E */
   wire E_isMULH   = DE_funct3_is[1];
   wire E_isMULHSU = DE_funct3_is[2];
   
   wire E_mul_sign1 = E_rs1[31] &  E_isMULH;
   wire E_mul_sign2 = E_rs2[31] & (E_isMULH | E_isMULHSU);

   wire signed [32:0] E_mul_signed1 = {E_mul_sign1, E_rs1};
   wire signed [32:0] E_mul_signed2 = {E_mul_sign2, E_rs2};
   wire signed [63:0] E_multiply = E_mul_signed1 * E_mul_signed2;
```

对于除法和求余则别无选择，必须创建多周期电路。这里采用经典算法，一次除法需要 33 个周期。实现深受 Claire Wolf 的 picorv 启发，也包含 Matthias Koch 的一些想法：

```verilog

   /* E */
   reg [31:0] EE_dividend;
   reg [62:0] EE_divisor;
   reg [31:0] EE_quotient;
   reg [31:0] EE_quotient_msk;

   reg  EE_div_sign;
   reg 	EE_divBusy     = 1'b0;
   reg 	EE_divFinished = 1'b0;

   wire E_divstep_do = (EE_divisor <= {31'b0, EE_dividend});
   
   always @(posedge clk) begin
      if (!EE_divBusy) begin
	 if(DE_isDIV & !dataHazard & !EE_divFinished) begin
	    EE_quotient_msk <= 1 << 31;
	    EE_divBusy     <= 1'b1;	    
	 end
	 EE_dividend <=   ~DE_funct3[0] & E_rs1[31] ? -E_rs1 : E_rs1;
	 EE_divisor  <= {(~DE_funct3[0] & E_rs2[31] ? -E_rs2 : E_rs2), 31'b0};
	 EE_quotient <= 0;
	 EE_div_sign <= ~DE_funct3[0] & (DE_funct3[1] ? E_rs1[31] : 
                         (E_rs1[31] != E_rs2[31]) & |E_rs2)       ;
      end else begin
	 EE_dividend <= E_divstep_do ? EE_dividend-EE_divisor[31:0]:EE_dividend;
	 EE_divisor  <= EE_divisor >> 1;
	 EE_quotient <= E_divstep_do ? EE_quotient|EE_quotient_msk :EE_quotient;
	 EE_quotient_msk <= EE_quotient_msk >> 1;
	 EE_divBusy <= EE_divBusy & !EE_quotient_msk[0];
      end 
      EE_divFinished <= EE_quotient_msk[0];
   end 
```

请注意，执行除法时，执行级会在寄存器中保存一些状态。为了与 wire 区分，这些寄存器使用 `EE` 前缀。

不同指令按如下方式多路复用，形成 ALU 输出。此外，`aluBusy` 信号指示当前是否正在执行除法。

```verilator
   wire [2:0] E_divsel = {DE_isDIV,DE_funct3[1],EE_div_sign};
   
   wire [31:0] E_aluOut_muldiv =
     (  DE_funct3_is[0]    ? E_multiply[31: 0] : 32'b0) | // 0:MUL
     ( |DE_funct3_is[3:1]  ? E_multiply[63:32] : 32'b0) | // 1:MH, 2:MHSU, 3:MHU
     (  E_divsel == 3'b100 ?  EE_quotient      : 32'b0) | // DIV
     (  E_divsel == 3'b101 ? -EE_quotient      : 32'b0) | // DIV (negative)
     (  E_divsel == 3'b110 ?  EE_dividend      : 32'b0) | // REM
     (  E_divsel == 3'b111 ? -EE_dividend      : 32'b0) ; // REM (negative)
   
   wire [31:0] E_aluOut = DE_isRV32M ? E_aluOut_muldiv : E_aluOut_base;

   wire aluBusy = EE_divBusy | (DE_isDIV & !EE_divFinished);
```

每当除法正在运行，指令都必须留在 `E` 级，即让 `E` 停顿。这意味着 `F` 和 `D` 也要停顿，并冲刷 `M`。创建新的流水线控制信号 `E_stall` 和 `M_flush`，并按如下方式连接所有流水线控制信号：

```verilator
   assign F_stall = aluBusy | dataHazard | halt;
   assign D_stall = aluBusy | dataHazard | halt;
   assign E_stall = aluBusy;
   
   assign D_flush = E_correctPC;
   assign E_flush = E_correctPC | dataHazard;
   assign M_flush = aluBusy;
```

新版本位于 [pipeline10.v](pipeline10.v)。首先需要重新配置系统，把目标从 RV32I 改为 RV32IM：

```
  $ cd learn-fpga/FemtoRV
  $ make ICEBREAKER.firmware_config
  $ cd TUTORIALS/FROM_BLINKER_TO_RISCV/FIRMWARE
  $ make clean raystones.pipeline.hex
```

然后运行：
```
  $ cd ..
  $ ./run_verilator.sh pipeline10.v
```

新内核得到 18.215 RAYSTONES。虽然 FPU 更适合光线追踪，但一个周期完成的硬件整数乘法仍带来了显著提升。

要观察处理器运行，可以取消 `CONFIG_DEBUG` 所在行的注释，并用 Verilator 运行。按 `g` 到达下一个断点（设置在 `DIV` 指令处），再按 `<return>` 逐周期运行。显示的除法掩码会体现计算进度。

## 第 10 步：针对 fmax 进行优化

我们已经看到 CPI 得到了改善，那么 fmax（以及 LUT 和 FF）表现如何？以下是 ULX3S 上的数值：

| 版本 | 说明 | Fmax | LUT | FF |
|------|------|------|-----|----|
|pipeline2.v | 3—4 状态多周期 | 50 MHz | 2100 | 532 |
|pipeline3.v | “顺序流水线” | 80 MHz | 1877 | 1092 |
|pipeline4.v | 停顿/冲刷 | 50 MHz | 2262 | 1148 |
|pipeline5.v | 停顿/冲刷，寄存器堆旁路 | 55 MHz | 2115 | 1040 |
|pipeline5_bis.v | 停顿/冲刷，组合式寄存器堆 | 55 MHz | 1903 | 1040 |
|pipeline6.v | 停顿/冲刷 + 寄存器转发 | 40 MHz | 2478 | 1040 |

- “顺序流水线”可在 80 MHz 通过时序，体现了简单流水级的好处；
- 加入流水线控制逻辑后（pipeline4），Fmax 迅速下降；
- pipeline5 和 pipeline5_bis 使用组合式寄存器堆（模拟或真实），写入值能在同一周期读取，因此流水线控制逻辑更简单，Fmax 又有所提升；
- 最后，加入寄存器转发逻辑后，Fmax 再次下降。

不过此前完全没有努力优化 Fmax，主要目标一直是降低 CPI。现在看看能做些什么。

有几项可行的改进：

### 在 `E` 级读取 `DATARAM`，在 `M` 级进行符号扩展和对齐

把存储器操作拆到多个流水级，可以缩短关键路径。

### `wbEn` 寄存器流水线

从最后一行 fmax 的下降可以推测，寄存器转发也是限制因素。仔细看看之前的寄存器转发实现：

```verilog
   wire E_M_fwd_rs1 = rdId(EM_instr) != 0 && writesRd(EM_instr) && 
	              (rdId(EM_instr) == rs1Id(DE_instr));
   
   wire E_W_fwd_rs1 = rdId(MW_instr) != 0 && writesRd(MW_instr) && 
	              (rdId(MW_instr) == rs1Id(DE_instr));

   wire E_M_fwd_rs2 = rdId(EM_instr) != 0 && writesRd(EM_instr) && 
	              (rdId(EM_instr) == rs2Id(DE_instr));
   
   wire E_W_fwd_rs2 = rdId(MW_instr) != 0 && writesRd(MW_instr) && 
	              (rdId(MW_instr) == rs2Id(DE_instr));
   
   wire [31:0] E_rs1 = E_M_fwd_rs1 ? EM_Eresult :
	               E_W_fwd_rs1 ? wbData     :
	               DE_rs1;
	       
   wire [31:0] E_rs2 = E_M_fwd_rs2 ? EM_Eresult :
	               E_W_fwd_rs2 ? wbData     :
	               DE_rs2;
```

其中 `writesRd(I)` 定义为 `!isStore(I) & !isBranch(I)`，而 `isStore()` 和 `isBranch()` 各要测试 7 位。再加上对 5 位 `rdId` 的测试，控制 `E_rs1` 和 `E_rs2` 两个多路复用器的四个表达式相当宽。怎样缩小它们？

思路是在每一级创建一个 `wbEnable` 流水线寄存器，指示指令是否回写寄存器堆。译码级负责测试 `!isStore(I) & !isBranch(I)`，执行级再测试 `rdId` 是否不为 0。

### 在 `D` 级译码，并从后续级移除 `instr` 和 `PC`

目前 `instr` 会逐级传播，并在每次需要时重新译码，这并不理想。更好的做法是在 `D` 级识别不同指令，再传播用于识别各条指令的 `is_xxxx` 标志。

### 谨慎优化 `D` 级

10 类主要指令的位模式编码并非随意选择，而是为效率精心设计。最明显的是最低两位始终为 `00`，无需测试；除此之外还有更微妙的结构。

例如，`JAL` 是唯一一条第 3 位为 1 的指令，因此 `D` 无需测试其他位。这一点很有意思，因为它简化了 PC 预测逻辑。

### 为寄存器转发构建流水线化的寄存器 ID 比较器

`E` 级开头的两个三选一多路复用器，由源寄存器 ID（`rs1Id` 和 `rs2Id`）与目标寄存器 ID `rdId` 在 (`DE`,`EM`) 和 (`DE`,`MW`) 之间的比较结果驱动。这些比较可以提前一个周期在 `D` 中完成，并把结果存入 4 个触发器：`DE_rs1Id_eq_EM_rdId`、`DE_rs1Id_eq_MW_rdId`、`DE_rs2Id_eq_EM_rdId` 和 `DE_rs2Id_eq_MW_rdId`。

名为 `TordBoyau` 的“最终成品”位于[这个项目](https://github.com/BrunoLevy/TordBoyau)。在 ARTY 上使用 Vivado 时，它可在 100—125 MHz 通过时序，并可成功超频到 140 MHz。
- 使用 RV32I 配置时，光线追踪测试达到 7.375 raystones（CPI 为 1.092，gshare + RAS 的效果非常好！）；
- 使用 RV32IM 配置时，达到 18.215 raystones。

## 结语

希望你喜欢这一系列教程。还有许多主题值得研究；等我理解它们以后，总有一天会准备以下教程：

- **优化：**还有几种办法能让处理器更快。首先是 `STORE`->`LOAD` 寄存器转发，让 `memcpy()` 达到每个字 1 个周期。其次，`TordBoyau` 的 `RV32IM` 版本只能在约 80 MHz 通过时序，但可安全超频到 140 MHz，因此很可能存在需要消除的伪路径。

- **缓存：**目前流水线处理器拥有 `PROGROM` 和 `DATARAM`。应在它们上连接一个缓存接口，再连接 SDRAM 控制器，按需从 SDRAM 取出和存入数据。我计划在 LiteX 系统中开发这部分；LiteX 的功能非常完整，包括 SDRAM 控制器、帧缓冲区等。这样就能在流水线处理器上运行 DOOM。第一篇中构建的内核已经可以运行 DOOM，因为 LiteX 开发者已把它接入自己的缓存控制器。

- **中断：**RISC-V ISA 包含多个章节，其中有一套“特权级 ISA”，提供用于控制中断和陷阱的特殊寄存器与指令。但我认为官方 ISA 文档很难读，因为它会列举每一种可能。我打算写一份简短说明，介绍运行 Linux-noMMU 等系统所需的最低配置。

- **MMU：**既然提到 MMU，它本身就是一个有趣主题。@ultraembedded 告诉我，加入 MMU 非常简单。

- **乱序执行：**本篇末尾，借助 gshare 分支预测和返回地址栈的共同作用，我们达到了 1.092 CPI，非常接近“光速”1 CPI。事实上，还可以设计“超光速”处理器：使用多个执行单元，挑选指令并以乱序方式（OoO）执行，从而做到**每个周期执行多条指令**。已有一些内核采用这种方式，例如 @dolu1990 的 NaxRiscV 和 @ultraembedded 的 BiRiscV。它们的微架构完全不同：仍由一组流水线构成，但组织成树形，并由一个负责路由指令的“中央机构”控制（Tomasulo 算法）。如果能有一套通用设计，让人自由创建自己的流水线树，并由代码自动生成“中央机构”，那就太棒了。LiteX 这样的框架或许可以用于实现。

## 参考资料

- [Intel、AMD 和 VIA 微架构](https://www.agner.org/optimize/microarchitecture.pdf)
- [宾夕法尼亚大学课程](https://acg.cis.upenn.edu/milom/cse372-Spring06/labs/lab4.html)
- [课程幻灯片](https://acg.cis.upenn.edu/milom/cis501-Fall05/lectures/06_pipeline.pdf)

## 乱序执行参考资料（由 Charles Papon 推荐）
- [Henry 的博士论文](https://www.stuffedcow.net/files/henry-thesis-phd.pdf)
- [Boom](https://docs.boom-core.org/en/latest/index.html)
- [Mashimo](https://www.rsg.ci.i.u-tokyo.ac.jp/members/shioya/pdfs/Mashimo-FPT'19.pdf)

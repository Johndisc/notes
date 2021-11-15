## Decoder Simulation

![image-20211010111155734](D:\notes\assets\zsim结构\image-20211010111155734.png)

### Micro-Ops

zsim会将一条指令分解为多条微指令，例如store指令包括地址计算和储存两条微指令。微指令储存在结构体     `struct DynUop(decoder.h)`中。结构体成员及说明：

| `DynUop` Field Name | Description                              |
| :-----------------: | ---------------------------------------- |
|         rs          | 源寄存器，可以是结构寄存器或临时寄存器   |
|         rd          | 目的寄存器，可以是结构寄存器或临时寄存器 |
|         lat         | 从调度到完成的总周期数                   |
|      decCycle       | 由解码器生成的相关周期数                 |
|        type         | 微指令类型                               |
|      portMask       | 微指令能够部署的端口                     |
|     extraSlots      | 非流水线执行的uop周期数                  |

### Architectural and Temporary Registers

结构寄存器由PIN提供，从(REG_LAST + 1) 开始为临时寄存器。结构寄存器储存指令的源操作数和目的操作数，临时寄存器储存微指令的源操作数和目的操作数。

### Pre-processed Instructions

指令预处理是在解码之前将PIN指令对象(`class INS`)中的寄存器和内存操作数提取出来，储存到zsim中的结构体`class Decoder::Instr`中，直接调用构造函数`Instr(INS)`实现。`Instr`的成员及说明如下：

| `Decoder::Instr` Field Name | Description                                                  |
| :-------------------------: | :----------------------------------------------------------- |
|             ins             | PIN instruction object.                                      |
|           loadOps           | An array of memory read operand IDs. Used with PIN `INS_OperandMemoryBaseReg()` and `INS_OperandMemoryIndexReg()`. |
|          numLoads           | Number of memory read operands (i.e. the size of the `loadOps` array). |
|          storeOps           | An array of memory write operand IDs. Used with PIN `INS_OperandMemoryBaseReg()` and `INS_OperandMemoryIndexReg()`. |
|          numStores          | Number of memory write operands (i.e. the size of the `loadOps` array). |
|           inRegs            | An array of source register IDs, both explicit and implicit (e.g. FLAGS). |
|          numInRegs          | Number of source registers (i.e. the size of the `inRegs` array). |
|           outRegs           | An array of destination register IDs, both explicit and implicit (e.g. FLAGS). |
|         numOutRegs          | Number of destination registers (i.e. the size of the `numOutRegs` array). |

### Converting Instructions to Uops

在一个基本块被PIN插桩前，`Trace()`会先调用`decodeBbl()`来解码基本块，信息储存在`BblInfo`对象中。`decodeBbl()`用PIN提供的遍历函数遍历指令列表，先调用`canFuse()`对每条指令检查是否能融合，只包括比较或测试指令接条件跳转指令的情况。如果为true，调用`decodeFusedInstrs()`进行融合。

不能融合则调用`decodeInstr()`将指令解析为一个`DynUop`结构并储存到数组`uopVec`中。再用`instrAddr`储存指令地址，`instrBytes`储存指令字节数，`instrUops`储存指令产生的微指令数，`instrDesc`储存指令本身。

### Simulating Pre-Decoder

x86指令的大小不是固定的，最长为15字节。zsim用3个重要参数来模拟预编码器：块大小，每周期最多指令数，每周期最多预编码子节数。预编码器一次性读取16字节的块，处理完当前块才会读取下一个块。一个块中最多6条指令或16字节能在同一个周期中处理。代码中，遍历每条指令，如果指令大小超过16字节或数量超过6或超过当前块，就表明在一个周期内处理掉这一块的指令，周期数自增。

`pcyc`储存当前预编码周期，从0开始。`pblk`储存相应的当前16字节大小块编号。`pcnt`储存已预编码的指令数。`psz`储存已预编码的字节数。`predecCycle`数组储存每条指令预编码的周期数。

### Simulating Decoder

编码器在预编码完成后工作，采用“4-1-1-1”原则：最多4条指令能同时用3个简单编码器和一个复杂编码器编码。三个简单编码器能编码小于8字节的指令，生成1个uop，复杂编码器能编码最多生成4个uop的指令，没有指令大小限制。

在代码中，遍历每条指令的每条指令，如果该指令只有一条微指令并且size小于8字节，则它是一条简单指令，简单指令的计数`dsimple`加一，否则复杂指令的计数`dcomplex`加一。每当总共出现两条复杂指令或一条复杂加三条简单时就处理掉这些指令，解码周期`dcyc`加一，否则当前指令的解码周期就等于预解码周期`pcyc`。另外，如果某条指令的预解码周期超过解码周期，说明解码解快了，需要等到预解码周期结束，即`dcyc = pcyc`。

指令信息储存在`BblInfo`中，包含大小，指令数，解码的微指令uop，即`DynBbl oooBbl[]`，其包含一个`DynUop`对象数组。

### The Inductive Model

zsim在一次迭代中模拟一条uop在所有流水线部件上接收和释放的周期。只有阻塞型的流水线部件会被模拟。给定流水线部件S1, …, X, Y, …, Sm，两条规则：

1. 如果此前的所有uop在所有部件上的释放周期已知，那么当前微指令在部件Y上的接收和释放周期是可计算的。

2. 条件同上时，如果：

   1. uop在X的接收周期已知
   2. uop在前面所有部件的接收和释放周期已知
   3. 之前所有微指令在所有部件上的接收周期已知

   那么任何uop在X的释放周期和在Y的接收周期都是可计算的。

例子：

X和Y是两个阻塞型部件，之间有k个非阻塞型部件。已知当前uop在X的接收周期CX，需要计算该uop在X的释放周期和在Y的接收周期。

简单情况：Y没有uop缓冲区

- 如果CX+k < CY，说明如果uop在CX释放，会在Y处理完前一条uop之前到达，因此需要将X阻塞(CY-k-CX)个周期，X的释放周期为(CY-k)，Y的接收周期为CY
- 如果CX+k > CY，同上可得X的释放周期为CX，Y的接收周期为(CX+k)

复杂情况：Y有uop缓冲区，大小为SZ。仅当FIFO有空位时才能接收uop，因此当前的前一条uop为uop(i-SZ)。因此需要比较uop接收周期和uop(i-SZ)的释放周期。

## Simulating The Rest of Frontend

### Issue Queue

事件队列在解码和发射之间，包含时间戳数组`buf`，表示释放周期，和指针`idf`，表示队列尾，实现为一个循环队列。

根据上面所述，对于解码器在周期C产生的uop，将C与当前`idx`所指的队尾`buf[idx]`比较，如果大于C，则阻塞解码器，否则等到解码周期结束立刻释放指令。最后将当前周期入队。

## Simulating The Backend Pipeline

### Backend Overview

后端包括Register Aliasing Table (RAT)，Register File (RF), 乱序执行引擎，load和store单元，Reorder Buffer (ROB)。

uop发射(issue)指将uop移到后端流水线，调度(dispatch)表示将uop移到功能单元执行。uop首先被发射到后端，被RAT重命名，再插入到ROB和指令窗口，再在所有源操作数和执行端口就绪时调度执行。

3个阶段：

1. 寄存器重命名

   RAT每周期重命名4个uop，不会阻塞，在两个周期内完成。

2. 寄存器读取

   读取重命名寄存器，有吞吐量限制。

3. ROB和指令窗口

   将uop插入到ROB和指令窗口中。

### Simulating Uop Issue

每个周期最多发射4个uop，满了就停滞一个周期。

### Reorder Buffer (ROB)

ROB是一个和事件队列类似的FIFO循环缓冲区，储存每条uop的退休周期，每条指令必须按顺序退休。两个变量：

- `curRetireCycle`记录前一条uop的退休周期。
- `curCycleRetires`记录当前周期要退休的uop数。

如果`curCycleRetires`达到了ROB的带宽W，则增加`curRetireCycle`并重置`curCycleRetires`。

在新插入一条退休指uop时，调用`markRetire`函数。如果该uop的退休周期小于等于`curRetireCycle`且没到带宽W，则增加`curCycleRetires`，否则增加`curRetireCycle`并重置`curCycleRetires`。如果退休周期大于`curRetireCycle`，则直接将`curRetireCycle`设置为该周期。

### Simulating RAT and RF and Register Scoreboard(RS)

每个寄存器的访问可在一个周期内完成。分数板`regScoreboard`中记录了每个寄存器的上一次访问结束的时间，在一轮uop迭代中，首先判断uop的两个源寄存器的结束时间是否小于当前周期，即两个寄存器是否可用，可用数加到`curCycleRFReads`中。如果`curCycleRFReads`超过了带宽上限`RF_READS_PER_CYCLE`，则停滞一个周期。

RAT+ROB+RS的最大周期，一方面等待两个源寄存器的最大完成周期`cOps`，另一方面取ROB的最大退休周期和当前周期的较大值，并加上这三个阶段，每个阶段两个周期的共6个周期。取这二者的较大值即是调度周期。

`uint64_t dispatchCycle = MAX(cOps, MAX(c2, c3) + (DISPATCH_STAGE - ISSUE_STAGE));`


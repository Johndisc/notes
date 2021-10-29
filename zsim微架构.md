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

x86指令的大小不是固定的，最长为15字节。zsim用3个重要参数来模拟预编码器：块大小，每周期最多指令数，每周期最多预编码子节数。预编码器一次性读取16字节的块，处理完当前块才会读取下一个块。一个块中最多6条指令或16字节能在同一个周期中处理。

`pcyc`储存当前预编码周期，从0开始。`pblk`储存相应的当前16字节大小块编号。`pcnt`储存已预编码的指令数。`psz`储存已预编码的字节数。`predecCycle`数组储存每条指令预编码的周期数。

### Simulating Decoder

编码器在预编码完成后工作，采用“4-1-1-1”原则：最多4条指令能同时用3个简单编码器和一个复杂编码器编码。三个简单编码器能编码小于8字节的指令，生成1个uop，复杂编码器能编码最多生成4个uop的指令，没有指令大小限制。

指令信息储存在`BblInfo`中，包含大小，指令数，解码的微指令uop，即`DynBbl oooBbl[]`，其包含一个`DynUop`对象数组。


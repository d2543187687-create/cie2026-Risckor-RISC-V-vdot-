# vadd.vv 在香山处理器中的完整执行流程

*基于 XiangShan 仿真波形的指令流与数据流追踪*

| 项目 | 内容 |
| --- | --- |
| 测试指令 | `vadd.vv v3, v1, v2` |
| PC | `0x80000170` |
| 向量配置 | VL = 4，SEW = 32 bit，LMUL = 1 |
| 输入向量 | v1 = {1,2,3,4}；v2 = {10,20,30,40} |
| 预期结果 | v3 = {11,22,33,44} |
| 用途 | 比赛第一阶段：依据仿真波形描述向量加法在香山中的执行过程 |

## 核心结论

本次波形已经完整追踪到：Decode / Rename → Dispatch → VF Issue Queue → DataPath → VFEX2 → VF Writeback Arbiter → Vector Physical Register File。最终写回数据为 `0x0000002C00000021000000160000000B`，对应逻辑结果 `{11,22,33,44}`。

整理日期：2026-09-15

## 1 测试程序与指令语义

本次仿真并不是单独执行一条孤立的加法指令，而是先通过向量加载指令把内存中的两个数组装入向量寄存器，再执行加法，最后通过向量存储指令把结果写回内存。

```text
vsetivli zero, 4, e32, m1, ta, ma
vle32.v v1, (a)
vle32.v v2, (b)
vadd.vv v3, v1, v2
vse32.v v3, (c)
```

其中 `vadd.vv v3, v1, v2` 的语义是对两个源向量逐元素执行整数加法：

```text
for i = 0 ... VL-1:
    v3[i] = v1[i] + v2[i]
```

本次配置中 VL=4、SEW=32，因此一个 128 bit 向量中正好包含 4 个 32 bit 元素。两个输入向量分别为 `{1,2,3,4}` 和 `{10,20,30,40}`，理论结果为 `{11,22,33,44}`。

需要注意：`vadd.vv` 本身并不访问内存。数据从内存进入向量寄存器由前面的 `vle32.v` 完成；`vadd.vv` 只处理已经位于向量寄存器/物理向量寄存器中的源数据；后面的 `vse32.v` 再把结果写回内存。

从软件视角看，数据流可以概括为：

| 阶段 | 作用 |
| --- | --- |
| 内存 a / b | 保存输入数组 `{1,2,3,4}` 和 `{10,20,30,40}`。 |
| `vle32.v` | 通过向量加载单元把内存数据装入 v1、v2。 |
| `vadd.vv` | 读取两个源向量并逐元素相加，产生 v3。 |
| `vse32.v` | 读取 v3，并把结果存入内存数组 c。 |

## 2 香山后端中的整体执行路径

从硬件微架构视角看，一条 `vadd.vv` 不会直接从译码器跳到加法器，而是要经过重命名、分派、调度、寄存器读取、执行和写回等多个阶段。

| 阶段 | 作用 |
| --- | --- |
| Decode | 识别该机器指令属于向量整数算术指令，并生成 fuType、源/目的寄存器类型、写使能等控制信息。 |
| Rename | 把逻辑寄存器 v1、v2、v3 转换为物理寄存器编号，并为新的 v3 版本分配 pdest。本次 `pdest=0x33`。 |
| Dispatch | 根据指令类型和功能单元需求，将微操作送入合适的调度器。本次从 `toIssueQueues_16` 进入 VF Scheduler。 |
| Issue Queue | 保存待执行微操作，跟踪源操作数是否 ready，并在条件满足后选择指令发射。 |
| DataPath | 依据 psrc 从向量物理寄存器文件取得真正的源操作数，并把微操作和数据送入执行单元。 |
| VFEX2 | 执行向量整数 ALU 运算；按 SEW=32 将 128 bit 数据划分为 4 个元素并分别相加。 |
| vfWbArbiter | 多个可能产生向量结果的执行源共享有限的 VF 写回端口，因此需要仲裁。 |
| Vector RF | 最终通过 VF write port 1 把结果写入物理向量寄存器 `0x33`。 |

**两条线必须分开理解：** 指令控制流关注“这条指令去哪里、何时 ready、被哪个调度器和执行单元接收”；数据流关注“两个源向量在哪里读出、如何进入 VFEX2、结果如何写回”。两条线在执行阶段汇合。

## 3 Decode：把机器码变成后端可理解的控制信息

当前 `vadd.vv` 的 PC 为 `0x80000170`。处理器取到该机器指令后，Decode 首先判断它是一条向量整数算术指令。Decode 的任务不是做加法，而是产生后端所需的控制信息，例如功能单元类型 fuType、源操作数类型、目的寄存器类型、向量写使能等。

因此，经过 Decode 后，后端处理的已经不再只是原始机器码，而是一条携带完整执行属性的微操作（uop / DynInst）。

## 4 Rename：从逻辑寄存器转换到物理寄存器

程序中的 v1、v2、v3 是架构可见的逻辑向量寄存器。香山是乱序处理器，为消除 WAR/WAW 等假相关并允许多条指令并行存在，后端使用物理寄存器进行实际执行。

| 项目 | 说明 |
| --- | --- |
| 逻辑源 v1/v2 | 经 Rename Table 查表后转换为物理源寄存器 psrc。 |
| 逻辑目的 v3 | 为新的 v3 版本分配新的物理目的寄存器 pdest。 |
| 本次追踪 | 后续各级一直携带 `pdest=0x33`，因此 `0x33` 成为追踪这条 `vadd.vv` 的重要标志之一。 |

**为什么 Rename 很重要？** 如果没有寄存器重命名，后端很难让多条写同一个逻辑寄存器的指令同时存在。Rename 让每个尚未提交的新版本拥有独立的物理寄存器，从而支撑乱序执行。

## 5 Dispatch：把 vadd.vv 送入 VF Scheduler

在 Dispatch 输入端，本次波形观测到：

```text
io_fromRename_0_bits_pc = 0x80000170
io_fromRename_0_valid = 1
io_fromRename_0_ready = 1
io_fromRename_0_bits_hasException = 0
```

这说明该 `vadd.vv` 已成功从 Rename 送达 Dispatch，并且没有异常。随后在 Dispatch 输出侧观察到：

```text
io_toIssueQueues_16_valid = 1
io_toIssueQueues_16_ready = 1
io_toIssueQueues_16_bits_pc = 0x80000170
io_toIssueQueues_16_bits_pdest = 0x33
io_toIssueQueues_16_bits_fuType = 0x000080000
```

其中 `valid=1` 且 `ready=1` 表示这一拍真正发生了 Decoupled 握手，因此该微操作成功进入后端调度队列，而不仅仅是“挂在接口上”。

## 6 为什么是 toIssueQueues_16

CtrlBlock 中把不同调度器的入队端口按 intUops → fpUops → vfUops → memUops 的顺序展平。根据本机配置：INT 有 8 路入队端口，FP 有 6 路，VF 有 6 路，因此全局编号 14~19 属于 VF Scheduler。

| 全局编号 | 类别 | 说明 |
| --- | --- | --- |
| 0 ~ 7 | INT | 整数调度器 |
| 8 ~ 13 | FP | 浮点调度器 |
| 14 ~ 19 | VF | 向量/向量浮点调度器 |
| 20 以后 | MEM | 访存调度器 |

因此 `toIssueQueues_16` 明确属于 VF Scheduler。进一步按 VF 内部每个 IssueBlock 的 `numEnq=2` 划分，可知 16/17 对应 VF IssueBlock 1。这里的“16”是展平后的全局入队端口编号，不是“第16个独立 Issue Queue”。

## 7 Issue Queue：等待、选择并乱序发射

`vadd.vv` 被 Dispatch 送入 VF Issue Queue 后，不一定立即执行。调度器需要判断源物理寄存器是否 ready、相应执行资源是否可用，以及是否存在更早或更合适的微操作需要先发射。

本次波形中，在约 10259 的波形时间位置看到 `toIssueQueues_16` 完成握手；之后在约 10281 的位置观察到：

```text
io_fromVfIQ_1_0_valid = 1
io_fromVfIQ_1_0_bits_common_pdest = 0x33
io_fromVfIQ_1_0_bits_common_fuType = 0x000080000
io_fromVfIQ_1_0_bits_common_robIdx_value = 0x64
```

pdest 和 fuType 与此前 Dispatch 阶段完全一致，因此可以确认，这次从 VF Issue Queue 发射出去的就是同一条 `vadd.vv`。

**入队与发射的区别：** `toIssueQueues_16.valid=1` 表示指令进入调度队列；`fromVfIQ_1_0.valid=1` 表示该指令已经满足发射条件，被调度器选中并送向执行数据通路。两者之间正是 Issue Queue 的等待与调度过程。

根据 VF Scheduler 参数，VF IssueBlock 1 包含 VFEX2 和 VFEX3，其中 VFEX2 支持 VialuCfg，因此 `vadd.vv` 从 `fromVfIQ_1_0` 方向发射，与其向量整数 ALU 属性一致。

## 8 DataPath：把源数据送到 VFEX2

Issue Queue 主要保存并调度微操作；真正的源向量数据需要通过 DataPath 从向量物理寄存器文件读出。发射的微操作携带 psrc、pdest、fuType、robIdx 等信息，DataPath 据此读取源物理寄存器，并把控制信息和操作数共同送往执行单元。

本次波形在 `fromVfIQ_1_0` 之后观察到：

```text
io_toVecExu_1_0_valid = 1
io_toVecExu_1_0_bits_pdest = 0x33
io_toVecExu_1_0_bits_fuType = 0x000080000
```

关键字段保持一致，说明该 `vadd.vv` 已经穿过 DataPath，并正式到达 VFEX2 对应的向量执行入口。波形中两组接口之间约有 1 个仿真时间单位的变化差，但这不能直接解释为“CPU 执行耗时 1 ps”；真实周期数必须结合时钟信号判断。

## 9 VFEX2：真正完成逐元素加法

此时 VL=4、SEW=32，因此 128 bit 源向量被解释为 4 个 32 bit 元素。VFEX2 内的向量整数 ALU 对活动元素分别执行加法，而不是把 128 bit 整体当作一个大整数相加。

| 元素 | src1 | src2 | result |
| --- | --- | --- | --- |
| 0 | 1 | 10 | 11 |
| 1 | 2 | 20 | 22 |
| 2 | 3 | 30 | 33 |
| 3 | 4 | 40 | 44 |

因此执行结果的逻辑元素顺序为 `{11,22,33,44}`。

## 10 vfWbArbiter：执行结果进入写回网络

VFEX2 得到结果后，并不能直接占用寄存器文件写端口。香山中有多个执行源可能产生向量寄存器结果，而向量物理寄存器文件的物理写端口数量更少，因此需要 vfWbArbiter 进行多对少的端口仲裁。

在当前生成的 RTL 中，vfWbArbiter 可以看到 9 路输入、6 路输出。9 路输入代表潜在的向量写回结果源；6 路输出代表实际可用的 VF 写回端口。

本次追踪中，执行结果进入：

```text
vfWbArbiter.io_in_4
    ↓
vfWbArbiter
    ↓
vfWbArbiter.io_out_1
```

这也与 VFEX2 的参数配置相符：VFEX2 的普通向量结果配置为 `VfWB(port = 1, priority = 0)`，因此本次结果最终使用 VF writeback port 1。

**为什么输入比输出多？** 并不是每个执行单元都必须独占一个寄存器写端口。通过仲裁器共享写回端口可以降低寄存器堆面积、布线和功耗。如果多个结果同时竞争同一端口，仲裁逻辑与 ready/valid 握手负责决定哪一路先通过。

## 11 Vector Physical Register File：结果真正落地

在最终写回处，波形同时观察到：

```text
io_out_1_bits_pdest = 0x33
vfRfWaddr_1 = 0x33
vfRfWdata_1 = 0000002C00000021000000160000000B
```

这三项构成了完整的写回证据：写回仲裁器输出的目的物理寄存器是 `0x33`，向量寄存器文件 1 号写端口的地址也为 `0x33`，写入数据则为 128 bit 的向量结果。

## 12 对最终 128 bit 写回数据进行验证

将 `vfRfWdata_1` 按 32 bit 拆分：

```text
0000002C | 00000021 | 00000016 | 0000000B
      44 |       33 |       22 |       11
```

波形通常按高位到低位显示，因此视觉上是 44、33、22、11；按照向量元素从低编号到高编号的逻辑顺序，则对应 `{11,22,33,44}`，与输入向量逐元素相加的理论结果完全一致。

**最终验证结论：** `vadd.vv` 已在香山后端中完成正确执行：目的物理寄存器始终保持 `pdest=0x33`，最终 `vfRfWaddr_1=0x33`，写回数据对应 `{11,22,33,44}`。因此 Dispatch、Issue、Execute 和 Writeback 各阶段已经通过波形形成闭环证据。

## 13 全流程信号证据链

| 阶段 | 关键接口/信号 | 本次观测 | 含义 |
| --- | --- | --- | --- |
| Dispatch 输入 | `fromRename_0` | `PC=0x80000170`；`valid=1`；`ready=1`；`hasException=0` | `vadd.vv` 正常进入 Dispatch |
| Dispatch 输出 | `toIssueQueues_16` | `valid=1`；`ready=1`；`pdest=0x33`；`fuType=0x80000` | 成功进入 VF Scheduler |
| Issue 发射 | `fromVfIQ_1_0` | `valid=1`；`pdest=0x33`；`fuType=0x80000`；`robIdx=0x64` | VF Issue Queue 选中并发射 |
| DataPath 输出 | `toVecExu_1_0` | `valid=1`；`pdest=0x33`；`fuType=0x80000` | 送入 VFEX2 |
| 执行后写回 | `vfWbArbiter in_4 → out_1` | 输出目的寄存器仍为 `0x33` | 执行结果进入 VF 写回端口 1 |
| 寄存器写入 | `vfRfWaddr_1 / vfRfWdata_1` | `addr=0x33`；`data=...2C...21...16...0B` | 物理向量寄存器写入正确结果 |

## 14 可直接用于技术文档的总结表述

当 `vadd.vv v3, v1, v2` 进入香山处理器后，Decode 阶段首先识别其为向量整数算术指令，并生成源寄存器类型、目的寄存器类型及功能单元类型等控制信息。Rename 阶段将逻辑向量寄存器映射至物理向量寄存器，并为目的寄存器分配新的物理寄存器，本次仿真中目的物理寄存器编号为 `0x33`。

指令随后经 Dispatch 送入 VF Scheduler。波形显示，PC 为 `0x80000170` 的 `vadd.vv` 通过 `toIssueQueues_16` 完成入队握手。根据后端 IssueBlock 配置，该端口属于 VF Scheduler 的第二个 IssueBlock。待源操作数就绪并满足发射条件后，该指令通过 `fromVfIQ_1_0` 被调度发射，并经 DataPath 的 `toVecExu_1_0` 送至支持 VialuCfg 的 VFEX2 执行单元。

在本测试中，VL=4、SEW=32，两个源向量分别为 `{1,2,3,4}` 和 `{10,20,30,40}`。VFEX2 的向量整数 ALU 对四个 32 bit 元素分别执行加法，得到 `{11,22,33,44}`。执行结果经 vfWbArbiter 进行写回端口仲裁，本次结果从输入通道 4 进入并由输出端口 1 写回。波形中 `vfRfWaddr_1=0x33`，与该指令的 `pdest=0x33` 一致，而 `vfRfWdata_1=0x0000002C00000021000000160000000B`，对应逻辑结果 `{11,22,33,44}`。由此可完整验证 `vadd.vv` 从译码、重命名、分派、调度发射、向量执行到物理寄存器写回的整个执行过程。

## 15 对后续 vdot.vv 设计的意义

这次 `vadd.vv` 追踪最重要的收获，是已经理解了香山后端中一条向量指令的公共框架。以后新增 `vdot.vv` 时，Rename、Dispatch、Issue Queue、DataPath 和写回网络大部分仍会复用现有机制；真正需要重点修改和验证的内容将集中在指令定义/译码、执行单元支持、SEW=8 数据拆分、符号扩展、乘加/累加器以及对应的功能验证。

**一句话记忆：** `vadd.vv` 的硬件执行可以概括为：“译码确定属性 → 重命名确定物理寄存器 → Dispatch 送入 VF 调度器 → Issue Queue 等待并发射 → DataPath 读取源向量 → VFEX2 逐元素相加 → 写回仲裁 → 结果写入物理向量寄存器”。

## 原版页面图像

### 第 1 页

![原版第 1 页](香山_vadd向量加法完整执行流程_assets/page-01.png)

### 第 2 页

![原版第 2 页](香山_vadd向量加法完整执行流程_assets/page-02.png)

### 第 3 页

![原版第 3 页](香山_vadd向量加法完整执行流程_assets/page-03.png)

### 第 4 页

![原版第 4 页](香山_vadd向量加法完整执行流程_assets/page-04.png)

### 第 5 页

![原版第 5 页](香山_vadd向量加法完整执行流程_assets/page-05.png)

### 第 6 页

![原版第 6 页](香山_vadd向量加法完整执行流程_assets/page-06.png)

### 第 7 页

![原版第 7 页](香山_vadd向量加法完整执行流程_assets/page-07.png)

### 第 8 页

![原版第 8 页](香山_vadd向量加法完整执行流程_assets/page-08.png)

### 第 9 页

![原版第 9 页](香山_vadd向量加法完整执行流程_assets/page-09.png)

### 第 10 页

![原版第 10 页](香山_vadd向量加法完整执行流程_assets/page-10.png)

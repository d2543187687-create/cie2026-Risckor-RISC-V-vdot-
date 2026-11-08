# 第二问：VDOT 指令 RTL 设计与实现技术文档

*基于 XiangShan Kunminghu V2 的自定义向量点积指令实现*

> 指令：vdot.vv
> 输入：SEW = 8，VLEN = 128
> 功能：8 组有符号 int8 乘法并归约求和
> 实现方式：复用 FuType.vimac，在 VIMacU 内扩展专用数据通路

*技术文档基础版本*

## 1. 任务要求与设计目标

第二问要求在 XiangShan Kunminghu V2 的现有向量执行框架基础上新增自定义向量点积指令 vdot.vv。本设计面向 SEW=8、VLEN=128 的固定场景，从两个源向量寄存器中分别取低 8 个有符号 8 bit 元素，逐元素相乘后完成归约求和，并将结果写回目标向量寄存器。

- 输入数据类型：signed int8。

- 参与计算的元素个数：8。

- 运算：result = Σ sign_extend(vs1[i]) × sign_extend(vs2[i])，i=0…7。

- 执行单元：复用现有 VIMacU，而不是新增 FuType。

- 结果格式：内部采用 19 bit 有符号累加；最终写入 vd[31:0]，vd[127:32] 清零。

- 验证：至少覆盖正数、负数、混合符号和边界数据。

### 1.1 数值范围分析

单个 signed int8 的取值范围为 [-128, 127]。两数相乘的最大正值为 (-128)×(-128)=16384。8 项全部取最大正乘积时，理论最大值为 8×16384=131072；最小值可由 (-128)×127×8 得到 -130048。因此 18 bit 有符号数不足以完整覆盖结果范围，19 bit 有符号累加可以安全表示。

| 项目 | 数值 | 说明 |
| --- | --- | --- |
| 单项乘积最大值 | 16384 | (-128) × (-128) |
| 8 项最大和 | 131072 | 8 × 16384 |
| 8 项最小和 | -130048 | 8 × (-128 × 127) |
| 内部累加宽度 | 19 bit | 覆盖完整理论范围 |

## 2. 总体实现方案

本设计不新增独立的功能单元类型，而是将 vdot.vv 作为 VIMAC 类操作集成进现有向量整数乘加执行路径。这样可以复用 XiangShan 已有的 Decode、Rename、Dispatch、Issue Queue、向量寄存器堆读取、Og2ForVector、VFEX0、向量写回等后端基础设施，仅在译码、内部操作码以及 VIMacU 数据通路中增加必要逻辑。

Decode → Rename → Dispatch → VF Issue Queue → DataPath → Og2ForVector<br>
→ VFEX0 / VIMacU → Vector Writeback

在当前参数配置中，VFEX0 包含 VfmaCfg、VialuCfg、VimacCfg 和 VppuCfg，因此将 vdot.vv 译码到 FuType.vimac 可以直接路由到 VFEX0 / VIMacU，同时避免新增调度端口、寄存器堆读端口以及新的写回端口。

### 2.1 关键文件修改

| 文件 | 作用 |
| --- | --- |
| xiangshan/backend/decode/Instructions.scala | 新增 VDOT_VV 指令编码匹配。 |
| xiangshan/backend/decode/VecDecoder.scala | 将 VDOT_VV 译码为 FuType.vimac + VimacType.vdot。 |
| yunsuan/encoding/Opcode/VimacOpcode.scala | 新增 vdot 内部操作码 111。 |
| yunsuan/package.scala | 新增 VimacType.vdot。 |
| xiangshan/backend/fu/wrapper/VIMacU.scala | 实现 8 路乘法、归约树和结果写回选择。 |

## 3. 指令编码与译码实现

### 3.1 自定义指令编码

本设计使用如下 BitPat 匹配 vdot.vv：

```scala
object VDot {
  def VDOT_VV = BitPat("b111001???????????000?????1010111")
}
```

其中 funct6 固定为 111001，funct3 为 000，opcode 为向量指令主操作码 1010111，vm 位保留为通配。对于测试指令 vdot.vv v3, v1, v2，vm=1 时使用的机器码为 0xe61101d7。

### 3.2 VecDecoder 译码

```scala
VDOT_VV -> OPIVV(
  FuType.vimac,
  VimacType.vdot,
  T, F, F
),
```

译码结果将该指令归入 FuType.vimac，因此无需新增 FuType。调度、向量寄存器读取以及 VFEX0 路由均复用已有 VIMAC 机制。

### 3.3 YunSuan 内部操作类型

```scala
def vdot = "b111".U(width.W)
```

现有 VIMAC 内部操作码已使用 000~110，111 可作为新增 vdot 操作码。

```scala
def vdot = LiteralCat(
  0.U(2.W),
  INT.S, INT.S, INT.X,
  FMT.VVV,
  VimacOpcode.vdot
)
```

该类型明确指定两个输入均按 signed 整数处理，采用 VVV 格式，并通过内部 opcode 区分于 vmul、vmacc 等既有 VIMAC 操作。

## 4. VIMacU 数据通路设计

### 4.1 8 路并行 signed int8 乘法

向量源操作数在 VIMacU 中分别通过 vs1 和 vs2 获得。设计从每个 128 bit 源向量的低 64 bit 中依次提取 8 个 8 bit 元素，使用 asSInt 转换为有符号数后并行相乘。每个 int8×int8 乘积使用 16 bit 有符号结果保存。

```scala
private val dotProducts = Wire(Vec(8, SInt(16.W)))
for (i <- 0 until 8) {
  val a = vs1(8*i+7, 8*i).asSInt
  val b = vs2(8*i+7, 8*i).asSInt
  dotProducts(i) := a * b
}
```

### 4.2 三级归约加法树

为了避免 8 项顺序累加形成过长组合路径，本设计采用平衡归约树。宽度按 16→17→18→19 bit 逐级扩展，使用 +& 保留进位，保证边界数据不溢出。

```scala
private val dotSum01 = dotProducts(0) +& dotProducts(1)
private val dotSum23 = dotProducts(2) +& dotProducts(3)
private val dotSum45 = dotProducts(4) +& dotProducts(5)
private val dotSum67 = dotProducts(6) +& dotProducts(7)

private val dotSum0123 = dotSum01 +& dotSum23
private val dotSum4567 = dotSum45 +& dotSum67

private val dotSum = dotSum0123 +& dotSum4567
private val dotSumPiped = SNReg(dotSum, latency)
```

```text
8× int8×int8
↓
4× 17 bit 部分和
↓
2× 18 bit 部分和
↓
1× 19 bit 点积结果
```

### 4.3 输出格式与写回选择

归约结果在内部为 19 bit signed。按照本项目采用的实现约定，结果符号扩展到 32 bit 后写入目标向量寄存器低 32 bit，其余高 96 bit 清零。该约定需在技术文档中明确说明，以避免与官方未明确的结果元素宽度解释混淆。

```scala
private val dotResult32 = Wire(SInt(32.W))
dotResult32 := dotSumPiped

private val vdotResult =
  Cat(0.U((dataWidth - 32).W), dotResult32.asUInt)

io.out.bits.res.data :=
  Mux(outIsVdot, vdotResult, mgu.io.out.vd)
```

## 5. 编译与仿真流程

完成 RTL 修改后，先进行 Chisel/Scala 编译，再生成仿真 RTL 并构建支持 FST 波形的 emu。

```bash
cd ~/github/xs-env/XiangShan
mill -i xiangshan.compile

export NOOP_HOME=/home/dongh/github/xs-env/XiangShan
make sim-verilog

rm -rf build/emu-compile build/emu
make emu EMU_TRACE=fst -j40
```

### 5.1 测试程序核心指令

```c
asm volatile(
  "li t0, 0x600\n\t"
  "csrs 0x300, t0\n\t"
  "vsetivli zero, 8, e8, m1, ta, ma\n\t"
  "vle8.v v1, (%0)\n\t"
  "vle8.v v2, (%1)\n\t"
  ".word 0xe61101d7\n\t"
  "vse8.v v3, (%2)\n\t"
  :
  : "r"(a), "r"(b), "r"(out)
  : "t0", "memory"
);
```

由于工具链不认识自定义助记符，测试程序通过 .word 0xe61101d7 直接插入指令机器码。测试前使用 mstatus.VS=Dirty 启用向量状态。

### 5.2 仿真命令

```bash
./build/emu   -i ../nexus-am/apps/vdot_test/build/vdot_test-riscv64-xs.bin   --no-diff   --dump-wave   --wave-path=vdot_multi.fst   -b 0 -e 150000 -C 150000   > vdot_multi.log 2>&1
```

## 6. 功能测试与验证结果

测试覆盖正数、负数、混合符号以及接近理论边界的输入。对于每组测试，同时检查软件输出与波形内部的乘积、部分和和最终结果。

| 测试类型 | 输入特征 | 期望结果 | 验证状态 |
| --- | --- | --- | --- |
| 正数 | A=[1..8], B=[8..1] | 120 | PASS |
| 负数 | A=[-1..-8], B=[8..1] | -120 | PASS |
| 混合符号 | 正负交错输入 | -68 | PASS |
| 边界测试 | 包含 -128、127 等边界值 | -127769 | PASS |
| 理论最大值 | 构造最大正和 | 131072 | PASS |
| 理论最小值 | 构造最大负和 | -130048 | PASS |

### 6.1 波形内部结果示例

以正数测试为例，8 路乘积和归约树中间值如下：

```text
dotProducts:
8, 14, 18, 20, 20, 18, 14, 8

第 1 层部分和:
22, 38, 38, 22

第 2 层部分和:
60, 60

最终:
120 = 0x78
```

波形中 dotProducts_0~7、dotSum01/23/45/67、dotSum0123、dotSum4567、dotSumPiped 与上述理论结果一致，说明乘法、归约树和流水对齐均工作正确。

## 7. 设计特点与技术分析

- 复用现有 VIMAC 调度与执行资源，修改范围小，降低了对后端整体结构的侵入性。

- 8 路乘法并行执行，避免标量逐项计算。

- 采用平衡归约树，组合深度由串行 7 级累加降低为 3 级归约。

- 通过逐级位宽扩展避免 signed int8 边界条件下的溢出。

- 使用现有 VIMacCfg 的流水延迟与控制通路，保持结果与 fuOpType 在时序上对齐。

- 当前设计面向固定 SEW=8、8 元素点积，尚未泛化到任意 VL、SEW 或更大点积长度。

### 7.1 当前实现假设与局限性

本实现将 19 bit 点积结果符号扩展为 32 bit 后写入 vd[31:0]，高 96 bit 清零。该格式是当前工程实现中的明确约定。若后续官方测试向量对结果元素宽度、尾部元素状态或非法条件提出更严格要求，应同步调整 RTL 与参考模型。

另外，Verilator 仿真可以证明周期级功能正确，但不能直接证明综合后的最高工作频率或 ASIC 时序收敛。若需要进一步评估面积、功耗和关键路径，应在综合工具或 FPGA/ASIC 后端流程中继续分析。

## 8. 第二问结论

本设计在 XiangShan Kunminghu V2 中完成了自定义 vdot.vv 指令的 RTL 集成。通过复用 FuType.vimac 和 VFEX0/VIMacU 执行路径，实现了 8 路 signed int8 并行乘法和三级归约加法树，内部使用 19 bit 有符号累加以覆盖完整理论结果范围，并将 32 bit 结果写入目标向量寄存器低位。编译、仿真和多组测试均验证了功能正确性，正数、负数、混合符号及边界输入均得到期望结果。因此，第二问所要求的专用点积指令设计、RTL 实现和功能验证已经完成。

## 附录 A：关键修改清单

- Instructions.scala：新增 VDOT_VV BitPat。

- VecDecoder.scala：新增 VDOT_VV → FuType.vimac / VimacType.vdot。

- VimacOpcode.scala：新增内部 opcode vdot = 111。

- yunsuan/package.scala：新增 VimacType.vdot。

- VIMacU.scala：新增 8 路乘法、三级归约树、流水寄存器和输出 Mux。

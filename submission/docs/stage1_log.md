# XiangShan 仿真环境部署与 Hello XiangShan 闭环验证日志

*第一阶段环境部署 编译与问题排查*

## 摘要

本阶段完成了 XiangShan RTL 仿真环境的部署与闭环验证。具体结果包括：完成 Chisel 设计展开并生成 build/rtl/SimTop.sv，使用 Verilator 5.020 生成并编译 RTL 仿真可执行程序 build/emu，使用 nexus-am 与 RISC-V GNU 交叉工具链生成 hello-riscv64-xs.bin，最终由 emu 加载并执行该测试负载，终端成功输出 Hello XiangShan。构建日志与运行日志均已保存，可用于后续波形采集、执行过程分析及 vdot.vv 功能开发。

本日志重点记录故障现象、定位依据、根因、处理方法与验证结果。终端内容仅保留能够支持判断的关键行，完整输出存放于日志文件中。

## 1 目标与范围

本阶段的目标是在 Ubuntu Linux 主机上建立可重复使用的 XiangShan RTL 仿真环境，并完成以下任务：

- 配置 xs-env 及 XiangShan 相关依赖；
- 由 XiangShan 的 Chisel 源码生成 SystemVerilog RTL；
- 使用 Verilator 生成并编译 C++ 仿真模型；
- 生成 RTL 仿真可执行程序 build/emu；
- 使用 nexus-am 编译 RISC-V 测试程序；
- 使用 emu 加载并执行 Hello XiangShan 测试负载；
- 保存构建与运行日志，为后续开发和验证提供可追溯记录。

完整构建链如下：

```text
XiangShan Chisel 源码
    ↓ 设计展开与 RTL 生成
SystemVerilog RTL  build/rtl/SimTop.sv
    ↓ Verilator
C++ 仿真模型  VSimTop*.cpp  VSimTop.mk
    ↓ GNU C++ 与 DiffTest 仿真框架
RTL 仿真可执行程序  build/emu
    ↓ 加载 nexus-am 生成的 RISC-V 测试负载
Hello XiangShan 输出
```

## 2 环境与关键产物

### 2.1 工程目录

| 项目 | 路径 |
| --- | --- |
| xs-env | /home/dongh/github/xs-env |
| XiangShan | /home/dongh/github/xs-env/XiangShan |
| nexus-am | /home/dongh/github/xs-env/nexus-am |

每次进入新的 Shell 会话后执行：

```text
source ~/github/xs-env/env.sh
```

脚本设置的关键环境变量如下：

```text
XS_PROJECT_ROOT=/home/dongh/github/xs-env
NOOP_HOME=/home/dongh/github/xs-env/XiangShan
NEMU_HOME=/home/dongh/github/xs-env/NEMU
AM_HOME=/home/dongh/github/xs-env/nexus-am
DRAMSIM3_HOME=/home/dongh/github/xs-env/DRAMsim3
```

其中，NOOP_HOME 指向 XiangShan 工程目录，AM_HOME 指向 nexus-am 工程目录。

### 2.2 最终使用的工具与产物

| 类别 | 最终状态 |
| --- | --- |
| Verilator | 5.020，安装路径为 /opt/verilator-5.020 |
| RISC-V 交叉编译器 | riscv64-linux-gnu-gcc 11.4.0 |
| SystemVerilog RTL | $NOOP_HOME/build/rtl/SimTop.sv |
| RTL 仿真程序 | $NOOP_HOME/build/emu |
| RISC-V 测试负载 | $AM_HOME/apps/hello/build/hello-riscv64-xs.bin |
| 构建日志 | emu_build.log |
| 运行日志 | $NOOP_HOME/hello_run.log |

## 3 EMU 构建问题排查

### 3.1 首次执行 make emu 失败

在 XiangShan 工程目录执行 make emu 后，流程进入 RTL 生成与 Verilator 处理阶段，但出现大量 UNOPTFLAT 警告，并最终退出：

```text
%Warning-UNOPTFLAT: ...
%Error: Exiting due to 48 warning(s)
make[1]: *** [...] Error 1
make: *** [...] Error 2
```

相关警告主要出现在自动生成的 build/rtl/SRAMTemplate_*.sv，以及与 ClockGate、TAGE、ITTAGE、wckEn 和 _resetState 等模块或信号有关的逻辑中。

UNOPTFLAT 表示 Verilator 无法对相关逻辑进行预期的展平优化。该警告通常首先影响仿真性能，但也可能为组合逻辑不收敛问题提供线索，因此不能仅凭警告名称断定 RTL 存在功能错误，也不应在未分析原因的情况下将其一概忽略。本次先核对实际使用的 Verilator 版本，没有直接修改自动生成的 RTL。

### 3.2 安装 4.210 后仍调用 4.038

检查实际执行程序：

```text
which verilator
verilator --version
```

输出为：

```text
/usr/bin/verilator
Verilator 4.038 2020-07-11
```

虽然 Verilator 4.210 已安装至 /opt/verilator-4.210，但 Shell 的命令搜索路径仍优先命中 /usr/bin/verilator，因此构建过程继续调用 4.038。随后调整并刷新当前 Shell 的 PATH：

```text
export PATH=/opt/verilator-4.210/bin:$PATH
hash -r
which verilator
verilator --version
```

验证结果为：

```text
/opt/verilator-4.210/bin/verilator
Verilator 4.210 2021-07-07 rev v4.210
```

由此确认：软件已安装、Shell 能够找到软件、构建系统实际调用该软件是三个不同状态。切换工具版本后，必须同时检查可执行文件路径与版本输出。

### 3.3 SystemVerilog DPI C 接口类型冲突的误判

较早的一次编译中，DiffTest C++ 接口与 Verilator 自动生成的 VSimTop__Dpi.h 之间出现函数声明类型冲突，涉及 flash_read、difftest_ram_read、difftest_ram_write、sd_read、sd_write、pte_helper 和 amo_helper 等函数，表面表现为 uint64_t 与 unsigned long long 等类型差异。

当时一度怀疑 Verilator 4.210 与 SystemVerilog DPI-C 接口的生成方式不兼容。复核后发现，该次构建实际仍由 4.038 执行，因此现有证据不能支持“4.210 与 DiffTest 不兼容”的判断。最终没有修改 VSimTop__Dpi.h 或其他自动生成的 DPI-C 接口文件。

这一过程说明：自动生成代码报错时，应先确认生成工具、实际版本与生成条件。直接修改生成文件只能形成一次性补丁，重新生成后仍会丢失。

### 3.4 Chisel 设计展开警告的判断

重新构建时，Chisel 设计展开阶段出现 148 条警告，例如：

```text
[warn] ... MSHR.scala ... [W004] Dynamic index ...
[warn] ... AXI4IntrGenerator.scala ... [W005] ...
There were 148 warning(s) during hardware elaboration.
```

虽然警告数量较多，但流程完成了对应任务，同时以下文件已成功生成：

```text
ls -lh build/rtl/SimTop.sv
```

build/rtl/SimTop.sv 的文件大小约为 1.2 MB。由此可确认，当前 Chisel 设计展开与 RTL 生成阶段已完成，警告不是本次构建终止的直接原因。后续排查范围因此收敛到 Verilator 与 C++ 仿真程序编译阶段。

### 3.5 本地处理 UNOPTFLAT 警告

真正切换到 Verilator 4.210 后，构建仍因 48 条 UNOPTFLAT 警告退出。检查当前工程的 difftest/verilator.mk 后发现，构建参数已包含 -Wno-STMTDLY 和 -Wno-WIDTH，但没有包含 -Wno-UNOPTFLAT。本地将参数调整为：

```text
-Wno-STMTDLY -Wno-WIDTH -Wno-UNOPTFLAT
```

该修改仅改变 Verilator 对 UNOPTFLAT 警告的处理方式，不改变 XiangShan RTL 的功能逻辑，也未修改 build/rtl/*.sv。它属于本地构建兼容性处理，不应表述为 XiangShan 的通用或官方配置。

需要说明的是，屏蔽警告并不等于消除其根因。若后续仿真出现不收敛、异常耗时或相关逻辑行为异常，应恢复该警告并结合具体信号进一步分析。本次构建和 Hello XiangShan 运行均成功完成，因此该处理满足当前阶段的闭环验证要求。

### 3.6 NOOP_HOME 为空导致路径展开错误

随后出现：

```text
No rule to make target '/build/rtl/SimTop.sv'
```

实际文件位于：

```text
/home/dongh/github/xs-env/XiangShan/build/rtl/SimTop.sv
```

检查 Makefile 后发现，DiffTest 通过 DESIGN_DIR=$(NOOP_HOME) 获取设计目录，而当时 NOOP_HOME 为空。因此，$(NOOP_HOME)/build/rtl/SimTop.sv 被展开成 /build/rtl/SimTop.sv。

处理与验证命令如下：

```text
export NOOP_HOME=/home/dongh/github/xs-env/XiangShan
echo "$NOOP_HOME"
ls -lh "$NOOP_HOME/build/rtl/SimTop.sv"
```

后续通过 source ~/github/xs-env/env.sh 设置全部环境变量。对于“已存在文件却提示找不到”的问题，应先检查构建系统展开后的完整路径，而不是立即重新生成文件。

### 3.7 Verilator 4.210 不识别 instr count dpi

修复环境变量并处理 UNOPTFLAT 警告后，构建出现新的首要错误：

```text
%Error: Invalid option: --instr-count-dpi
```

直接检查 Verilator 4.210：

```text
verilator --help 2>&1 | grep instr-count-dpi
/opt/verilator-4.210/bin/verilator --instr-count-dpi 1 --version
```

第一条命令没有匹配结果，第二条命令报告参数无效，说明当前安装的 4.210 不支持 --instr-count-dpi。

检查当前 DiffTest 构建脚本可见，当版本判断满足 >= 4.210 时，脚本会加入该参数。由于本地 4.210 实际不识别该选项，形成了明确的版本与构建参数不匹配。与其直接删除项目预期使用的参数，本次选择继续升级到 Verilator 5.x。

### 3.8 正确安装并验证 Verilator 5.020

首次安装 5.020 时，which verilator 显示 /opt/verilator-5.020/bin/verilator，但 verilator --version 仍输出 4.210。检查源码仓库后得到：

```text
HEAD detached at v4.210
nothing to commit, working tree clean
v4.210
8e2ba6a00 (HEAD, tag: v4.210) Version bump
```

根因是执行 ./configure --prefix=/opt/verilator-5.020 时，源码工作树仍停留在 v4.210。--prefix 只决定安装目录，不决定源码版本，因此 4.210 被安装到了名为 verilator-5.020 的目录中。

随后重新切换源码版本并安装：

```text
cd ~/github/verilator
git fetch --tags
git checkout v5.020
git describe --tags --always
```

安装完成后，通过以下三项共同确认版本：

```text
git describe --tags --always
which verilator
verilator --version
```

并验证关键参数：

```text
verilator --help 2>&1 | grep instr-count-dpi
```

输出中出现：

```text
--instr-count-dpi <value>  Assumed dynamic instruction count of DPI imports
```

由此确认 Verilator 5.020 已生效且支持构建所需参数。

### 3.9 切换版本后的增量清理与重新构建

build/rtl/SimTop.sv 已由 Chisel 设计展开阶段成功生成，因此切换 Verilator 版本后，仅清理与 Verilator 版本相关的中间产物：

```text
rm -rf build/emu-compile
```

该目录包含 VSimTop*.cpp、VSimTop*.h 和 VSimTop.mk 等生成文件，应由 Verilator 5.020 重新生成。保留 build/rtl/SimTop.sv 避免了重复执行耗时较长的设计展开与 RTL 生成。

重新构建前完成环境检查：

```text
export NOOP_HOME=/home/dongh/github/xs-env/XiangShan
export PATH=/opt/verilator-5.020/bin:$PATH
hash -r
which verilator
verilator --version
echo "$NOOP_HOME"
```

随后执行构建并保存完整输出：

```text
make emu 2>&1 | tee emu_build.log
```

### 3.10 C++ 编译警告与构建完成判据

C++ 编译阶段出现以下警告：

```text
warning: ignoring return value of 'char* fgets(char*, int, FILE*)'
warning: ignoring return value of 'size_t fread(void*, size_t, size_t, FILE*)'
```

此外还有与 __builtin___snprintf_chk 相关的说明信息。这些内容属于编译器对返回值未检查或潜在输出长度的静态诊断。日志中没有出现 fatal error:、collect2: error:、Killed 或紧随其后的 make: *** Error，各级 Makefile 正常退出，最终生成 build/emu。

因此，本阶段没有为消除这些非阻断性警告而修改 DiffTest 源代码。判断构建是否成功应以最早的阻断性错误、命令退出状态和目标文件是否生成共同为依据，不能只根据是否出现 warning 判断。

## 4 Hello XiangShan 测试负载编译

### 4.1 首次编译失败

进入 nexus-am 示例目录并执行：

```text
cd "$AM_HOME/apps/hello"
make ARCH=riscv64-xs
```

关键输出为：

```text
# Building hello [riscv64-xs] with AM_HOME {/home/dongh/github/xs-env/nexus-am}
# Building lib-am [riscv64-xs]
+ AS src/nemu/common/mainargs.S
make[2]: riscv64-unknown-linux-gnu-gcc: No such file or directory
make[2]: *** [...] Error 127
make[1]: *** [Makefile:6: default] Error 2
make: *** [...] Error 2
```

其中，第一条真正导致失败的信息是 riscv64-unknown-linux-gnu-gcc: No such file or directory。Shell 返回的 Error 127 表示待执行命令未找到；后续两个 Error 2 是下层构建失败后逐级向上传递的结果，并非三个独立故障。

### 4.2 根因是交叉工具链前缀不一致

which riscv64-unknown-linux-gnu-gcc 没有输出，最初一度判断为系统未安装 RISC-V 交叉编译器。进一步检查 xs-env 的安装脚本后发现，系统安装的是 Ubuntu/Debian 提供的 riscv64-linux-gnu- 前缀工具链：

```text
which riscv64-linux-gnu-gcc
riscv64-linux-gnu-gcc --version
which riscv64-linux-gnu-g++
riscv64-linux-gnu-g++ --version
```

实际版本为：

```text
/usr/bin/riscv64-linux-gnu-gcc
riscv64-linux-gnu-gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0
/usr/bin/riscv64-linux-gnu-g++
riscv64-linux-gnu-g++ (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0
```

因此，交叉工具链并未缺失，真正的问题是 nexus-am 默认查找的命令前缀与系统现有工具链前缀不一致。

### 4.3 使用项目提供的配置开关

检查 am/arch/isa/riscv64.mk 可见：

```text
MARCH ?= rv64gc
ifeq ($(LINUX_GNU_TOOLCHAIN),1)
CROSS_COMPILE := riscv64-linux-gnu-
else
CROSS_COMPILE := riscv64-unknown-linux-gnu-
endif
COMMON_FLAGS := -fno-pic -march=$(MARCH) -mcmodel=medany
CFLAGS += $(COMMON_FLAGS) -static
ASFLAGS += $(COMMON_FLAGS) -O0
LDFLAGS += -melf64lriscv
```

nexus-am 已通过 LINUX_GNU_TOOLCHAIN 支持两种命令前缀，因此无需修改 riscv64.mk、建立伪装命令的软链接或重新编译一套 GNU 工具链。本次采用项目原生配置：

```text
make ARCH=riscv64-xs LINUX_GNU_TOOLCHAIN=1
```

最终生成：

```text
$AM_HOME/apps/hello/build/hello-riscv64-xs.bin
```

## 5 运行验证与日志保存

### 5.1 加载并执行测试负载

进入 XiangShan 工程目录并执行：

```text
cd "$NOOP_HOME"
./build/emu \
  -i "$AM_HOME/apps/hello/build/hello-riscv64-xs.bin" \
  --no-diff
```

其中，-i 指定待加载的 RISC-V 二进制测试负载；--no-diff 表示本次快速验证不启用差分测试。运行过程中成功出现：

```text
Hello XiangShan
```

这表明 nexus-am 源码已完成 RISC-V 交叉编译，生成的二进制文件已被 XiangShan RTL 仿真程序加载并执行，字符输出路径正常。

### 5.2 终端中看不到输出的原因

程序输出 Hello XiangShan 后，仿真程序继续打印大量性能计数器信息，例如：

```text
[PERF ][time=10423] SimTop.l_soc.l3cacheOpt.slices_0.mshrAlloc: mshr_latency_max, 56
[PERF ][time=10423] SimTop.l_soc.l3cacheOpt.slices_1.mshrAlloc: mshr_latency_max, 52
Seed=0 Guest cycle spent: 10,424
Host time spent: 62,877ms
```

因此，Hello XiangShan 被后续输出顶出终端当前可见区域。该现象不表示程序没有执行或没有输出。

### 5.3 保存并检索完整日志

将标准输出和标准错误合并后同时显示并写入文件：

```text
cd "$NOOP_HOME"
./build/emu \
  -i "$AM_HOME/apps/hello/build/hello-riscv64-xs.bin" \
  --no-diff \
  2>&1 | tee hello_run.log
```

直接定位关键输出：

```text
grep -ni "Hello XiangShan" hello_run.log
grep -ni -B 10 -A 10 "Hello XiangShan" hello_run.log
```

该记录方式既保留实时终端输出，也保存完整的标准输出与标准错误，适合后续提交关键片段并保留原始日志作为复核依据。

## 6 误判与修正汇总

| 现象 | 初步判断或风险 | 核实后的结论与处理 |
| --- | --- | --- |
| 大量 UNOPTFLAT | 生成的 RTL 可能存在功能错误 | 先核对 Verilator 版本与警告处理；本地屏蔽警告后完成闭环验证，同时保留后续复查边界 |
| 安装 4.210 后仍失败 | 默认认为 4.210 已生效 | PATH 仍命中 /usr/bin/verilator 4.038 |
| DPI-C 接口类型冲突 | 怀疑 4.210 与 DiffTest 不兼容 | 当次生成工具实际仍为 4.038，证据不足；未修改生成文件 |
| Chisel 产生 148 条警告 | 设计展开可能失败 | SimTop.sv 已生成，阻断点位于后续阶段 |
| /build/rtl/SimTop.sv 不存在 | 准备重新生成 RTL | NOOP_HOME 为空导致路径展开错误 |
| --instr-count-dpi 无效 | 考虑删除构建参数 | 本地 4.210 不支持该参数，改用 5.020 |
| 安装目录名为 verilator-5.020 | 默认认为程序版本为 5.020 | 源码仍处于 v4.210，目录名不能作为版本证据 |
| 找不到 riscv64-unknown-linux-gnu-gcc | 认为交叉工具链未安装 | 系统已安装 riscv64-linux-gnu- 前缀工具链 |
| 终端未看到 Hello 输出 | 怀疑测试程序未运行 | 输出被后续性能日志顶出可见区域，日志检索可确认 |

## 7 形成的工程排障方法

本阶段形成的排障流程如下：

- 从日志中找到第一条阻断性错误，不把上层 Makefile 的连锁返回码当作多个独立故障；
- 区分 warning、note、error、fatal error 与进程被系统终止等不同状态；
- 判断故障属于设计展开、RTL 生成、Verilator、C++ 编译、Makefile、环境变量、交叉工具链或运行时中的哪一层；
- 使用 which、--version、Git 标签和完整命令行确认实际执行的软件与源码版本；
- 优先阅读项目提供的 Makefile 与配置开关，采用范围最小且可验证的处理；
- 避免手工修改自动生成文件；
- 保留已确认正确且生成成本较高的中间产物，仅清理与故障相关的目录；
- 保存完整日志，在技术文档中只引用支持结论的关键片段。

## 8 最终结果

| 验证项 | 结果 | 验证依据 |
| --- | --- | --- |
| xs-env 环境变量配置 | PASS | NOOP_HOME、AM_HOME 等变量指向正确目录 |
| Chisel 设计展开与 RTL 生成 | PASS | build/rtl/SimTop.sv 已生成 |
| Verilator 版本与参数支持 | PASS | 版本为 5.020，帮助信息包含 --instr-count-dpi |
| C++ 仿真模型编译 | PASS | build/emu 已生成 |
| RISC-V 交叉工具链识别 | PASS | riscv64-linux-gnu-gcc 11.4.0 可用 |
| nexus-am Hello 编译 | PASS | hello-riscv64-xs.bin 已生成 |
| 测试负载加载与执行 | PASS | emu 成功运行并输出 Hello XiangShan |
| 日志留存 | PASS | emu_build.log 与 hello_run.log 已保存 |

## 9 工程结论与后续工作

本阶段的主要问题不是单一依赖缺失，而是工具版本、Shell 路径优先级、源码 Git 标签、Verilator 参数兼容性、环境变量展开以及交叉工具链命令前缀等因素连续叠加。排障过程中最重要的判断依据始终是实际执行路径、版本输出、构建脚本行为、第一条阻断性错误和可验证的产物。

最终，XiangShan 的 SystemVerilog RTL、Verilator C++ 仿真模型、RTL 仿真可执行程序和 nexus-am 测试负载均已成功生成；build/emu 实际加载并执行 hello-riscv64-xs.bin，输出 Hello XiangShan，完成了从 RTL 生成到目标程序执行的闭环验证。

现有环境已具备继续开展向量指令执行过程分析、波形采集和 vdot.vv 功能开发的基础条件。后续工作应优先记录仓库提交号、子模块版本与本地构建参数，使环境能够在其他机器或服务器上复现；同时应保留 -Wno-UNOPTFLAT 的本地修改记录，并在升级工程版本或出现仿真收敛问题时重新评估。

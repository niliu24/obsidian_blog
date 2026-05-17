---
tags: [cpu, microarchitecture]
type: entity
---

> CPU 微架构（Microarchitecture）是处理器的**硬件实现方案**，规定了指令如何在物理电路中流动。经典的 **5 阶段流水线**是理解一切微架构的基石——从它出发，可以理解冒险、转发、分支预测、乱序执行、超标量和 SMT。

## 5 阶段经典流水线

典型的 RISC 流水线将指令执行拆分为 5 个阶段，每阶段一个时钟周期：

```
时钟周期:  T1    T2    T3    T4    T5    T6    T7    T8    T9
指令 i:   [IF]  [ID]  [EX]  [MEM] [WB]
指令 i+1:       [IF]  [ID]  [EX]  [MEM] [WB]
指令 i+2:             [IF]  [ID]  [EX]  [MEM] [WB]
指令 i+3:                   [IF]  [ID]  [EX]  [MEM] [WB]
```

| 阶段 | 名称 | 操作 |
|------|------|------|
| **IF** | Instruction Fetch | 从 I-cache 取指令，PC 自增 |
| **ID** | Instruction Decode | 译码，读取寄存器，生成立即数 |
| **EX** | Execute | ALU 运算或地址计算 |
| **MEM** | Memory Access | 读/写 D-cache（仅加载/存储指令） |
| **WB** | Write Back | 将结果写回寄存器堆 |

理想情况下每周期完成 1 条指令（IPC = 1），但**流水线冒险**会破坏这一目标。

## 流水线冒险 (Hazards)

### 结构冒险 (Structural Hazard)

硬件资源冲突，如同一周期内 IF 和 MEM 同时访问同一级缓存。

> 现代 CPU 通过独立的指令缓存 (I-cache) 和数据缓存 (D-cache) 消除大多数结构冒险。

### 数据冒险 (Data Hazard)

指令之间存在数据依赖时，由于流水线重叠导致读到的值不正确。

| 类型 | 全称 | 示例 | 说明 |
|------|------|------|------|
| **RAW** | Read After Write | `add x1, x2, x3; sub x4, x1, x5` | 后一条读前一条的结果（**真依赖**） |
| **WAR** | Write After Read | `sub x1, x2, x3; add x2, x4, x5` | 后一条写前一条读的位置 |
| **WAW** | Write After Write | `add x1, x2, x3; sub x1, x4, x5` | 两条写同一目标寄存器 |

> 在 5 阶段流水线中，RAW 冒险最常见且最关键。WAR 和 WAW 在顺序流水线中较少见，但在乱序执行中需专门处理。

### 控制冒险 (Control Hazard)

分支指令导致下一条指令地址不确定。这是性能损失最大的冒险类型。

### 转发/旁路 (Forwarding / Bypassing)

RAW 冒险的核心解法：将 ALU 计算结果直接从 EX 阶段输出**转发**到后续指令的 EX 输入，无需等待 WB 写回寄存器。

```
无转发: add  x1, x2, x3    [ID]→[EX]→[MEM]→[WB]
        sub  x4, x1, x5             等待 x1 ←          [ID] 读到旧值 ❌

有转发: add  x1, x2, x3    [ID]→[EX]→[MEM]→[WB]
        sub  x4, x1, x5         ←转发←          [ID] 读到新值 ✅
```

| 冒险类型 | 转发方案 | 是否需要停顿 |
|----------|---------|-------------|
| EX 到 EX（相邻 ALU 指令） | EX 阶段直接转发 | 否 |
| MEM 到 EX（加载后紧接使用） | MEM → EX 转发 | 否（但 Load-use 需 1 停顿） |
| Load-use | 转发 + 1 个气泡 | 需要 1 周期停顿 |
| 无法转发（如复杂操作） | 软件 NOP | 需要 |

### Load-use 冒险

加载指令的结果不能立即被下一条指令使用，因为数据在 MEM 阶段末尾才就绪：

```
lw   x1, 0(x2)    [IF] [ID] [EX] [MEM] [WB]
add  x4, x1, x5   [IF] [ID] [EX] ← 需要 x1，但还在 MEM ❌
                   ←插入气泡→   [STALL]
add  x4, x1, x5         [IF] [ID] [EX]  ← 正确
```

编译器可以通过**指令调度**（在 `lw` 后插入不相关的指令）来隐藏 load-use 延迟。

## 分支预测 (Branch Prediction)

### 静态预测

| 策略 | 描述 | 准确率 |
|------|------|--------|
| 永远不跳转 | 假设条件分支不跳转 | ~50% |
| 永远跳转 | 假设条件分支跳转 | ~50% |
| 向后跳转、向前不跳转 | 循环（backward）通常跳转 | ~70% |
| 编译器 hint | 编译器通过编码位给出预测 | 依赖编译器 |

### 动态预测

#### 1. 1 位饱和计数器

```
上次不跳 → 预测不跳；上次跳转 → 预测跳转。
问题：内层循环的最后一次会预测错误（两次错误）。
```

#### 2. 2 位饱和计数器（最常用）

```
         ┌──────────┐   不跳转   ┌──────────┐
         │  强不跳转  │◄──────────│  弱不跳转  │
         │ (00)      │           │ (01)      │
         └────┬─────┘           └────┬─────┘
              │ 跳转                  │ 跳转
              ▼                      ▼
         ┌──────────┐   不跳转   ┌──────────┐
         │  弱跳转    │──────────►│  强跳转    │
         │ (10)      │           │ (11)      │
         └──────────┘           └──────────┘
```

需要连续 2 次预测错误才会翻转预测方向，对循环的**最后一次**错误容忍度更好。

#### 3. 分支目标缓冲 (BTB)

记录分支指令的 PC 和目标地址的映射表。在 IF 阶段查询 BTB，若命中则直接预测目标地址。

| BTB 项 | 内容 |
|--------|------|
| 标签 (Tag) | 分支指令 PC 的部分高位 |
| 目标地址 (Target) | 上次跳转的目标 PC |
| 预测位 (Prediction) | 2 位饱和计数器状态 |

#### 4. 返回地址栈 (RAS)

专门预测函数返回地址。MIPS/ARM 的 `jalr` 或 x86 的 `ret` 指令。

```
CALL func:   将返回地址（下一条指令）压入 RAS
RET:         从 RAS 弹出地址作为预测目标
```

| RAS 深度 | 典型值 | 溢出策略 |
|----------|--------|---------|
| 16-32 项 | 覆盖大多数嵌套深度 | 丢掉最旧（或截断） |

#### 5. Tournament 预测器

同时维护两种预测器（通常为基于局部历史的 + 基于全局历史的），用一个**选择器**选择更准确的那个。

```
输入: 分支 PC + 全局历史
       │
  ┌────┴────┐
  │ 局部预测器 │    ┌────────┐
  │ (PHT)    │───►│ 选择器   │──► 最终预测
  └─────────┘    │ (Meta)  │
  ┌─────────┐    └────────┘
  │ 全局预测器 │───►
  │ (Gshare) │
  └─────────┘
```

> Alpha 21264 最早采用 Tournament 预测器，现代 Intel/AMD CPU 使用更复杂的 **TAGE** (TAgged GEometric) 预测器。

#### 6. 分支误预测惩罚 (Misprediction Penalty)

```
分支指令在 EX 阶段才得出实际方向。
误预测时需要冲刷（flush）流水线中预取的所有后续指令。

惩罚周期 ≈ 流水线级数（从 IF 到 EX 的深度）
```

| 流水线深度 | 误预测惩罚 |
|-----------|-----------|
| 5 级 | 2-3 周期 |
| 14 级 (Intel Core) | 10-12 周期 |
| 20+ 级 (ARM Cortex-A) | 15+ 周期 |

### 分支预测友好 vs 不友好代码

```c
// 分支预测友好：规则模式
int sum_array(int *arr, int n) {
    int sum = 0;
    for (int i = 0; i < n; i++) {      // 循环分支：大部分情况跳转 → 预测一致
        if (arr[i] > 0) {               // 正负数分布随机时 → 预测困难
            sum += arr[i];
        }
    }
    return sum;
}
```

```c
// 分支预测不友好：不可预测模式
int sum_random_threshold(int *arr, int n, int threshold) {
    int sum = 0;
    for (int i = 0; i < n; i++) {
        if (arr[i] > threshold) {       // threshold 为 0，数据随机
            sum += arr[i];               // 50% 概率跳转/不跳转 → 预测准确率 ~50%
        }
    }
    return sum;
}
```

```c
// 分支预测友好：使用无分支算法
int sum_abs_signed(int *arr, int n) {
    int sum = 0;
    for (int i = 0; i < n; i++) {
        int mask = arr[i] >> 31;        // 符号位扩展掩码
        sum += (arr[i] ^ mask) - mask;  // 绝对值运算，无分支
    }
    return sum;
}
```

> 现代 CPU 的 TAGE 预测器对规则模式可以达到 >99% 的准确率，但对随机数据的准确率仍接近 50%。

## 乱序执行 (Out-of-Order Execution)

### Tomasulo 算法

Tomasulo 算法是乱序执行的经典实现，最早在 IBM System/360 Model 91 上使用。核心思想：**只要有可用资源和源数据，指令就可以执行**，不受程序顺序约束。

```
         ┌──────────────────────┐
         │   指令发射队列 (Issue)  │
         └──────────┬───────────┘
                    │
         ┌──────────▼───────────┐
         │   Reservation Stations │  ← 等待操作数就绪
         │   (保留站)              │
         └──┬───────┬───────┬───┘
            │       │       │
     ┌──────▼──┐ ┌──▼────┐ ┌▼──────┐
     │ ALU RS  │ │ ALU RS│ │MEM RS │
     └──────┬──┘ └──┬────┘ └───┬───┘
            │       │          │
     ┌──────▼───────▼──────────▼───┐
     │   执行单元 (Execution Units)   │
     └──────┬──────────────────┬───┘
            │                  │
     ┌──────▼──────┐  ┌───────▼──────┐
     │  Common Data  │  │     ROB       │
     │  Bus (CDB)    │  │  (Reorder      │
     │  广播结果      │  │   Buffer)      │
     └──────────────┘  └───────┬───────┘
                               │
                        ┌──────▼───────┐
                        │   提交 (Commit) │
                        └──────────────┘
```

| 组件 | 功能 |
|------|------|
| **保留站 (RS)** | 等待操作数就绪并监控 CDB，操作数齐备即发射 |
| **公共数据总线 (CDB)** | 广播计算结果，所有 RS 同时监听（数据旁路） |
| **重排序缓冲 (ROB)** | 保存乱序执行结果，按程序顺序提交（提交 → 写回寄存器或内存） |
| **寄存器重命名表** | 将逻辑寄存器映射到物理寄存器，消除 WAR/WAW |

### 寄存器重命名

消除 WAR 和 WAW 冒险的关键：为每个逻辑寄存器分配不同的物理寄存器。

```
// 无需重命名 —— x1 被不同函数使用，存在 WAR 冒险
mul x1, x2, x3     // 写 x1
add x2, x1, x4     // 读 x1 ❌ 这里的 x1 是旧值还是新值？ → 需要重命名
```

重命名后，`add` 读取的是映射到物理寄存器 P17 的 `x1`，而 `div` 写的是 P42：

```
// 硬件重命名后
mul P17, P02, P03  // x1 → P17
add P02, P17, P04  // x1 → P17（读取 mul 的结果）

// 后续指令重用 x1
div P42, P05, P06  // x1 → P42（新的物理寄存器）
add P07, P42, P08  // x1 → P42（读取 div 的结果）
```

> 物理寄存器数量远多于架构寄存器（如 Intel Core：~200 个物理寄存器 vs 16 个架构寄存器）。

### 乱序执行示例

```c
// 待执行的 RISC-V 指令序列
mul  x1, x2, x3    // 乘法，6 周期延迟
add  x4, x1, x5    // 等待 x1（RAW 依赖）
lw   x6, 0(x7)     // 加载，4 周期延迟
addi x8, x6, #1    // 等待 x6（RAW 依赖）
add  x9, x10, x11  // 无依赖，可立即执行
```

| 周期 | 发射 | 执行 | 说明 |
|------|------|------|------|
| T1 | `mul` + `add` + `lw` + `addi` + `add` | — | 所有指令同时译码并分发到保留站 |
| T2 | — | `add x9, x10, x11` | 无依赖，立即执行 |
| T3-T4 | — | `lw x6, 0(x7)` | 加载执行 |
| T5 | — | `addi x8, x6, #1` | 通过 CDB 获得 x6，立即执行 |
| T6-T10 | — | `mul x1, x2, x3` | 乘法长延迟 |
| T11 | — | `add x4, x1, x5` | 通过 CDB 获得 x1，立即执行 |

乱序执行让 `add x9` 在 T2 就完成，而 `mul` 要到 T6 才开始。

## 超标量 (Superscalar)

### 多发射 (Multiple Issue)

每周期发射多条指令，指令宽度（Width）决定并行度。

| 微架构 | 发射宽度 | 典型 IPC |
|--------|---------|---------|
| Intel Pentium | 2 | < 1.5 |
| Intel Core i7 (Skylake) | 6 (4 ALU + 2 Load/Store) | 2-4 |
| ARM Cortex-A78 | 5 | 1.5-3 |
| Apple M1 (Firestorm) | 8 | 3-5 |

### VLIW vs 超标量

| 特性 | VLIW (Very Long Instruction Word) | 超标量 (Superscalar) |
|------|-----------------------------------|----------------------|
| 并行安排 | **编译时**静态安排 | **运行时**硬件动态安排 |
| 硬件复杂度 | 低（简单译码） | 高（依赖检测、重命名、调度） |
| 代码兼容性 | 差（不同硬件需重新编译） | 好（二进制兼容） |
| 典型代表 | Itanium (IA-64)、DSP | x86 Core、ARM Cortex、Apple Silicon |
| 编译器负担 | 重 | 轻 |

## 同步多线程 (SMT / Hyper-Threading)

一个物理核提供多个逻辑核，共享执行资源但维护独立的状态（PC、寄存器）。

```
物理核逻辑视图:

┌────────────────────────────────────────┐
│              物理核                      │
│  ┌─────┐ ┌─────┐  ┌───┐  ┌───┐       │
│  │L1 I$│ │L1 D$│  │ALU│  │ALU│       │
│  └──┬──┘ └──┬──┘  │ FP│  │LS │       │
│     │       │     └───┘  └───┘       │
│  ┌──┴───────┴────────────────────┐   │
│  │  逻辑核 0 (线程 A) 状态: PC, Regs │   │
│  │  逻辑核 1 (线程 B) 状态: PC, Regs │   │
│  │           共享执行资源             │   │
│  └─────────────────────────────────┘   │
└────────────────────────────────────────┘
```

| 特性 | 说明 |
|------|------|
| 实现 | Intel: Hyper-Threading (2 线程/核)；IBM POWER: SMT4/8 |
| 收益 | 单核性能提升 15-30% |
| 代价 | ~5% 面积增加，每个逻辑核需完整寄存器文件和状态 |
| 共享资源 | ALU、FPU、L1/L2 缓存、TLB |
| 专用资源 | 架构寄存器、PC、返回地址栈、中断控制器状态 |
| 安全影响 | SMT 线程之间可通过缓存侧信道（如 Spectre）攻击 |

```c
// 多线程场景：SMT 提升负载利用率
// 线程 A：计算密集型（ALU 繁忙）
void thread_a() {
    while (1) {
        result_a[i] = heavy_compute(data[i++]); // ALU 饱和
    }
}
// 线程 B：内存密集型（cache miss 时 ALU 空闲）
void thread_b() {
    while (1) {
        result_b[j] = data_b[large_stride[j++]]; // 大量 cache miss
    }
}
// SMT 在线程 B 等待内存时，SMT 将 ALU 资源让给线程 A
```

## 来源

* Hennessy & Patterson, *Computer Architecture: A Quantitative Approach* (6th ed.), Morgan Kaufmann, 2017
* Patterson & Hennessy, *Computer Organization and Design (RISC-V Edition)*, Morgan Kaufmann, 2021
* Tomasulo, R. M., "An Efficient Algorithm for Exploiting Multiple Arithmetic Units", *IBM Journal*, 1967
* Intel, *Intel 64 and IA-32 Architectures Optimization Reference Manual*, 2023
* Fog, Agner, *The Microarchitecture of Intel, AMD and VIA CPUs*, Copenhagen University College of Engineering, 2022
* ARM, *ARM Cortex-A78 Core Technical Reference Manual*, 2021

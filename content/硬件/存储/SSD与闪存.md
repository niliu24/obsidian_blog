---
tags: [storage, ssd, nand-flash]
type: entity
---

> NAND Flash 是当前 SSD 的核心存储介质，是一种非易失性、电可擦除的可编程只读存储器 (EEPROM)。它以页 (Page) 为读写单位、以块 (Block) 为擦除单位，这种 " 写前擦除 " (Erase-Before-Write) 的特性深刻影响了现代存储软件栈的设计。

## NAND Flash 物理结构

### 芯片层次结构

```
NAND 芯片 (Chip/Package)
  └── Target (Die 选择)
        └── Die (晶粒，也称 LUN, Logical Unit Number)
              └── Plane (平面，有独立的页缓存寄存器)
                    └── Block (块，擦除的最小单位，含 256~1024 Pages)
                          └── Page (页，读写的最小单位，4KB/8KB/16KB)
                                └── Cell (存储单元，即浮栅晶体管)
```

### 各层级规格例示

| 层级 | 容量例 | 说明 |
|------|--------|------|
| 1 Chip (Package) | 512 GB - 2 TB | 封装级，含多 Die |
| 1 Die (LUN) | 128 - 512 GB | 独立 CE (Chip Enable) 信号 |
| 1 Plane | 8 - 32 GB | 有独立 Sense Amplifier 和 Page Buffer |
| 1 Block | 2 - 16 MB | 擦除单位，典型 256-1024 Pages |
| 1 Page | 4 - 16 KB | 读写单位，含 Data + Spare (ECC + Metadata) |
| 1 Cell | 1-4 bit | SLC/MLC/TLC/QLC 不等 |

## 读 / 写 / 擦除不对称

NAND Flash 的读、写（编程）、擦除操作在时间和能耗上高度不对称：

| 操作 | 典型耗时 | 单位 | 说明 |
|------|---------|------|------|
| 读 (Read) | ~20-100 μs | Page | 相对最快 |
| 写/编程 (Program) | ~200-1500 μs | Page | 比读慢 10-20x |
| 擦除 (Erase) | ~2-15 ms | Block | 最慢，且必须整块擦除 |

```
操作延迟对比（典型 TLC NAND）:
Read    ██░░░░░░░░░░░░░░   ~50 μs
Program ██████████████░░   ~800 μs
Erase   ████████████████████████████████████  ~4 ms
```

### 不能原地更新的原因

NAND Flash **不能**像 HDD 那样直接覆盖写入已有数据，因为：

1. **电压干扰风险**：编程（写）需要将电子注入浮栅，擦除需要将电子拉出（通过隧道效应）
2. **物理限制**：NAND 的写操作只能将 1→0（NAND 逻辑），将 0→1 必须经过块擦除
3. **级联效应**：每个 Cell 的编程电压必须精确控制，直接覆盖会破坏相邻 Cell 状态

因此，SSD 必须通过 FTL (Flash Translation Layer) 实现**异地更新 (Out-of-Place Update)**：将新数据写入新位置，旧位置标记为失效 (Stale)，随后通过垃圾回收释放空间。

## SLC / MLC / TLC / QLC 对比

| 类型 | bits/cell | 电压状态数 | P/E 循环 (耐久度) | 典型成本 | 读取速度 | 主要用途 |
|------|-----------|-----------|-----------------|---------|---------|---------|
| SLC | 1 | 2 | ~50,000-100,000 | 最高 | 最快 | 企业缓存、工业 |
| MLC | 2 | 4 | ~5,000-10,000 | 高 | 较快 | 企业 SSD (已淘汰) |
| TLC | 3 | 8 | ~1,000-3,000 | 中 | 中等 | 消费级/企业主流 |
| QLC | 4 | 16 | ~300-1,000 | 低 | 较慢 | 大容量读取密集 |
| PLC (规划) | 5 | 32 | ~100-300 | 最低 | 最慢 | 归档存储 |

```
SLC:   ────┐ Elevated
            │
MLC:   ────┬───┬───┬───   4 个电压层
           │   │   │
TLC:   ────┬─┬─┬─┬─┬─┬─┬─ 8 个电压层
           │ │ │ │ │ │ │
QLC:   ────┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬┬ 16 个电压层 (区分难度大增)
```

> **核心趋势**：随着每 Cell 位数增加，密度上升、成本下降，但耐久度、性能和读干扰灵敏度都劣化。PLC (5-bit/cell) 已进入研发阶段，但可靠性挑战极大。

## FTL (Flash Translation Layer)

FTL 是 SSD 的固件核心，负责将 Host 的 LBA (Logical Block Address) 请求映射到 NAND 的 PBA (Physical Block Address)。没有 FTL，Host 无法直接操作具有 " 写前擦除 " 特性的 NAND。

### 核心功能

```
Host (LBA 请求)
    │
    ▼
┌──────────────────────────────┐
│         FTL 固件              │
│  ┌──── LBA → PBA 映射 ────┐  │
│  │  地址映射表 (Map Table)  │  │
│  ├──── 垃圾回收 (GC) ──────┤  │
│  ├──── 磨损均衡 (WL) ──────┤  │
│  ├──── 坏块管理 (BBM) ─────┤  │
│  └──── ECC 纠错 ──────────┘  │
└──────────────────────────────┘
    │
    ▼
NAND Flash (PBA 操作)
```

### 地址映射 (LBA → PBA)

| 映射粒度 | 优点 | 缺点 | 典型场景 |
|---------|------|------|---------|
| 页级映射 | 灵活，写入放大低 | 映射表大 | 高性能 SSD |
| 块级映射 | 映射表小 | 写入放大高 | 低端/嵌入式 |
| 混合映射 | 折中 | 复杂度高 | 早期方案 |

映射表大小估算（4KB 页 + 1TB SSD）：

```
1 TB / 4 KB = 2.56 亿项
每项 4 字节 (L2P 映射项) → 映射表 ~1 GB
→ 需要 DRAM 缓存映射表，或分级映射 (如 L1/L2 两级)
```

### 垃圾回收 (GC, Garbage Collection)

当有效数据占比降低时，FTL 需要回收 Block：

```
GC 前 (Block 内有无效页):
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ V  │ I  │ V  │ I  │ I  │ V  │ I  │ I  │  (V=有效, I=无效/废弃)
└────┴────┴────┴────┴────┴────┴────┴────┘
         │
         ▼ GC 操作:
         1. 读取所有有效页到 Buffer
         2. 将有效页写入新 Block
         3. 擦除原 Block

GC 后 (Block 已擦除，空闲):
Free Block (可写入)
```

* **GC 效率**：Block 内有效页越少，GC 效率越高
* **写放大**：GC 过程中有效页的**回写**是写放大的主要来源之一
* **后台 GC**：SSD 空闲时执行，减少前台 GC 对性能的影响

### 磨损均衡 (Wear Leveling)

NAND Block 有 P/E 循环上限。磨损均衡确保所有 Block 被均匀磨损，延长 SSD 寿命。

| 策略 | 说明 | 效果 |
|------|------|------|
| 动态磨损均衡 | 分配新 Block 时选擦除次数少的 | 基础保障 |
| 静态磨损均衡 | 主动将冷数据从低擦除 Block 搬出，强迫该 Block 参与擦除 | 全面均衡 |

### 坏块管理 (BBM)

* **出厂坏块**：NAND 芯片出厂时就存在的坏块，记录在芯片的 Bad Block Table
* **运行坏块**：使用过程中产生的坏块，FTL 需要检测（EEC 纠错失败 ⇒ 重读 ⇒ 替换）
* **坏块替换**：从预留的 Over-Provisioning 区域中替换

### ECC 纠错

| 纠错技术 | 错误纠正能力 | 实现方式 | 典型应用 |
|---------|------------|---------|---------|
| BCH 码 | 可纠正多 bit 错 | 硬件解码器 | SLC/MLC |
| LDPC 码 | 更强纠错，软判决 | 需要多次重读 | TLC/QLC |
| RAID-like | Block/Die 级冗余 | 跨 Die 的 XOR 校验 | 企业级 SSD |

## 写放大 (Write Amplification)

### 定义

写放大因子 (WAF, Write Amplification Factor) 是 SSD 实际写入 NAND 的数据量与 Host 请求写入数据量之比：

```
WAF = 实际写入 NAND 的数据量 / Host 写入的数据量
```

### 影响因素

| 因素 | 影响 | 说明 |
|------|------|------|
| 写入块大小 | 小块写入 ⇒ 高 WA | 要求 4KB 更新 64KB Block ⇒ WA ~16x |
| GC 效率 | Block 有效数据越多 ⇒ WA 越高 | GC 需要先搬走有效页 |
| Over-Provisioning | OP 越大 ⇒ WA 越低 | 更多空闲块 = 更好的 GC 效率 |
| Trim (discard) | 减轻 FTL GC 负担 | Host 通知 SSD 哪些 LBA 已废弃 |

```
WAF 典型范围:
  理想情况 (大块顺序写):  WAF ≈ 1.0-1.5
  典型消费级负载:        WAF ≈ 2-5
  恶劣情况 (小块随机写):  WAF ≈ 10-50+
```

## Over-Provisioning (OP)

OP 是 SSD 预留的额外 NAND 容量（Host 不可见），用于 GC、磨损均衡、坏块替换。

```
256 GB SSD 的物理 vs 逻辑容量:
┌──────────────────────────────────────┐
│  Host 可见: 256 GB (LBA 范围)         │  ← 逻辑容量
├──────────────────────────────────────┤
│  Over-Provisioning: ~28 GB (~10%)    │  ← 额外物理 NAND
├──────────────────────────────────────┤
│  Spare Area (元数据): ~4 GB          │
└──────────────────────────────────────┘
  物理 NAND 总容量: ~288 GB
```

| OP 比例 | 典型场景 | WAF 改善 |
|---------|---------|---------|
| 0% | 极端廉价 SSD | 高 |
| 7-10% | 消费级 SSD 默认 | 中等 |
| 15-25% | 企业级 SSD | 低 |
| 50%+ | 极端写优化 (如 Optane) | 接近 1 |

## SLC Cache

TLC/QLC SSD 通常将部分 NAND 配置为 SLC 模式（1 bit/cell），作为**高速写入缓存**：

```
SLC Cache 工作原理:
1. 写入的数据先进入 SLC 区域（速度: ~500 MB/s 起）
2. SLC 满后：直接写入 TLC 区域（速度: ~100-300 MB/s）
3. 空闲时：后台将 SLC 数据 Compact 到 TLC（释放 SLC 空间）
```

**性能特征**：

```
写入速度 vs 写入量（典型 TLC SSD + SLC Cache）:
速度
│
│  ████████████████        ← SLC 模式: ~1000 MB/s
│                 ██       ← SLC 耗尽: 直写 TLC ~200-400 MB/s
│                   ███████ ← 持续写入
└────────────────────── 写入量
         ↑ SLC Cache 大小 (~10-50 GB)
```

* SLC Cache 大小：通常为 SSD 物理容量的 ~1-10%
* SLC Cache 按需调整（动态 SLC Cache）：SSD 空闲区块越多，SLC Cache 越大
* 消费级 SSD 的 SLC Cache 耗尽后写入速度急剧下降（这是测试中常见的 " 掉速 " 现象）

## SSD 稳态性能 vs 新鲜状态

```
写入性能随时间变化:
速度
100%
 │  ████████████████████         ← 新鲜状态 (FOB)
 │                          ← 下降
 │                    █████████  ← 稳态 (Steady State)
 └────────────────────────── 时间

新鲜状态 (FOB, Fresh Out of Box):
  - 所有 Block 已擦除，直接写入无需 GC
  - 性能最好，但**不代表真实使用**
  - 测试时经常出现"首次跑分高"

稳态性能 (Steady State):
  - SSD 中已写满数据，有有效/无效页混合
  - GC 持续运行，WAF 稳定
  - **真实世界性能** - 衡量 SSD 好坏的标准
```

> **SNIA 测试规范**要求：SSD 性能必须报告**稳态 (Steady State)** 指标，而非新鲜状态。购买/评估 SSD 时应重点关注稳态性能。

## fio 基准测试

`fio`（Flexible I/O Tester）是 Linux 上最流行的存储基准测试工具。

```bash
# 安装
sudo apt-get install fio

# 基本语法
fio [options] [jobfile]

# 测试随机 4K 读取 (IOPS 测试)
fio --name=randread \
    --ioengine=libaio \
    --iodepth=64 \
    --rw=randread \
    --bs=4k \
    --size=1G \
    --numjobs=1 \
    --runtime=60 \
    --time_based \
    --group_reporting \
    --direct=1  # 绕过 Page Cache

# 测试随机 4K 写入
fio --name=randwrite \
    --ioengine=libaio \
    --iodepth=64 \
    --rw=randwrite \
    --bs=4k \
    --size=1G \
    --numjobs=1 \
    --runtime=60 \
    --time_based \
    --group_reporting \
    --direct=1

# 测试顺序 128K 读取 (带宽测试)
fio --name=seqread \
    --ioengine=libaio \
    --iodepth=16 \
    --rw=read \
    --bs=128k \
    --size=4G \
    --numjobs=1 \
    --runtime=30 \
    --time_based \
    --group_reporting \
    --direct=1

# 混合读写测试 (70% 读, 30% 写)
fio --name=rwmix \
    --ioengine=libaio \
    --iodepth=32 \
    --rw=randrw \
    --rwmixread=70 \
    --bs=4k \
    --size=1G \
    --runtime=120 \
    --time_based \
    --group_reporting \
    --direct=1
```

### fio 输出解读

```bash
# 输出示例 (简化)
randread: (groupid=0, jobs=1): err= 0: pid=12345: Sat May 16 10:00:00 2026
  read: IOPS=185k, BW=724MiB/s (759MB/s)(42.5GiB/60001msec)
    slat (nsec): min=1025, max=34512, avg=1850.34, stdev=456.21
    clat (nsec): min=1562, max=89520, avg=8250.12, stdev=1200.45
    lat (usec): min=5, max=125, avg=10.12, stdev= 1.95
    cpu          : usr=12.50%, sys=35.20%, ctx=123456, majf=0, minf=12
  IO depths    : 1=0.1%, 2=0.2%, 4=0.4%, 8=1.0%, 16=2.0%, 32=5.0%, >=64=91.3%
    submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
    complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
    issued rwts: Total=11111111,0,0,0 Short=0,0,0,0
    latency   : target=0, window=0, percentile=0.00%, depth=64

Run status group 0 (all jobs):
   READ: bw=724MiB/s (759MB/s), 724MiB/s-724MiB/s (759MB/s-759MB/s), io=42.5GiB (45.6GB), run=60001-60001msec
```

> 关键指标: IOPS (随机读写性能), BW/Bandwidth (顺序吞吐), clat (完成延迟), slat (提交延迟)

## 来源

* [SNIA NAND Flash 101](https://www.snia.org/sites/default/files/ESF/SC13/SC13.SolidStateStorage.pdf) — SNIA Solid State Storage
* [Inside NAND Flash Memories](https://link.springer.com/book/10.1007/978-90-481-9431-5) — Springer, R. Micheloni et al.
* [Understanding Flash Write Amplification](https://www.snia.org/forums/sssi) — SNIA SSSI
* [NAND Flash Architecture and Specification Trends](https://www.flashmemorysummit.com/) — Flash Memory Summit
* [Linux kernel: drivers/md/ — DM-Cache, DM-Integrity](https://elixir.bootlin.com/linux/latest/source/drivers/md/)
* [fio 官方文档](https://fio.readthedocs.io/en/latest/fio_doc.html)
* [NAND Flash Data Recovery (ECC, LDPC)](https://ieeexplore.ieee.org/document/6871405)

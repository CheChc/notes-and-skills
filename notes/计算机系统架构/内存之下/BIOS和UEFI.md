---
tags:
  - BIOS
  - UEFI
  - 固件
  - 内存架构
created: 2026-08-14
updated: 2026-08-14
---

# BIOS 和 UEFI：Host 侧固件与内存初始化

> 这是"内存之下"的层级：NPU 的 HBM 初始化之前，**Host 侧固件（BIOS/UEFI）**需完成服务器硬件初始化。本文梳理 UEFI 启动流程、内存参考代码（MRC/FSP）如何初始化 Host 内存，以及 ACPI 如何为内核提供错误上报通道——**理解上游，才能定位 NPU 的 HBM 固件在整个系统中的位置**。
>
> 相关笔记：[[固件启动全流程]] · [[内核内存错误处理（CE与UCE）]] · [[从总线访问到PA]] · [[DRAM、SRAM、FLASH]]

---

## 一、BIOS → UEFI：为什么换代

| | 传统 BIOS | UEFI |
|---|----------|------|
| 接口宽度 | 16-bit 实模式 | 32/64-bit 保护模式 |
| 磁盘启动 | MBR（2TB 上限） | **GPT**（大容量） |
| 扩展性 | 中断向量 + INT 13h，难扩展 | **EFI 驱动/应用**，模块化 |
| 安全性 | 无 | **Secure Boot**（签名校验） |
| 网络/图形 | 基本 | UEFI Shell、图形界面、网络启动 |
| 现在 | 兼容模式（CSM） | **默认** |

**UEFI 本质**：一个运行在 CPU 上的**小型操作系统**——有内存管理、驱动模型、文件系统（FAT）、可执行格式（PE/COFF）、变量存储（NVRAM）。固件镜像存在 **NOR Flash**（见 [[DRAM、SRAM、FLASH]]）。

---

## 二、UEFI 启动流程（四阶段）

```mermaid
flowchart LR
    A["SEC<br/>(上电自检核心)"] --> B["PEI<br/>(Pre-EFI 初始化)"]
    B --> C["DXE<br/>(驱动执行环境)"]
    C --> D["BDS<br/>(启动设备选择)"]
    D --> E["OS Loader<br/>(引导系统)"]
```

| 阶段 | 干什么 | 与内存的关系 |
|------|--------|-------------|
| **SEC** | 第一条指令、CPU 微码、临时栈 | 还在用 cache as RAM（CAR）——**此时 DRAM 不可用** |
| **PEI** | 发现内存、**初始化内存控制器** | **MRC/FSP 在此完成 Host 主存初始化** |
| **DXE** | 加载驱动：PCIe、存储、网络、ACPI | 枚举设备、分配资源（NPU 的 BAR 在这里配） |
| **BDS** | 按启动策略找 OS Loader | 交接给内核 |

> **关键对照**：Host 主存的初始化发生在 **PEI 阶段的 MRC**；而 **NPU 板卡上 HBM 的初始化发生在板卡自己的固件里**（[[固件启动全流程]]）——两者互不干扰，但 Host 侧要通过 PCIe（[[PCIe：Host 与 NPU 的连接]]）把 NPU 使能起来，HBM 固件才能跑。

---

## 三、MRC / FSP：Host 侧的内存初始化（对照阅读）

- **MRC（Memory Reference Code）**：Intel 提供的初始化内存控制器/内存条的子程序，被 BIOS 调用。其职责与 [[固件启动全流程]] 高度对应：
  1. 探测 DIMM（SPD 读取：容量、时序、制造商）
  2. 配置控制器时序（tCK/tRCD/tRFC…）
  3. 跑 **memory training**（DQS/DQ 校准、边际测试）
  4. 校验与上报（BIST、ECC 初始化）
- **FSP（Firmware Support Package）**：Intel 把 MRC 等封成可复用的二进制模块，供第三方固件集成——"内存初始化外包给芯片厂商"的实践。
- 差异点：Host 内存是 **DIMM（SPD 可插拔、可替换）**，NPU 的 HBM 是**焊死在封装里（无 SPD，配置来自固件内置表）**。

---

## 四、ACPI：固件留给内核的"接口"

UEFI 固件除了启动系统，还要生成 **ACPI 表**，其中与内存 RAS 最相关的是：

| 表 | 全称 | 作用 |
|----|------|------|
| **HEST** | Hardware Error Source Table | 声明错误源（GHES 等） |
| **GHES** | Generic Hardware Error Source | 通用错误源，固件写 CPER 记录 |
| **BERT** | Boot Error Record Table | 启动阶段错误记录 |
| **ERST** | Error Record Serialization Table | 错误记录持久化 |
| **CPER** | Common Platform Error Record | 错误记录的统一封装格式 |

> 这套机制让 **NPU/板卡的 HBM 错误**也能进内核的统一处理路径（[[内核内存错误处理（CE与UCE）]]）：固件把硬件错误写成 CPER，内核 ghes 驱动读取，走 memory_failure 等标准流程。**BIOS/UEFI 不只是"开机"，还是错误上报链路的第一个节点。**

---

## 五、UEFI 与 NPU 的衔接点

1. **PCIe 枚举与资源分配**：DXE 阶段给 NPU 分配 MMIO BAR（包括 Resizable BAR，让 NPU 的显存/寄存器空间可达）——细节见 [[PCIe：Host 与 NPU 的连接]] §二。
2. **选项 ROM / 板卡固件**：NPU 板卡自己的固件（含 HBM 初始化）由板卡侧运行，Host 通过 PCIe 配置空间与其交互。
3. **电源管理**：ACPI 电源状态（D0-D3）、NPU 上电时序由 Host 固件编排。
4. **错误通道**：HEST/GHES 声明 NPU 的错误上报路径；PCIe 总线自身错误走 AER/DPC（[[PCIe：Host 与 NPU 的连接]] §五）。

```
UEFI（Host）── PCIe 枚举/BAR/电源 ──▶ NPU 板卡固件 ── HBM 初始化（[[固件启动全流程]]）
UEFI（Host）── ACPI HEST/GHES ──────▶ 内核 ghes ──▶ 内存错误处理（[[内核内存错误处理（CE与UCE）]]）
```

---

## 参考资料

- [UEFI 规范（uefi.org）](https://uefi.org/specifications)
- [UEFI Forum：UEFI 启动阶段介绍](https://uefi.org/sites/default/files/resources/UEFI_Plugfest_SD_2013_FirmwareInnovation.pdf)
- [Intel Firmware Support Package (FSP)](https://www.intel.com/content/www/us/en/developer/articles/tool/firmware-support-package.html)
- [ACPI 规范（HEST/GHES）](https://uefi.org/specs/ACPI/6.5/)
- 关联笔记：[[固件启动全流程]] · [[内核内存错误处理（CE与UCE）]] · [[从总线访问到PA]] · [[DRAM、SRAM、FLASH]]

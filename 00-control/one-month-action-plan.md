# 1 个月面试准备行动计划

## 当前共识

本阶段准备分三条线：

```text
A 线：MCU / SoC 裸机固件项目
B 线：Linux BSP / Driver 项目
C 线：MCU + Linux 共性底层能力
```

A 线和 B 线同等重要，都是核心项目经历；C 线不单独抢项目权重，用来打通底层思维，避免重复学习。

当前目标不是马上投递，而是在 1 个月内把项目证据、知识体系和面试表达整理到可面试状态。

## 总体目标

1 个月后需要达到：

- 能讲清 SmartFusion2/V3LoadX/MiV-RV32 项目，不虚、不散。
- 能讲清 Linux PCIe 驱动和用户态控制链路。
- 能把 MCU 与 Linux 在 MMIO、中断、异常、总线、调试上的共性和差异说清。
- 至少准备 2 个 MCU 侧问题定位案例，1-2 个 Linux 侧问题定位案例。
- 简历能形成 MCU 和 Linux 两条核心项目线，而不是一主一次。

## 学习方式

每个主题按以下闭环学习：

```text
复习已有材料 -> 画链路图/写概要 -> 子线程技术验收 -> 回主线程沉淀结论 -> 反哺简历
```

不建议：

- 只刷题，不整理项目。
- 只写项目，不补知识。
- 重复做已有 Linux-learning 实验。
- 一上来深挖内核源码大模块。

建议：

- 已经学过的内容以复习和面试化整理为主。
- 没有把握的内容放到子线程做技术检验。
- 主线程只收敛结论、证据和下一步计划。

## A 线：MCU / SoC 裸机固件

### A1. SmartFusion2 系统架构

目标：

- 能讲清 SmartFusion2 系统里 MSS、fabric、MiV-RV32、PL 寄存器、Flash/MRAM、CAN/RS422/Ethernet/UART/JTAG 等资源关系。
- 能说明 V3LoadX 是主工程，MiV-RV32 是子模块/辅助控制工程。

你需要输出：

- `02-projects/smartfusion2-architecture.md`

完成证明：

- 有一张系统架构图或文字版架构图。
- 能用 3 分钟讲清系统组成和两个 ELF 分工。
- 子线程可以根据文档追问你 5 个架构问题，你能答住。

学习方式：

- 先基于已有两个临时 MD 整理，不读大段代码。
- 必要时只补模块边界，不补实现细节。

### A2. V3LoadX 模块边界

目标：

- 区分正式使用、历史保留、预留/不确定模块。
- 明确哪些是你主写，哪些是厂商库/前人基础适配。

你需要输出：

- `02-projects/v3loadx-module-boundary.md`

完成证明：

- 每个主要模块能归类：
  - 正式使用
  - 历史保留
  - 预留/不确定
  - 厂商库/适配
- 简历中不会误写未启用或不确定模块。

学习方式：

- 先按模块名列清单，不急着解释代码。
- 对不确定模块明确标注“不确定”，后续再查。

### A3. 远程维护与可靠性链路

目标：

- 能讲清 Flash/IAP/CRC/DirectC/JTAG/主备/TMR/MRAM 的整体可靠性设计。
- 能说明哪些是厂商能力，哪些是你的适配和业务集成。

你需要输出：

- `02-projects/v3loadx-reliability-flow.md`

完成证明：

- 能画出升级/CRC/IAP 或 DirectC/JTAG 的流程。
- 能说明失败时如何回退、上报或恢复。
- 能明确 DirectC 算法库不是自研。

学习方式：

- 先从“流程图”入手，不从代码函数入手。
- 优先讲清工程价值：远程维护、无人值守、异常恢复。

### A4. MCU 侧问题定位案例

目标：

- 准备至少 2 个真实问题定位案例。

你需要输出：

- `02-projects/smartfusion2-issue-record-01.md`
- `02-projects/smartfusion2-issue-record-02.md`

完成证明：

每个案例包含：

```text
现象：
影响：
初始假设：
定位手段：
关键证据：
修复方案：
复盘：
```

学习方式：

- 不要求一开始写漂亮。
- 先列事实，再由主线程帮你整理成面试表达。

## B 线：Linux BSP / Driver

### B1. PCIe 完整链路

目标：

- 能从 PCIe 设备上电、枚举、probe、BAR/MMIO、MSI/中断讲到用户态状态管理和告警上报。

你需要输出：

- `02-projects/linux-pcie-flow.md`

完成证明：

- 能用 3 分钟讲清完整链路。
- 能解释 `endpoint.c` 在链路里的位置。
- 能说明用户态初始化、状态采集、告警分发如何接上内核驱动。

学习方式：

- 复习：
  - `/home/flamboy/linux-learning/docs/week-pcie/pcie_fpga_register_access_interview.md`
  - `/home/flamboy/linux-learning/docs/week-pcie/week-pcie_endpoint_notes.md`
  - `/home/flamboy/linux-learning/docs/week-pcie/week-pcie_pcie_access_chain_experiment.md`
- 不重做实验，先把链路图写出来。

### B2. PCIe ownership 边界

目标：

- 准确说明 `endpoint.c` 是在前人基础上持续改造维护，整个文件能讲清，但不包装成从 0 独立完成完整 PCIe 栈。

你需要输出：

- `02-projects/linux-pcie-ownership.md`

完成证明：

- 能列出：
  - 你改造维护过的部分
  - 共同维护的部分
  - 只是理解/联调的部分
  - 用户态你负责的部分

学习方式：

- 按模块职责整理，不按代码行数整理。
- 避免自我削弱，也避免过度包装。

### B3. UIO / SPI / UART / I2C 覆盖面

目标：

- 作为 Linux BSP 覆盖面，能讲清接口路径、访问方式和风险。

你需要输出：

- `02-projects/linux-uio-note.md`
- 可选：`02-projects/linux-spi-uart-i2c-note.md`

完成证明：

- UIO 能讲清：
  - 为什么用
  - 暴露什么资源
  - 用户态如何 mmap
  - 并发、权限、越界、寄存器误写风险
- SPI/UART/I2C 能讲清：
  - 使用标准内核接口、现有封装还是 UIO
  - 典型调试路径
  - 常见异常现象

学习方式：

- 不把 SPI/UART/I2C 做成主卖点。
- 目标是回答常见外设驱动问题，不被基础追问卡住。

### B4. Linux 侧问题定位案例

目标：

- 至少准备 1-2 个 Linux 相关问题定位案例。

你需要输出：

- `02-projects/linux-issue-record-01.md`
- 可选：`02-projects/linux-issue-record-02.md`

完成证明：

案例方向可以是：

- PCIe 链路异常
- BAR/MMIO 访问异常
- MSI/中断异常
- 用户态状态上报异常
- UIO 寄存器访问异常
- SPI/UART 通信异常

学习方式：

- 优先使用真实工作经历。
- 如果真实案例暂时不能写细，可用私下实验复现形成补强案例，但要标注来源。

## C 线：MCU + Linux 共性底层能力

### C1. 共性能力对照表

目标：

- 避免 MCU 和 Linux 重复学习。
- 把 MCU 经验转成 Linux 面试优势。

你需要输出：

- `02-projects/mcu-linux-common-ground.md`

完成证明：

至少覆盖：

| 共性能力 | MCU 侧 | Linux 侧 |
| --- | --- | --- |
| MMIO | 直接寄存器访问 | resource/ioremap/readl/writel |
| 中断 | ISR/清标志/flag | request_irq/bottom half/workqueue |
| 并发 | 关中断/临界区 | spinlock/mutex/atomic/waitqueue |
| 异常 | trap dump/寄存器现场 | oops/panic/dmesg/栈回溯 |
| 总线外设 | SPI/UART/I2C/PCIe 协议 | Linux 子系统/设备树/用户态接口 |
| 启动 | Reset_Handler/链接脚本 | bootloader/kernel/module/probe |

学习方式：

- 每个主题只写“共性 + Linux 差异化”，不写两份重复笔记。

### C2. Linux 底层机制复习

目标：

- 覆盖 Linux 面试高频底层问题，但不深挖无关源码。

需要掌握：

- Linux 启动大链路
- module 加载与 probe
- 内核态/用户态边界
- ioctl/mmap/read/write/sysfs/proc
- 中断上下文和进程上下文
- 同步原语
- 内存分配和 DMA 基础

完成证明：

- 子线程逐项问答验收。
- 主线程只接收验收结论。

## 1 个月节奏

| 周期 | A 线 MCU/SoC | B 线 Linux | C 线共性 |
| --- | --- | --- | --- |
| 第 1 周 | SmartFusion2 架构、V3LoadX 模块边界 | PCIe 完整链路、`endpoint.c` 梳理 | MMIO 共性 |
| 第 2 周 | Flash/IAP/CRC/DirectC 可靠性链路 | ioctl/mmap、BAR/MMIO、用户态控制层 | 用户态/内核态边界 |
| 第 3 周 | 中断、主循环、mailbox、状态机 | MSI/中断、同步、UIO | 中断与并发对比 |
| 第 4 周 | 主备/TMR/MRAM、MCU 问题案例 | SPI/UART/I2C、DMA、Linux 问题案例 | 模拟面试整合 |

## 你每周需要给主线程的东西

每周只给主线程三类内容：

```text
1. 本周完成的文档路径
2. 子线程验收结论
3. 仍然卡住的问题
```

格式：

```text
本周完成：
- xxx.md
- yyy.md

子线程验收：
- PCIe 链路：通过/未通过，问题是 xxx
- 中断同步：通过/未通过，问题是 xxx

卡点：
- xxx
```

## 最终通过标准

1 个月后，如果下面都能勾上，就可以进入简历重构和岗位检索阶段：

- [ ] A 线能讲 3 分钟 SmartFusion2/V3LoadX 项目。
- [ ] A 线有 2 个问题定位案例。
- [ ] B 线能讲 3 分钟 Linux PCIe 项目。
- [ ] B 线有 1-2 个问题定位案例。
- [ ] C 线能讲清 MCU 与 Linux 在 MMIO/中断/异常/调试上的共性和差异。
- [ ] 简历能形成 MCU 与 Linux 两条核心项目线。
- [ ] 子线程完成 Linux 高频点验收。
- [ ] 子线程完成 MCU 项目深挖验收。


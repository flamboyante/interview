# MCU/SoC 项目深挖与面试化计划

## 定位

本计划对应 A 线：

```text
MCU / SoC 裸机固件项目
```

这不是从零学习 MCU 基础，而是把已有 SmartFusion2 / V3LoadX / MiV-RV32 项目打穿，形成可写简历、可讲项目、可应对追问的证据链。

当前核心项目：

```text
主工程：V3LoadX / SmartFusion2 Cortex-M3 MSS
子模块：MiV-RV32 RISC-V 软核辅助控制工程
协同链路：MSS <-> fabric <-> RV32 mailbox/寄存器协同
```

## 总体目标

1 个月内达到：

- 能用 3 分钟讲清 SmartFusion2 项目架构。
- 能说明 V3LoadX 是主工程，MiV-RV32 是子模块/辅助亮点。
- 能讲清启动、链接、中断、主循环、状态机等裸机底层机制。
- 能讲清多链路通信、远程升级、Flash/IAP/CRC、DirectC/JTAG、主备/TMR/MRAM 等核心模块。
- 能准备至少 2 个真实问题定位案例。
- 能明确哪些是自研、哪些是厂商库、哪些是移植适配、哪些是历史保留。

## 第一阶段：SmartFusion2 架构打底

### 目标

讲清整个系统由哪些部分组成，以及每部分负责什么。

必须覆盖：

- SmartFusion2 MSS Cortex-M3
- FPGA fabric / PL
- MiV-RV32 软核
- PL 寄存器 / mailbox
- Flash / MRAM
- CAN / RS422 / Ethernet / UART
- SPI / GPIO / Timer / System Services
- DirectC/JTAG

### 你需要输出

- `02-projects/smartfusion2-architecture.md`

### 完成证明

- 有一张系统架构图或文字版架构图。
- 能讲清 MSS、fabric、RV32 的关系。
- 能说明两份 ELF 的职责边界。
- 能说明 mailbox/寄存器协同的大方向。

### 学习方式

- 先基于已有临时文档整理。
- 不急着读代码实现。
- 优先建立“系统全景图”。

## 第二阶段：V3LoadX 主工程边界

### 目标

明确 V3LoadX 中哪些模块正式使用，哪些是历史保留、预留或不确定。

必须覆盖：

- CAN
- RS422/UART
- Ethernet
- Flash 分区/镜像管理
- IAP
- CRC
- DirectC/JTAG
- AD/电压电流监控
- PLL/时钟芯片控制
- PL 寄存器协同
- 主备 BMU
- TMR/三模
- MRAM
- 自检/产测/shell

### 你需要输出

- `02-projects/v3loadx-module-boundary.md`

### 完成证明

每个模块至少标注：

```text
状态：正式使用 / 历史保留 / 预留 / 不确定
你的角色：主写 / 改造维护 / 移植适配 / 了解
外部依赖：厂商库 / 前人代码 / PL 逻辑 / 硬件设计
能否写简历：可以 / 谨慎 / 不建议
```

### 学习方式

- 先列清单，不追求一次写完整。
- 不确定就标注不确定，后续再查。
- 重点避免简历写到未启用或不可证明的模块。

## 第三阶段：裸机底层机制

### 目标

把 MCU/SoC 底层基础和项目绑定，不停留在概念层。

必须能讲清：

- 上电/复位后程序如何运行。
- 启动文件、向量表、`Reset_Handler` 的作用。
- `.text/.data/.bss/stack/heap` 的初始化和放置。
- 链接脚本为什么重要。
- 中断向量、ISR、清中断、共享变量保护。
- 裸机超级循环和任务状态机。
- 为什么没有 RTOS，以及无 RTOS 的调度取舍。

### 你需要输出

- 可合并进 `smartfusion2-architecture.md`
- 或单独输出：`02-projects/smartfusion2-boot-interrupt-note.md`

### 完成证明

- 能画出启动到主循环的链路。
- 能说明 V3LoadX 中定时器、中断、主循环如何协作。
- 能说明 ISR 和主循环之间如何传递状态。

### 学习方式

- 以项目路径为线索复习，不做泛泛 MCU 基础题。
- 子线程可专项检查启动/链接脚本/中断。

## 第四阶段：多链路通信与业务数据流

### 目标

讲清 V3LoadX 为什么不是普通外设 demo，而是多链路通信控制系统。

必须覆盖：

- CAN 遥测遥控
- RS422/UART 帧处理
- Ethernet / IPv6 / UDP 适配
- PLP0/CPU 透传
- 多链路进入统一业务命令分发
- 帧解析、分包、拼包、校验、异常统计

### 你需要输出

- `02-projects/v3loadx-communication-flow.md`

### 完成证明

- 能选一条典型命令，从输入链路讲到本地处理或透传。
- 能说明多链路如何统一进入业务处理。
- 能说明低带宽 CAN 和较高带宽 422/ETH 的差异和处理风险。

### 学习方式

- 不需要覆盖所有命令。
- 选 1-2 条最典型的数据流讲透。

## 第五阶段：远程维护与可靠性链路

### 目标

把 Flash/IAP/CRC/DirectC/JTAG/主备/TMR/MRAM 整理成可靠性故事。

必须覆盖：

- Flash 分区和镜像类型
- update/golden/iap 等恢复语义
- BMU 自身 IAP
- 其他模块镜像透传
- CRC 软件/硬件分工
- DirectC/JTAG 互助重构
- 主备 BMU 切换
- TMR/三模
- MRAM 状态持久化

### 你需要输出

- `02-projects/v3loadx-reliability-flow.md`

### 完成证明

- 能画出一次远程升级或重构流程。
- 能说明失败时如何发现、上报或恢复。
- 能明确哪些是厂商组件，哪些是你的适配和业务集成。

### 学习方式

- 先从“流程和风险”入手。
- 不从函数细节入手。
- 面试重点是可靠性设计，而不是背 API。

## 第六阶段：MiV-RV32 子模块与 mailbox 协同

### 目标

把 MiV-RV32 讲成 SmartFusion2 系统里的辅助控制亮点，而不是让它抢主工程权重。

必须覆盖：

- RISC-V Machine Mode 裸机运行。
- system tick / machine timer。
- trap dump。
- UART/SPI/DAC8550。
- PL 寄存器握手。
- MSS 先启动，RV32 后启动或由 MSS 控制复位释放。
- MSS 通过心跳计数感知 RV32。
- MSS 可拉 `init down` 给 RV32 作为 reset 信号。

### 你需要输出

- 可并入 `smartfusion2-architecture.md`
- 或单独输出：`02-projects/miv-rv32-mailbox-note.md`

### 完成证明

- 能说明 RV32 在系统中的位置。
- 能说明 MSS 下发哪些类型命令。
- 能说明 RV32 回传哪些状态和维测信息。
- 能说明心跳和 reset 控制的意义。

### 学习方式

- 不要把它包装成独立大项目。
- 作为“RISC-V 软核 + 异构协同”亮点使用。

## 第七阶段：问题定位案例

### 目标

至少准备 2 个 MCU/SoC 侧真实问题定位案例。

你需要输出：

- `02-projects/smartfusion2-issue-record-01.md`
- `02-projects/smartfusion2-issue-record-02.md`

每个案例格式：

```text
问题现象：
影响范围：
初始假设：
定位手段：
关键证据：
修复方案：
复盘总结：
```

可选方向：

- 通信链路异常
- Flash/IAP/CRC 异常
- DirectC/JTAG 重构异常
- 主备切换异常
- PL 寄存器协同异常
- RV32 心跳/reset 异常
- 电压电流监控异常
- 串口/以太网/CAN 数据流异常

## 一个月节奏

| 周期 | 重点 | 输出 |
| --- | --- | --- |
| 第 1 周 | SmartFusion2 架构、V3LoadX 模块边界 | `smartfusion2-architecture.md`、`v3loadx-module-boundary.md` |
| 第 2 周 | 启动/链接/中断、多链路通信 | `smartfusion2-boot-interrupt-note.md`、`v3loadx-communication-flow.md` |
| 第 3 周 | 远程维护、可靠性、MiV-RV32 协同 | `v3loadx-reliability-flow.md`、`miv-rv32-mailbox-note.md` |
| 第 4 周 | 问题定位案例、模拟面试整合 | `smartfusion2-issue-record-01.md`、`smartfusion2-issue-record-02.md` |

## 最终验收标准

- [ ] 能 3 分钟讲清 SmartFusion2 总体架构。
- [ ] 能讲清 V3LoadX 主工程职责和你的 ownership。
- [ ] 能说明 MiV-RV32 是子模块/辅助亮点。
- [ ] 能讲清启动、链接、中断、主循环。
- [ ] 能讲清多链路通信统一分发。
- [ ] 能讲清 Flash/IAP/CRC/DirectC/JTAG 可靠性链路。
- [ ] 能讲清主备/TMR/MRAM 的高可靠设计。
- [ ] 能准备 2 个真实问题定位案例。
- [ ] 能明确厂商库/自研/适配/历史保留边界。


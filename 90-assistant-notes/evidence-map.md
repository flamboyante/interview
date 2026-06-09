# 证据地图

本文件用于记录“哪些能力有证据支撑，哪些只是了解，哪些需要补强”。它是助手工作笔记，不是直接投递材料。

## 证据等级

```text
A：主负责 / 从 0 到 1 / 可被项目和代码支撑
B：参与实现 / 有调试经验 / 能讲清楚关键路径
C：了解原理 / 看过或改过局部 / 面试需谨慎表达
D：待补强 / 暂无足够证据支撑简历表达
```

## 当前能力证据

| 能力方向 | 当前等级 | 已知依据 | 风险 |
| --- | --- | --- | --- |
| SmartFusion2 Cortex-M3 高可靠控制固件 | A | `V3LoadX` 是 SmartFusion2 主工程，覆盖多链路通信、升级、Flash、DirectC/JTAG、主备、TMR、PL 协同，用户为主负责开发人 | 需避免把厂商 DirectC/HAL 描述为自研 |
| MiV-RV32 RISC-V 软核裸机框架 | B+ | RISC-V soft processor，是 SmartFusion2 系统中的子模块/辅助控制工程；包含 systick、trap dump、UART/SPI/DAC、PL 寄存器握手、分层 BSP/HAL | 当前业务入口偏 DAC，需突出框架和协同价值，但不能让它盖过 V3LoadX 主工程 |
| 双 ELF / 异构协同 | A/B | MSS 与 RV32 两个 ELF，通过 `MSS <-> fabric <-> RV32` mailbox/寄存器协同；MSS 下发维测/业务/功能命令，RV32 回传状态/结果/错误/维测信息；MSS 先启动并通过心跳计数感知 RV32，可拉 `init down` 复位 RV32 | 需补充典型命令例子、心跳超时阈值和恢复策略 |
| 启动/链接脚本/中断 | A/B | 用户明确负责启动代码、链接脚本、中断；RISC-V 文档提到 machine timer/trap；M3 文档涉及 CMSIS 启动与裸机主循环 | 需确认能否讲清启动流程和内存布局 |
| HAL/外设驱动适配 | B | 基于厂商 demo 移植、适配、修改 | 不能包装成完全自研 |
| 多链路通信/协议网关 | A | V3LoadX 覆盖 CAN、RS422、Ethernet、UART，统一遥测遥控处理和透传 | 需补充实际最强链路和具体问题记录 |
| 在线升级/镜像管理/可靠性 | A | Flash 分区、BMU IAP、CRC、MRAM、主备、TMR、DirectC/JTAG | 需明确哪些功能已量产启用，哪些是预留/备用 |
| Linux PCIe 驱动与用户态控制层 | B+ | 用户与同事共同编写维护 PCIe endpoint 驱动，参考 `/home/flamboy/linux-learning/labs/week-pcie/endpoint.c`；该驱动基于前人基础持续改造维护，用户表示整个文件都能理解；同时做用户态初始化、整合、状态管理、上报、告警、分发和维护 | 不能包装成从 0 独立完成完整 PCIe 栈；需补充典型问题定位和用户态链路边界 |
| Linux SPI/UART/UIO 外设 | B/C | SPI/UART 在现有架构上基于内核驱动接口开发维护；部分功能通过自定义 UIO 直接访问寄存器 | 不宜作为主证据；可作为 BSP 覆盖面表达；UIO 需说明资源暴露和访问边界 |
| Linux 驱动个人学习补强 | B | `/home/flamboy/linux-learning` 有字符设备、MMIO、同步、事件、中断、设备树、PCIe endpoint 等实验材料 | 作为学习证据有价值，但不能替代真实工作项目；需整理成学习路线和实验总结 |
| QEMU 建模验证 | B/C | 简历已有 QEMU 外设建模经历 | 需补充可运行测试或验证闭环 |
| AI SoC 方向 | D/B | 目标方向明确，但直接 AI SoC 项目证据不足 | 应表达为 SoC 底层系统软件，不包装成 AI 算法/Runtime |

## 当前推荐表达

较稳表达：

```text
SoC 底层系统软件工程师，具备 ARM/RISC-V 裸机固件、BSP/外设驱动、Linux PCIe 驱动和 QEMU 验证经验。
```

暂不建议表达：

```text
AI 算法工程师
AI Runtime 工程师
纯 Linux 内核专家
纯 QEMU 虚拟化专家
```

## 待确认问题

1. MSS 与 MiV-RV32 mailbox/寄存器协议的典型命令例子：维测类、业务类、功能类各 1-2 个。
2. 心跳计数的超时阈值、`init down` reset 触发条件和恢复策略。
3. V3LoadX 中哪些模块是正式使用，哪些是历史保留或备用方案。
4. Linux PCIe 模块典型问题定位：链路、BAR/MMIO、中断/MSI、用户态状态上报中至少补 1-2 个案例。
5. SPI/UART 是通过标准用户态接口、内核驱动改动，还是 UIO/寄存器访问实现。
6. QEMU 项目是否有可运行测试或回归结果。
7. 下一份工作城市、薪资、行业和强度约束。

# 面试项目状态快照

## 项目目标

长期目标：

```text
通过面试，找到更高薪资、更长职业寿命、更有成长性的嵌入式/SoC/Linux 驱动相关工作。
```

当前阶段目标：

```text
先不急着投递，先用 1 个月左右把项目证据、知识体系、简历表达和面试口径整理到可面试状态。
```

## 主线程工作规则

本线程是面试项目总控台，只负责：

- 总体规划
- 阶段复盘
- 简历和项目材料整理
- 证据链判断
- 后续岗位检索和 JD 匹配

本线程不负责：

- 具体代码审查
- 单个面试题训练
- 大段工程实现细节
- Linux/MCU 某个知识点的深度教学

具体学习和技术验收放到子线程；子线程结论再同步回本线程。

## 当前三条准备线

```text
A 线：MCU / SoC 裸机固件项目
B 线：Linux BSP / Driver 项目
C 线：MCU + Linux 共性底层能力
```

A 线和 B 线同等重要，都是核心项目经历。C 线用于打通底层能力，避免 MCU 和 Linux 重复学习。

## 当前岗位定位假设

暂定方向：

```text
SoC 底层系统软件工程师
Linux BSP / Driver 工程师
嵌入式 Linux / PCIe / 外设驱动工程师
MCU/SoC 固件与板级 bring-up 工程师
```

不建议当前包装为：

```text
AI 算法工程师
AI Runtime 工程师
纯 Linux 内核专家
纯 QEMU 虚拟化专家
```

目标是靠 MCU/SoC 固件 + Linux BSP/Driver 两条项目线，去匹配 AI SoC、高性能 SoC、通信设备、板级系统软件相关岗位。

## A 线：MCU / SoC 裸机固件

核心项目：

```text
SmartFusion2 / V3LoadX / MiV-RV32
```

当前判断：

- `V3LoadX` 是 SmartFusion2 项目的主工程。
- `MiV-RV32` 是 SmartFusion2 系统里的子模块/辅助控制工程。
- 用户是 SmartFusion2 相关固件的主负责开发人。
- 项目不是普通 MCU demo，而是高可靠通信控制与在线维护系统。

关键能力：

- Cortex-M3 MSS 裸机固件
- 多链路通信：CAN、RS422、Ethernet、UART
- Flash 分区、镜像管理、IAP、CRC
- DirectC/JTAG 互助重构
- 主备 BMU、TMR、MRAM、可靠性设计
- PL 寄存器协同
- MiV-RV32 RISC-V 软核控制框架
- MSS <-> fabric <-> RV32 mailbox/寄存器协同

当前边界：

- DirectC 算法库是厂商组件，不能描述为自研。
- HAL/厂商驱动不是完全从 0 写，应描述为移植、适配、封装和业务集成。
- MiV-RV32 不能盖过 V3LoadX 主工程权重。

## B 线：Linux BSP / Driver

核心项目：

```text
PCIe endpoint 内核驱动 + 用户态控制链路
```

当前判断：

- Linux 线不是辅助线，而是核心项目经历之一。
- 用户与同事共同维护 PCIe endpoint 内核驱动。
- `endpoint.c` 来自内部工程搬运整理，是可公开参考的雏形。
- 该驱动不是用户从 0 独立新写，而是在前人基础上长期改造维护，用户表示整个文件都能讲清。
- 用户态部分包括初始化、整合、状态管理、处理、上报、告警、分发和维护。
- SPI/UART/I2C/UIO 是 Linux BSP/外设覆盖面，不作为最强主证据。

关键能力：

- PCIe driver
- BAR/MMIO
- MSI/中断
- ioctl/mmap
- 用户态控制层
- 状态管理与告警上报
- SPI/UART/I2C 接口维护
- UIO 寄存器访问

当前边界：

- 不能写成“从 0 独立完成完整 Linux PCIe 驱动栈”。
- 可以写成“参与并长期维护 PCIe endpoint 驱动和用户态控制链路，能够讲清完整链路”。
- 个人 `linux-learning` 学习材料可作为能力补强证据，但不能替代工作项目。

## C 线：MCU + Linux 共性底层能力

用途：

```text
把 MCU 经验转成 Linux 面试优势，避免两套知识重复学习。
```

重点共性：

| 能力 | MCU 侧 | Linux 侧 |
| --- | --- | --- |
| MMIO | 直接寄存器访问 | resource/ioremap/readl/writel |
| 中断 | ISR、清标志、flag | request_irq、bottom half、workqueue |
| 并发 | 关中断、临界区 | spinlock、mutex、atomic、waitqueue |
| 异常 | trap dump、寄存器现场 | oops、panic、dmesg、栈回溯 |
| 启动 | Reset_Handler、链接脚本 | bootloader、kernel、module/probe |
| 调试 | 串口、JTAG、示波器 | dmesg、lspci、ftrace、strace |

## 已有关键文档

总控文档：

- `00-control/strategy.md`
- `00-control/linux-study-plan.md`
- `00-control/mcu-soc-study-plan.md`
- `00-control/one-month-action-plan.md`
- `00-control/project-state.md`

项目文档：

- `02-projects/smartfusion2-overview.md`
- `02-projects/linux-bsp-driver-overview.md`

档案与证据：

- `01-profile/career-profile.md`
- `90-assistant-notes/evidence-map.md`

原始/临时材料：

- `临时文档存放/RISC-V_MCU项目面试亮点整理.md`
- `临时文档存放/V3LoadX项目架构与技术含金量融合版.md`
- `T3562简历.md`

外部学习材料：

- `/home/flamboy/linux-learning/docs`
- `/home/flamboy/linux-learning/labs/week-pcie/endpoint.c`

## 当前 1 个月需要完成的输出

优先级最高：

1. `02-projects/smartfusion2-architecture.md`
2. `02-projects/linux-pcie-flow.md`
3. `02-projects/v3loadx-module-boundary.md`
4. `02-projects/linux-pcie-ownership.md`
5. `02-projects/mcu-linux-common-ground.md`

随后补充：

6. `02-projects/v3loadx-reliability-flow.md`
7. `02-projects/linux-uio-note.md`
8. `02-projects/smartfusion2-issue-record-01.md`
9. `02-projects/smartfusion2-issue-record-02.md`
10. `02-projects/linux-issue-record-01.md`

## 完成证明

每个输出不以“写完文档”为终点，而以以下标准验收：

```text
1. 能用 3 分钟讲清。
2. 子线程追问 5 个问题能答住。
3. 能说明项目边界和风险。
4. 能反哺简历 bullet。
```

## 后续恢复上下文方法

如果新开 session 或上下文丢失，优先阅读：

1. `00-control/project-state.md`
2. `00-control/one-month-action-plan.md`
3. `90-assistant-notes/evidence-map.md`
4. `02-projects/smartfusion2-overview.md`
5. `02-projects/linux-bsp-driver-overview.md`

读完后继续按 A/B/C 三线推进。

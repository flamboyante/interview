# SmartFusion2 双核固件项目总览

## 定位

这是当前最核心的项目证据。项目基于 Microchip/Microsemi SmartFusion2，包含 Cortex-M3 MSS 与 MiV-RV32 RISC-V 软核两个固件工程。两个工程分别生成 ELF，承担不同功能模块，并通过 fabric/mailbox/寄存器方式协同。

当前总控层面的推荐表达：

```text
基于 SmartFusion2 MSS + MiV-RV32 软核平台，主负责双 ELF 裸机固件开发，覆盖 Cortex-M3 高可靠通信控制、RISC-V 软核外设控制、PL 寄存器协同、远程维护、异常上报和板级调试闭环。
```

## 工程一：V3LoadX / Cortex-M3 MSS

### 项目形态

`V3LoadX` 是运行在 SmartFusion2 Cortex-M3 上的 BMU 管理控制固件。它不是单一外设 demo，而是板级管理控制系统。

### 主要功能域

- 多链路通信接入：CAN、RS422、Ethernet、UART。
- 遥测遥控处理：多链路命令解析、分发、本地处理和透传。
- 远程升级：文件传输、Flash 镜像管理、BMU IAP。
- JTAG 互助重构：集成适配 DirectC，通过远程命令触发在线编程/重构。
- 健康监测：AD7490 电压采样、电流/过流状态、PLL/时钟配置。
- 可靠性控制：主备 BMU、MRAM 状态、TMR/三模、CRC 校验、Flash 轮转。
- PL 协同：通过 fabric 地址空间和 PL 寄存器完成控制、状态读取和硬件任务触发。

### 工程特点

- 裸机超级循环，无 RTOS。
- 主循环通过状态机、任务轮询、外设回调、Timer 中断协作。
- M3 侧承担协议、状态机、调度和控制决策。
- FPGA fabric/PL 侧承担 CoreIP 外设、共享寄存器、硬件状态、CRC/TMR/采样/主备相关逻辑。

### 面试价值

该工程支撑以下能力表达：

- 高可靠裸机固件开发
- 多链路协议网关
- 远程升级和镜像管理
- MCU + FPGA fabric 协同
- 主备/TMR/CRC/MRAM 可靠性设计
- 板级 bring-up、现场调试和异常诊断

### 表达边界

- DirectC 算法库是厂商组件，不能描述为自研。
- HAL/厂商驱动不是完全从 0 写，应描述为移植、适配、封装和业务集成。
- INA3221/PMBus、CDCM6208 等是否实际启用需后续确认，避免在简历中强写。

## 工程二：MiV-RV32 裸机控制框架

### 项目形态

这是运行在 Microchip Mi-V RV32 RISC-V 软核上的裸机 MCU 控制工程。当前以 DAC8550 输出测试为业务入口，但核心价值是 RISC-V 软核裸机控制框架。

### 主要能力

- RISC-V Machine Mode 裸机运行。
- 基于 machine timer 的 1ms systick。
- 中断 flag 和主循环事件消费。
- UART 日志与命令行调试。
- SPI DAC8550 外设驱动。
- CPU 与 PL 侧寄存器握手协议。
- trap dump 异常诊断，包括 CSR、寄存器和栈现场。
- `app / bsp / drivers / platform / hal / miv_rv32_hal` 分层。

### 面试价值

该工程支撑以下能力表达：

- RISC-V 软核裸机开发
- Machine timer、trap、中断、CSR 等底层机制
- CPU 与 PL 侧寄存器握手
- 外设驱动框架和板级 BSP 分层
- 裸机异常诊断和 bring-up 调试闭环

### 表达边界

- 不建议把项目说成“只做 SPI DAC”。
- 更稳的说法是“以 DAC8550 作为外设样例验证 RISC-V 软核裸机控制框架”。

## 双工程协同

当前已知协同方式：

```text
MSS <-> fabric <-> MiV-RV32
```

主要通过 mailbox/寄存器方式完成。MSS 侧先启动，RV32 侧后启动或由 MSS 控制复位释放；MSS 可以通过心跳计数感知 RV32 运行状态，并可通过拉 `init down` 给 RV32 作为 reset 信号。

当前已知协同内容：

- MSS 会通过 mailbox/寄存器向 RV32 下发多类命令，包括维测类、业务类和功能类命令。
- RV32 会回传执行状态、结果、错误信息和较多维测信息。
- 协同链路不是简单单向触发，而是具备命令下发、状态反馈、心跳感知和复位控制的闭环。

这一点适合在简历和面试中表达为：

```text
设计并维护 MSS 与 MiV-RV32 之间的 mailbox/寄存器协同机制，支持命令下发、结果回传、维测信息上报、心跳监测和 RV32 复位控制。
```

待补充：

- mailbox 寄存器地址或逻辑名称。
- 典型命令分类示例：维测类、业务类、功能类各 1-2 个即可。
- 心跳计数的超时阈值、异常判定和恢复策略。
- RV32 reset/init down 的触发条件和恢复流程。
- 协同失败时的错误码、超时和降级策略。

## 当前最需要补的材料

1. 一张系统架构图：MSS、fabric、MiV-RV32、外设、Flash/MRAM、通信链路。
2. 一份 mailbox 协同说明：命令、状态、结果、异常。
3. 两个真实问题定位记录：现象、定位路径、证据、修复。
4. 一份“哪些是从 0 写，哪些是厂商 demo/库适配”的边界说明。

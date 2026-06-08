# RISC-V MCU 项目面试亮点整理

## 1. 项目定位

本项目是一个基于 Microchip Mi-V RV32 RISC-V 软核的裸机 MCU 控制工程，运行在 FPGA SoC 场景下。项目围绕 RISC-V 裸机启动、中断系统、板级外设驱动、CPU 与 PL 侧寄存器通信、串口调试和事件主循环构建了一套可扩展的软件框架。

项目当前已经接入 UART、SPI、DAC8550、PL 寄存器协议、系统 tick、异常 trap dump 等能力。虽然现阶段以 DAC 输出测试为主要业务入口，但整体结构并不局限于 DAC，后续可以继续挂载 ADC、GPIO、PWM、I2C 传感器、SPI Flash、定时采样模块、上位机通信协议等外设。

一句话概括：

> 基于 RISC-V 软核构建裸机 MCU 运行框架，通过分层 BSP/HAL/platform/app 结构，实现外设驱动、软硬件寄存器握手、系统 tick 调度和串口调试闭环。

## 2. 技术栈与工程环境

- 处理器：Microchip Mi-V RV32 RISC-V soft processor
- 开发环境：Microchip SoftConsole
- 运行模式：裸机程序，Machine Mode
- 外设接口：UART、SPI、CoreGPIO/MMIO、PL 侧寄存器
- 当前外设：DAC8550 SPI DAC
- 系统节拍：基于 RISC-V machine timer 的 1 ms systick
- 调试方式：UART 日志、命令行交互、trap dump、异常现场输出
- 工程结构：app / bsp / drivers / platform / hal / miv_rv32_hal 分层

## 3. 架构亮点

### 3.1 分层清晰，方便移植外设

项目不是把业务逻辑、寄存器访问和外设驱动混在一起，而是拆成几层：

```text
Application
    业务逻辑、命令处理、周期任务

BSP
    板级资源封装，如 UART、DAC、PL 寄存器访问

Drivers
    具体 IP 或芯片驱动，如 CoreSPI、CoreUARTapb、DAC8550

Platform
    系统 tick、中断 flag、delay、panic、平台级事件

HAL / Mi-V HAL
    RISC-V CSR、machine timer、trap、中断底层封装
```

这种结构的优点是：新增外设时，只需要在 driver 层实现设备驱动，在 BSP 层做板级绑定，在 app 层增加命令或业务流程，不需要反复改动 RISC-V 中断、启动和平台公共逻辑。

面试可以强调：

> 我把工程拆成 app、BSP、driver、platform、HAL 几层。这样当前虽然接的是 DAC，但后面接 ADC、PWM、I2C 传感器或者其他 SPI 外设时，主要扩展 driver 和 BSP，不会破坏底层 tick、中断和主循环框架。

### 3.2 裸机事件主循环框架

项目采用典型裸机 event loop 模式：

- ISR 只记录事件或置位 flag
- 主循环轮询并消费事件
- 周期任务由 systick 驱动
- 外部命令由 PL 寄存器或 UART 命令触发

这种模式适合小型 MCU 或 FPGA 软核控制场景，逻辑简单、时序可控、调试方便，也为后续迁移到 RTOS 保留了清晰边界。

可以这样讲：

> 这个项目没有直接上 RTOS，而是先实现了一个轻量裸机事件框架。中断只做最小动作，复杂业务放到主循环里处理，避免 ISR 过长影响实时性，也减少共享资源竞争。

### 3.3 RISC-V machine timer systick

项目没有依赖操作系统，而是基于 RISC-V machine timer 实现系统节拍。核心链路包括：

```text
MTIME 递增
MTIMECMP 比较匹配
触发 machine timer interrupt
CPU 进入 mtvec trap 入口
底层保存上下文
分发到 timer handler
调用应用层 SysTick_Handler
platform 层更新 tick 和 flag
主循环消费 tick 事件
```

这个点的面试含金量较高，因为它能体现对 RISC-V 底层机制的理解，而不只是会调用外设库。

可展开关键词：

- `mstatus.MIE`：全局中断开关
- `mie.MTIE`：machine timer interrupt 使能
- `mip.MTIP`：timer pending 状态
- `mtvec`：trap 入口
- `mcause`：trap 原因
- `MTIME/MTIMECMP`：RISC-V machine timer 基础

### 3.4 CPU 与 PL 侧寄存器握手协议

项目通过 memory-mapped register 方式实现 CPU 与 FPGA PL 侧通信。协议中包含：

- heartbeat：CPU 周期性向 PL 报告自身存活状态
- command：PL 向 CPU 下发命令
- result：CPU 向 PL 返回状态、命令号和错误码

状态设计包括：

- READY：CPU 空闲，等待命令
- BUSY：CPU 已接收命令并正在处理
- DONE：命令执行成功
- ERROR：命令执行失败

这种握手协议比简单读写寄存器更接近真实项目，因为它解决了命令重复触发、执行状态可见、错误码反馈和软硬件协同调试的问题。

可以这样讲：

> 我没有只做单向寄存器读写，而是设计了 command/result/heartbeat 三类寄存器，让 PL 可以下发命令，CPU 执行后回写状态和错误码。这样软硬件两侧都有明确的状态机，方便联调和问题定位。

### 3.5 外设驱动可替换

当前示例外设是 DAC8550，主要验证了：

- SPI master 初始化
- SPI slave select 控制
- 24-bit DAC frame 发送
- 原始码值与电压输出的映射
- high-z / power-down 模式控制
- UART 和 PL 两种方式触发 DAC 测试

但从架构上看，DAC 只是一个设备实例。以后可以按照同样方式扩展：

```text
drivers/CoreI2C_xxx
drivers/CoreSPI_xxx
drivers/CoreADC_xxx
drivers/CorePWM_xxx
bsp/board_adc.c
bsp/board_sensor.c
bsp/board_pwm.c
app command dispatch
PL command protocol extension
```

简历和面试中可以弱化“只做 DAC”，强调“以 DAC 为样例验证可移植外设框架”。

### 3.6 UART 调试命令行

项目提供 UART 日志和简单命令行，可以通过串口触发状态查询、DAC 测试、high-z 控制、panic 测试等操作。

这体现了嵌入式工程里的一个重要能力：不是只把功能写出来，还要能在板子上验证、观测和定位问题。

可以这样讲：

> 我做了 UART 命令入口，方便在没有复杂上位机的情况下直接通过串口触发外设测试、查看 tick 状态和验证异常处理。这样板级 bring-up 时可以快速闭环。

### 3.7 trap dump 异常诊断能力

裸机环境下没有操作系统自动帮忙保存错误现场。项目中增强了 trap 处理逻辑，异常发生时可以输出：

- trap 类型：interrupt 或 exception
- `mcause`
- `mepc`
- `mtval`
- `mtvec`
- `mstatus`
- `mie`
- `mip`
- 通用寄存器现场
- 栈范围和栈内容

这对于定位 illegal instruction、load/store fault、栈溢出、空指针访问、非法地址访问等问题非常有价值。

面试中这是非常好的加分项：

> 裸机调试最怕异常后只停在死循环里看不到上下文，所以我在 trap handler 里加入了 CSR、寄存器和栈 dump。这样即使没有完整操作系统，也能根据 `mepc/mcause/mtval` 快速判断异常指令、异常原因和访问地址。

## 4. 项目含金量评估

### 4.1 比普通 MCU demo 更强的地方

普通 MCU demo 往往只体现外设 API 调用，比如点灯、串口打印、SPI 发送数据。本项目更有价值的地方在于：

- 运行在 RISC-V FPGA 软核环境，而不是固定芯片 SDK demo
- 涉及启动、链接脚本、trap、中断、CSR 等底层知识
- 有 CPU 与 PL 侧通信协议，属于软硬件协同场景
- 有 BSP/driver/platform/app 分层，具备扩展外设的工程结构
- 有 UART 命令行和 trap dump，具备调试闭环
- 有 systick 和事件主循环，具备裸机任务调度雏形

### 4.2 面试含金量等级

如果只说“我用 SPI 控制了 DAC”，含金量中等偏低。

如果按下面方式讲，含金量会明显提高：

> 我基于 Mi-V RV32 RISC-V 软核搭建了一个裸机 MCU 控制框架，完成了 machine timer systick、中断分发、UART 调试、SPI 外设驱动、PL 寄存器握手协议和异常现场 dump。DAC8550 是当前接入的一个外设样例，后续可以按相同 BSP/driver 模式扩展 ADC、PWM、I2C/SPI 传感器或其他板级模块。

这种表述体现的是底层框架能力，而不是单一外设功能。

## 5. 简历写法

### 版本一：简洁版

基于 Microchip Mi-V RV32 RISC-V 软核实现裸机 MCU 控制框架，完成 systick 中断、UART 调试、SPI DAC8550 驱动、CPU-PL 寄存器握手协议和异常 trap dump；采用 app/BSP/driver/platform/HAL 分层设计，支持后续扩展 ADC、PWM、I2C/SPI 传感器等多类外设。

### 版本二：偏底层版

基于 RISC-V Machine Mode 构建裸机运行环境，梳理并实现 `MTIME/MTIMECMP` systick、中断 flag 消费、trap 分发和异常现场 dump；封装 UART、SPI、PL MMIO 访问和 DAC8550 板级驱动，设计 command/result/heartbeat 软硬件握手协议，用于 FPGA PL 与 CPU 协同控制。

### 版本三：偏工程版

设计并实现 Mi-V RV32 裸机外设控制框架，采用 platform/BSP/driver/app 分层，将系统 tick、中断处理、日志调试、PL 寄存器通信和外设驱动解耦；当前完成 DAC8550 SPI 输出控制和串口/PL 双入口测试，框架可复用到更多板级外设。

## 6. 面试 1 分钟自述

这个项目是我在 Microchip Mi-V RV32 RISC-V 软核上做的一个裸机 MCU 控制框架。它不是单纯的外设 demo，而是从底层启动、中断、系统 tick、BSP 驱动、PL 寄存器通信到应用层命令处理都做了分层。

当前业务上接了 UART、SPI 和 DAC8550，同时设计了 CPU 与 FPGA PL 侧的 command/result/heartbeat 握手协议。PL 可以通过寄存器下发命令，CPU 执行后回写状态和错误码；本地也可以通过 UART 命令触发测试，方便板级 bring-up。

底层方面，我重点梳理了 RISC-V machine timer 的 systick 链路，包括 `MTIME/MTIMECMP`、`mie.MTIE`、`mstatus.MIE`、`mtvec` 和 `mcause`。中断里只置 flag，主循环消费事件，保证 ISR 足够短。项目还增强了 trap dump，异常时能输出 CSR、寄存器和栈现场，方便裸机调试。

虽然现在主要验证的是 DAC 输出，但工程结构是可扩展的。后续如果接 ADC、PWM、I2C 传感器或其他 SPI 外设，只需要扩展 driver 和 BSP，再把命令接入 app 层和 PL 协议即可。

## 7. 可重点展开的问题

### 7.1 为什么要分层？

分层是为了降低外设扩展成本。RISC-V 底层中断、系统 tick、trap、平台状态属于公共能力，不应该随着每个外设变化而反复修改。外设差异应该集中在 driver 和 BSP 层。

### 7.2 为什么 ISR 里只置 flag？

ISR 会打断主流程，执行时间越长，对实时性影响越大，也更容易引入共享资源竞争。项目里 ISR 只更新时间和事件标志，复杂任务放在主循环执行，逻辑更可控。

### 7.3 `volatile` 是否能保证安全？

不能。`volatile` 只能保证变量每次都真实读写，不被编译器优化掉。对于“读取后清零”这类多步操作，还需要关中断保护临界区，防止 ISR 和主循环同时访问导致事件丢失。

### 7.4 CPU 和 PL 为什么要设计握手协议？

因为软硬件协同时不能只靠一个寄存器值表达所有状态。命令是否被接收、是否正在执行、是否完成、是否失败，都需要明确状态反馈。握手协议可以降低联调成本，也方便 PL 侧做状态机。

### 7.5 为什么说这个框架可以移植更多外设？

因为外设驱动、板级绑定、应用命令和底层平台能力是分开的。新增外设时，通常只需要：

1. 在 driver 层实现芯片或 IP 驱动
2. 在 BSP 层绑定具体 SPI/I2C/GPIO/地址资源
3. 在 app 层增加命令和业务流程
4. 如需 PL 控制，再扩展 command/result 协议

RISC-V 启动、中断、tick、日志和异常诊断逻辑可以继续复用。

## 8. 后续可以继续增强的方向

- 增加统一 device registry 或 board device table，方便多个外设统一初始化
- 给 PL command 协议增加 sequence id，避免重复命令或旧结果误判
- 增加 command timeout，防止某个外设操作卡住主循环
- 增加外设自检流程，上电后输出 board bring-up report
- 给 SPI/I2C 外设增加 bus lock 或统一事务接口
- 增加 ring buffer 日志，降低 UART 阻塞对实时性的影响
- 增加示波器/逻辑分析仪测试记录，如 SPI 波形、DAC 输出电压、heartbeat 周期
- 整理编码格式，避免中文注释在不同编辑器中显示乱码

## 9. 推荐最终表达

最终面试时建议不要把项目说成“我写了一个 DAC 驱动”，而是说：

> 我做的是一个 RISC-V 软核裸机外设控制框架。DAC8550 是当前接入的验证外设，项目真正的重点是 RISC-V machine timer systick、中断和 trap 处理、BSP/driver/platform 分层、CPU 与 PL 的寄存器握手协议，以及 UART 调试闭环。这个框架后续可以继续扩展 ADC、PWM、I2C/SPI 传感器等外设。

这样讲，项目的技术含金量会从“单个外设驱动”提升到“RISC-V 裸机平台框架 + 软硬件协同控制”。

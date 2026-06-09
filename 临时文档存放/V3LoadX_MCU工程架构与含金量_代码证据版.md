# V3LoadX MCU 工程架构与含金量说明

> 代码证据版。本文面向嵌入式软件工程师视角，基于当前仓库实际代码整理项目架构、外设资源、核心流程、模块含金量和后续可补强方向。  
> 原则：已实现能力和可补强工作分开；厂商组件和自研适配边界分开；不把普通外设初始化包装成系统能力，但也不低估这个工程的软硬件协同价值。

## 1. 项目总体定位

`V3LoadX` 是一个运行在 Microchip/Microsemi SmartFusion2 Cortex-M3 上的 BMU 管理控制固件。它的核心不是单个通信接口，也不是单一主备切换功能，而是一个板级管理控制系统：M3 侧负责多链路协议接入、裸机任务调度、远程升级、Flash 镜像管理、DirectC/JTAG 互助重构、健康监测、主备冗余和 PL 寄存器控制；FPGA fabric/PL 侧提供 CoreIP 外设、共享寄存器、硬件状态、CRC/TMR/采样/保护等协同能力。

更准确的工程定位可以概括为：

- SmartFusion2 Cortex-M3 裸机 BMU 管理控制固件。
- 多链路遥测遥控与远程维护网关。
- MCU + FPGA fabric 协同的板级可靠性控制系统。
- 面向无人值守/远程维护场景的升级、重构、监控和冗余控制平台。

这个工程的含金量主要来自四类能力：

- **链路复杂度**：CAN、RS422、Ethernet、UART 多链路同时进入统一业务命令体系。
- **维护复杂度**：支持 Flash 镜像管理、BMU IAP、其他模块透传升级、CRC 校验、JTAG 互助重构。
- **软硬件协同复杂度**：M3 通过 fabric 地址空间访问 CoreIP 和 PL 寄存器，控制 Flash、CRC、TMR、主备、AD 状态等硬件逻辑。
- **可靠性复杂度**：主备 BMU、MRAM 状态持久化、三模/TMR、电源健康监测、链路恢复、异常统计。

## 2. 硬件平台与外设资源

### 2.1 SmartFusion2 与 Cortex-M3

代码证据：

- `CMSIS/m2sxxx.h`
- `CMSIS/system_m2sxxx.c`
- `CMSIS/startup_gcc/startup_m2sxxx.S`
- `.cproject`
- `drivers_config/sys_config/sys_config_mss_clocks.h`

工程使用 SmartFusion2 系列 CMSIS 和启动文件，`.cproject` 中 Cortex-M3 目标配置为 `-mcpu=cortex-m3`。系统时钟配置显示 M3 主频为 128 MHz，APB0 为 64 MHz，APB1/APB2 为 32 MHz，FIC0 为 64 MHz，FIC1/FIC64 为 128 MHz。

运行模式是裸机超级循环，没有 FreeRTOS/uCOS/RTX 等 RTOS 痕迹。实时性依赖 Timer 中断、任务状态机、主循环轮询和外设回调协作。

### 2.2 MSS 原生外设

代码证据：

- `main.c`
- `drivers/mss_gpio`
- `drivers/mss_can`
- `drivers/mss_spi`
- `drivers/mss_ethernet_mac`
- `drivers/mss_timer`
- `drivers/mss_sys_services`
- `drivers/mss_uart`

MSS 外设在工程中的主要用途：

- **MSS GPIO**：Flash 控制脚、主备握手、对端 BMU 状态输入输出、DirectC/JTAG 相关 IO 控制。
- **MSS CAN**：注册为统一 CAN 抽象层中的 `CAN_0`。
- **MSS SPI**：BMU 本地 SPI Flash、IAP 镜像访问、系统服务编程流程。
- **MSS Ethernet MAC**：以太网链路接入，RX callback 入队，TX buffer 状态管理。
- **MSS Timer1/Timer2**：裸机时间基准、周期任务、延时、任务耗时统计基础。
- **MSS System Services**：读取 design version，执行 IAP authenticate/program/verify。
- **MSS UART**：调试、控制台和部分串口缓冲清理。

### 2.3 FPGA fabric CoreIP 外设

代码证据：

- `BMU_hw_platform.h`
- `drivers/CoreSPI`
- `drivers/CoreUARTapb`
- `drivers/CoreCAN`
- `drivers/CoreI2C`
- `drivers/CoreGPIO`
- `User/User_Driver/pl/pl.h`

`BMU_hw_platform.h` 给出了 M3 通过 FIC 访问 fabric 外设的地址映射：

| CoreIP | 地址/实例 | 工程用途 |
| --- | --- | --- |
| CoreSPI | `CORESPI_PLL` | AD9528、AD9528_1、预留 CDCM6208 |
| CoreSPI | `CORESPI_AD7490_0/1` | 两路 ADC SPI，挂 6 片 AD7490 |
| CoreSPI | `CORESPI_FLASH_0` 到 `CORESPI_FLASH_8` | 多路外部 SPI Flash |
| CoreUARTapb | SELFMG、KA_RX、KA_TX、SC、BMU、PLP0 | 多路 RS422/UART 通道 |
| CoreCAN | `CORECAN_BASE_ADD` | fabric CAN，注册为 `CAN_1` |
| CoreI2C | `PMBUS0/1/2` | PMBus/INA3221 电流采样适配 |
| CoreGPIO | OUT_ADDR/OUT_DATA/IN_DATA | PL 寄存器读写桥 |
| CoreMRAM | `COREMRAM_AHB_C0_0` | 主备、Flash、TMR 等状态持久化语义 |
| SERDES | `SERDES_IF2_C0_0` | SERDES 配置相关 |

这里的关键点是：项目不是只在 M3 内部跑协议，而是通过 FIC 访问 fabric CoreIP 和 PL 寄存器。这种 MCU + FPGA fabric 协同是工程复杂度的核心来源。

## 3. 软件架构分层

### 3.1 基础层

代码证据：

- `CMSIS/`
- `hal/`
- `drivers/`
- `drivers_config/`

基础层由厂商 CMSIS、HAL、MSS 驱动、CoreIP 驱动和 Libero 生成配置组成。这部分不能包装成自研底层驱动，但可以说明工程基于厂商驱动完成了板级适配和业务封装。

### 3.2 板级驱动层

代码证据：

- `User/User_Driver/can`
- `User/User_Driver/CoreUart`
- `User/User_Driver/uart`
- `User/User_Driver/i2c`
- `User/User_Driver/AD7490`
- `User/User_Driver/INA3221`
- `User/User_Driver/PLL`
- `User/User_Driver/pl`
- `User/User_Driver/MasterBackupSwitch`
- `User/User_Driver/shell`

板级驱动层把厂商驱动和具体硬件设计连接起来。例如：

- 用 `can_dev` 抽象统一 MSS CAN 和 CoreCAN。
- 用 CoreUART 封装多路 SC/KA/SELFMG/BMU/PLP0 串口。
- 用 CoreSPI 管理 AD7490、PLL、多片外部 Flash。
- 用 CoreGPIO 封装 PL 寄存器读写。
- 用主备状态机封装 GPIO、BMU 间 UART、PL/MRAM 状态。

### 3.3 业务应用层

代码证据：

- `User/APP/RemCT_StateMachine`
- `User/APP/RemCT_Can`
- `User/APP/RemCT_422`
- `User/APP/RemCT_ETH`
- `User/APP/CRC`
- `User/APP/IAP`
- `User/APP/AD`
- `User/APP/tmr`
- `User/APP/Reset`
- `User/APP/Menu`

业务应用层是主循环调度和命令处理的核心。它围绕遥测遥控入口做命令分发，并把命令落到 Flash、CRC、TMR、主备、AD、PLL、PL 寄存器、复位等具体模块。

### 3.4 存储、网络与组件层

代码证据：

- `User/IpConfig/Flash`
- `User/IpConfig/IP`
- `User/IpConfig/Tod`
- `User/component/DirectC`
- `User/component/CRC`
- `User/Self_Test`

这一层承担 Flash 分区映射、IPv6/UDP 封装、TOD 时间处理、DirectC/JTAG 编程组件适配、CRC32 算法和自检/产测能力。

## 4. 启动流程与主循环调度

### 4.1 启动初始化

代码证据：

- `main.c`
- `Fdd_Bmu_Init()`

启动主线：

1. 关闭 Watchdog。
2. 初始化服务层：控制台、delay、日志。
3. 通过 PL 寄存器读取槽位号 `0x201` 和 PL 版本 `0x100`。
4. 初始化 PLL SPI，并注册 AD9528、AD9528_1。
5. 初始化 SC、SELFMG、KA_TX、KA_RX 等多路 UART，主要波特率为 921600。
6. 初始化 CoreUART、BMU UART、MSS CAN、CoreCAN。
7. 初始化 MSS Flash 和 CoreSPI Flash。
8. 初始化 MSS GPIO，配置 Flash 控制、主备握手、对端状态输入输出。
9. 根据槽位选择 CAN 地址并初始化 BMU CAN。
10. 初始化 AD7490 电压采样。
11. 初始化 Ethernet MAC。
12. 初始化 Timer1。
13. 延时后配置 AD9528_1 和 AD9528。
14. 初始化 MSS System Services，读取 design version。
15. 初始化 PLP0 UART 接收、YCYK 帧、CRC 任务、UART FIFO。
16. 进入永久主循环。

### 4.2 裸机超级循环

代码证据：

- `User/APP/RemCT_StateMachine/RemCT_StateMachine.c`
- `Fdd_RemCT_StateMachine()`

主循环任务包括：

- `Fdd_RemCT_Can()`：CAN 遥测遥控处理。
- `Fdd_RemCT_422()`：RS422/UART 遥测遥控处理。
- `Fdd_RemCT_ETH()`：以太网遥测遥控处理。
- `uart_plp0_recv_task()`：PLP0 串口接收处理。
- `ad_monitor_task()`：AD 电压电流/过流监控。
- `bmu_flash_enum()`：Flash 枚举，完成后初始化 TMR/MRAM 状态。
- `MS_Switch_Deal_Task()`：主备切换任务。
- `Fdd_Flash_Erase_Work()`：Flash 擦除后台任务。
- `reset_task()`：整站软复位任务。
- `crc_task_run()`：CRC 后台任务。
- `bmu_menu()`：菜单/shell 调试入口。
- `Can_Recovery()`：CAN 恢复。
- `bmu_uart_frame_recv()`：BMU 间 UART 帧处理。
- `tmr_monitor_task()`：TMR 监控。

含金量点：

- 这是典型裸机协作式调度：不是所有功能都靠中断硬顶，而是将通信、Flash、CRC、AD、主备、TMR 等任务拆成可轮询状态机。
- 当前已有 Timer 计数和任务化基础，后续很适合补 Profiling，把“裸机实时性控制”变成可量化成果。

## 5. 核心模块含金量说明

### 5.1 多链路遥测遥控网关

代码证据：

- `User/APP/RemCT_Can/RemCT_Can.c`
- `User/APP/RemCT_422/RemCT_422.c`
- `User/APP/RemCT_ETH/RemCT_Eth.c`
- `User/IpConfig/IP/net_bmu_packet.c`
- `User/APP/UartRecv/uart_plp0.c`

已实现能力：

- CAN、RS422、Ethernet、UART 多链路进入同一遥测遥控业务体系。
- BMU 能本地处理状态、电压、Flash、CRC、TMR、主备、复位、寄存器读写等命令。
- BMU 不能处理或属于下游模块的命令，通过 `NET_DATA_SEND()`、PLP0 UART 或 CAN 封装转发。
- CAN 命令覆盖总线状态、总线切换/恢复、启动区选择、时间广播、Flash CRC、TMR、JTAG、主备切换、电压获取等。
- RS422/ETH 命令覆盖心跳、复位、文件传输、固件版本查询、软件版本查询、星历分发、自升级流程等。

难点：

- 不同链路的帧格式、带宽、错误模型不一样，但最终要统一到同一套业务命令。
- BMU 同时是命令执行端和网关转发端，需要处理本地/透传边界。
- 低带宽 CAN 与较高带宽 Ethernet/422 并存，需要处理分片、拼包、校验、超时和异常统计。

工程价值：

- 体现多源异构通信接入能力。
- 体现协议适配、命令分发、链路冗余和远程维护能力。
- 对嵌入式软件工程师来说，比单一 UART/CAN 收发更有系统含金量。

### 5.2 双 CAN 抽象与 CAN 多帧处理

代码证据：

- `User/User_Driver/can/can_dev.h`
- `User/User_Driver/can/can_dev.c`
- `User/User_Driver/can/sf2_mss_can.c`
- `User/User_Driver/can/sf2_core_can.c`
- `User/User_Driver/ycyk/ycyk_can.c`
- `User/APP/RemCT_Can/RemCT_Can.c`

已实现能力：

- `can_dev` 定义统一接口：`start/stop/reset/config/filter/send/recv/status`。
- `sf2_mss_can_init()` 将 MSS CAN 注册为 `CAN_0`。
- `sf2_core_can_init()` 将 fabric CoreCAN 注册为 `CAN_1`。
- 上层通过 `can_send_msg()`、`can_recv_msg()` 访问，不直接依赖底层控制器。
- CAN 接收侧有多帧接收、帧类型判断、bitmap、补充接收和错误计数基础。
- CAN ID 中编码优先级、源、组、目的、帧类型、计数、功能号等信息。

难点：

- MSS CAN 与 CoreCAN 寄存器和驱动接口不同，但业务层需要统一处理。
- CAN 大包必须拆成多帧，接收端要处理起始帧、中间帧、结束帧、单帧、漏帧、重复帧和异常帧。
- 双 CAN 链路还涉及过滤器、错误计数、总线恢复和主备策略。

工程价值：

- 体现驱动抽象能力，不是直接堆厂商 API。
- 体现低带宽总线下大包传输可靠性设计。
- 后续如果补超时恢复、重复帧过滤、乱序专项测试，会非常适合嵌入式通信可靠性包装。

### 5.3 RS422/UART FIFO 帧处理

代码证据：

- `User/User_Driver/CoreUart/CoreUart_RePackage.c`
- `User/APP/RemCT_422/Uart_Fifo_Frame.c`
- `User/APP/RemCT_422/RemCT_422.c`
- `User/User_Driver/fifo/fifo.c`

已实现能力：

- 多路 CoreUARTapb 通道：SC、KA_TX、KA_RX、SELFMG、BMU、PLP0。
- UART 初始化支持 921600 和保留的 115200 配置。
- 软件 FIFO 缓冲接收数据。
- `Uart_Fifo_Frame` 做帧提取、状态判断、ready/reset 和错误统计。
- RS422 业务帧支持复位、心跳、文件传输开始/数据/结束/异常终止、版本查询、星历分发、自升级等命令。
- 部分通道接收后直接通过 `NET_DATA_SEND()` 转成统一网络封装。

难点：

- 高频串口不能在主循环里逐字节阻塞等待，必须用 FIFO 和任务化解析。
- 多路串口都可能产生业务帧，需要共享解析策略但保持通道状态独立。
- 文件传输和普通控制命令复用同一接收链路，异常帧恢复和缓冲边界更复杂。

工程价值：

- 体现软件 FIFO、帧同步、串口高频接收和异常恢复能力。
- 后续补 FIFO 水位、滑动重同步、单轮处理预算后，可以形成很漂亮的“UART/RS422 接收链路可靠性优化”成果。

### 5.4 Ethernet MAC、队列与 IPv6/UDP 适配

代码证据：

- `User/APP/RemCT_ETH/eth/eth.c`
- `User/APP/RemCT_ETH/RemCT_Eth.c`
- `User/IpConfig/IP/net_utils.c`
- `User/IpConfig/IP/net_bmu_packet.c`
- `User/IpConfig/IP/net_addr.c`

已实现能力：

- MSS Ethernet MAC 初始化，配置 MAC 地址、自协商 1000M 全双工、最大帧长。
- RX callback 中把接收包放入软件队列，主循环消费，避免在中断/回调里做重业务。
- TX 侧维护发送 buffer 状态，发送完成回调释放 buffer。
- 有软 RX/TX 计数、drop 计数、TX error 计数。
- 网络层支持 MAC、VLAN、IPv6、UDP 解析与构造，以及 IPv6 UDP checksum。
- Ethernet 业务最终进入和 RS422 类似的遥测遥控命令处理流程。

难点：

- MAC 驱动回调与上层协议处理要解耦，否则突发包会拖慢中断/回调路径。
- TX ring/buffer 满时需要处理丢包和错误统计。
- IPv6/UDP/VLAN 封装比普通裸 UDP Demo 更接近真实通信协议适配。

工程价值：

- 体现网络收发队列、协议解析、驱动回调解耦能力。
- 后续补队列峰值、丢包原因、TX pending/重试策略，可以增强“突发流量下稳定性”叙事。

### 5.5 Flash 分区、镜像管理、IAP 与 CRC

代码证据：

- `User/IpConfig/Flash/flash.h`
- `User/IpConfig/Flash/flash.c`
- `User/IpConfig/Flash/bmu_flash.c`
- `User/IpConfig/Flash/file_flash.c`
- `User/APP/IAP/iap.c`
- `User/APP/CRC/crc_task.c`
- `User/component/CRC/crc32.c`

已实现能力：

- Flash 文件类型包含 FPGA、SCP、BBPS、BBPKA、BMU。
- CPU 分区包含 app、cfg、os、exc、ata2。
- BMU 分区包含 dir、update、golden、iap。
- 文件子类型用 bitfield 表示槽位、区号、主备属性。
- 根据文件类型映射到 SPI 控制器、片选、起始地址、长度和镜像类别。
- 支持 UART/ETH 文件传输升级；BMU 本地可擦写，其他模块可透传。
- BMU 自身 IAP 通过 `MSS_SYS_initiate_iap()` 执行 authenticate/program/verify。
- CRC 分为两条路径：
  - BMU 本地 Flash 由 M3 分块读取并软件 CRC32。
  - SCP/BBPS/BBPKA/PLP0/PLP1 等由 M3 配置 PL 寄存器触发硬件 CRC，并读取结果寄存器。

难点：

- 升级不是写死一个 Flash 地址，而是要解析文件类型、槽位、区号、主备，再映射到实际 Flash 和分区。
- BMU 自升级和下游模块透传升级边界不同。
- Flash 擦写、CRC 校验和通信处理同时存在，裸机主循环容易被长耗时操作影响。
- CRC 同时有 M3 软件路径和 PL 硬件路径，要处理状态、结果、冲突和查询。

工程价值：

- 体现远程固件维护、镜像管理、分区管理和完整性校验能力。
- IAP + golden/update/iap 分区语义比普通 Bootloader 描述更有系统性。
- 后续补 Flash 写入后台化、CRC 分块可配置、进度/错误码回传，能显著增强嵌入式软件工程师含金量。

### 5.6 DirectC/JTAG 互助重构

代码证据：

- `User/APP/RemCT_Can/RemCT_Can.c`
- `CAN_CMD_JTAG_HELP 0x9A13`
- `ycyk_can_proc_jtag_help()`
- `User/component/DirectC/dpuser.c`
- `User/component/DirectC/dpcom.c`
- `User/component/DirectC/dpalg.c`
- `User/component/DirectC/JTAG`
- `User/component/DirectC/G3Algo`
- `User/component/DirectC/G4Algo`
- `User/component/DirectC/G5Algo`
- `User/component/DirectC/RTG4Algo`

已实现能力：

- CAN 命令触发 JTAG 互助重构。
- BMU 作为嵌入式 JTAG Host，设置 `Action_code = DP_PROGRAM_ACTION_CODE` 后调用 `dp_top()`。
- `dpuser.c` 中通过 `JTAG_WRITE`、`JTAG_READ` 适配 JTAG TMS/TDI/TDO/TCK 时序。
- DirectC 从外部 Flash 分页读取镜像数据。
- 执行结果通过 `u8JtagHelpRet` 和打印信息记录。
- 已统计 `dp_top()` 总耗时和 Flash 读取次数 `g_read_flash_times`。

边界说明：

- DirectC 是 Microchip/Microsemi 厂商组件，不应说成自研编程算法。
- 工程价值在于 DirectC 移植适配、JTAG 硬件抽象、CAN 远程触发、Flash 镜像读取、状态回传和诊断。

难点：

- IAP 是更新 M3 自己；JTAG 互助重构是 BMU 帮其他 FPGA/配置器件在线编程，能力边界不同。
- 涉及 JTAG 时序适配、厂商算法入口、Flash 分页读取、远程命令触发和失败诊断。
- 要保证远程触发条件、槽位参数和目标器件路径正确，避免误编程。

工程价值：

- 这是本工程最能体现“远程维护/在线重构”的模块之一。
- 对嵌入式软件工程师来说，它体现的是跨器件维护、厂商组件适配、底层时序接口和远程命令系统的结合。
- 后续补阶段耗时、错误码映射、page size 对比，会很容易做出数据化成果。

### 5.7 M3-PL 寄存器协同

代码证据：

- `User/User_Driver/pl/pl.h`
- `User/User_Driver/pl/pl.c`
- `BMU_hw_platform.h`
- `User/APP/CRC/crc_task.c`
- `User/APP/tmr/tmr.c`
- `User/APP/AD/ad_monitor.c`
- `User/User_Driver/MasterBackupSwitch/MasterBackupSwitch.c`

已实现能力：

- 通过 CoreGPIO 地址、写数据、读数据寄存器封装 PL 寄存器访问。
- PL 寄存器承载槽位号、PL 版本、Flash 选择、MRAM 更新、CRC 控制、TMR 控制、AD 异常、PG、主备状态、复位等控制面。
- M3 侧通过 `pl_write_reg()` 下发控制，通过 `pl_read_reg()` 获取状态。

难点：

- 这不是单纯 MCU 软件逻辑，而是软硬件状态机共同完成。
- 软件要理解 PL 寄存器地址、触发位、回读状态、清除流程和持久化逻辑。
- CRC、TMR、AD、主备等模块都依赖这条边界，调试复杂度高。

工程价值：

- 体现 MCU + FPGA 协同开发能力。
- 体现寄存器协议、硬件状态控制、跨模块状态一致性。
- 这是这个工程从“普通 M3 固件”升级到“异构 SoC 板级控制系统”的关键证据。

### 5.8 主备 BMU、MRAM 与 TMR/三模

代码证据：

- `User/User_Driver/MasterBackupSwitch/MasterBackupSwitch.c`
- `User/User_Driver/MasterBackupSwitch/MasterBackupSwitch.h`
- `User/APP/tmr/tmr.c`
- `User/APP/tmr/tmr.h`
- `User/APP/UartRecv/uart_bmu.c`

已实现能力：

- 主备状态使用 `0x55` 和 `0xAA`。
- GPIO5/6/15/16 用于本端/对端状态和配置握手。
- BMU 间 UART 帧同步主备参数，处理切换请求、应答、参数请求、参数应答、强制切换。
- MRAM/PL 寄存器保存主备标志、Flash 选择、三模状态等关键参数。
- 初始化场景和运行时切换场景分开处理。
- 主备切换会更新 GPIO 输出、PL 寄存器、CAN 处理策略，并触发 AD9528 相关动作。
- TMR 模块支持打开/关闭三模、强制打开指定模块、清除不可恢复校验组、保存到 MRAM。

难点：

- 主备切换不是单点 GPIO 翻转，而是状态机：本端 MRAM、对端参数、槽位优先级、运行时命令、PL 状态要一致。
- TMR 与 MRAM、Flash 轮转、PL 寄存器和主备状态关联。
- 主备模式还会影响 CAN 命令处理策略。

工程价值：

- 体现高可靠控制、状态持久化、故障恢复和冗余切换。
- 适合和远程升级、Flash 镜像、TMR 一起讲成可靠性控制闭环。

### 5.9 AD/PLL 健康监测与板级控制

代码证据：

- `User/User_Driver/AD7490/ad7490.c`
- `User/User_Driver/AD7490/ad7490.h`
- `User/APP/AD/ad_monitor.c`
- `User/APP/AD/ad_monitor.h`
- `User/User_Driver/INA3221`
- `User/User_Driver/PLL/pll.c`
- `User/User_Driver/PLL/ad9528.c`
- `User/User_Driver/PLL/ad9528_1.c`

已实现能力：

- 2 路 CoreSPI 控制 6 片 AD7490，每片 16 通道。
- 电压采样覆盖 BBPS、BBPKA、SCP、PLP0、PLP1、BMU、TRX、SSD、PLL、OCXO 等大量电源轨。
- `ad_monitor_task()` 周期读取 PL 侧异常状态、PG、电压值和电流换算值。
- 支持阈值读写、异常状态查询、异常打印和状态上报基础。
- PLL 模块通过 CoreSPI 控制 AD9528、AD9528_1，CDCM6208 驱动保留。

难点：

- 健康监测不是单路 ADC，而是多片 ADC、多模块电源轨和 PL 过流状态协同。
- 电源异常需要锁存、读取、清除、记录时间戳并进入遥测体系。
- PLL 配置影响板级关键时钟，属于上电初始化和现场调试的重要部分。

工程价值：

- 体现板级 Bring-up、健康监测、电源轨管理和时钟芯片控制能力。
- 对嵌入式软件工程师而言，这是比单纯通信协议更贴近硬件系统的经验。

## 6. 代码证据索引

| 模块 | 关键路径 | 证据点 |
| --- | --- | --- |
| 入口与初始化 | `main.c` | 外设初始化、槽位/PL版本读取、主循环入口 |
| 主循环调度 | `User/APP/RemCT_StateMachine/RemCT_StateMachine.c` | 超级循环任务编排 |
| CAN 业务 | `User/APP/RemCT_Can/RemCT_Can.c` | CAN 命令表、本地处理、透传、JTAG/TMR/CRC 命令 |
| CAN 抽象 | `User/User_Driver/can/can_dev.*` | `can_dev` 统一接口 |
| MSS CAN | `User/User_Driver/can/sf2_mss_can.c` | MSS CAN 注册为 `CAN_0` |
| CoreCAN | `User/User_Driver/can/sf2_core_can.c` | CoreCAN 注册为 `CAN_1` |
| CAN 多帧 | `User/User_Driver/ycyk/ycyk_can.c` | 多帧接收、bitmap、错误统计 |
| RS422 | `User/APP/RemCT_422/RemCT_422.c` | 串口业务帧处理和透传 |
| UART FIFO | `User/APP/RemCT_422/Uart_Fifo_Frame.c` | FIFO 帧提取、ready/reset、错误统计 |
| Ethernet MAC | `User/APP/RemCT_ETH/eth/eth.c` | MAC RX callback、队列、TX buffer、统计 |
| IPv6/UDP | `User/IpConfig/IP/net_bmu_packet.c` | 网络封装和 `NET_DATA_SEND()` |
| Flash 映射 | `User/IpConfig/Flash/flash.h` | 文件类型、分区、slot/region/主备 bitfield |
| Flash 任务 | `User/IpConfig/Flash/flash.c`、`file_flash.c` | 擦写、读取、文件传输支持 |
| IAP | `User/APP/IAP/iap.c` | System Services authenticate/program/verify |
| CRC | `User/APP/CRC/crc_task.c` | BMU 软件 CRC、PL 硬件 CRC 触发 |
| DirectC/JTAG | `User/component/DirectC/dpuser.c` | JTAG TMS/TDI/TDO 适配 |
| DirectC 入口 | `User/component/DirectC/dpalg.c` | `dp_top()` 分发 G3/G4/G5/RTG4/SPIFlash |
| PL 寄存器 | `User/User_Driver/pl/pl.h` | CoreGPIO 读写 PL 寄存器 |
| 主备 | `User/User_Driver/MasterBackupSwitch/MasterBackupSwitch.c` | 主备状态机、GPIO、MRAM、BMU UART |
| TMR | `User/APP/tmr/tmr.c` | 三模开关、强制模式、MRAM 保存 |
| AD 监控 | `User/APP/AD/ad_monitor.c` | PL 异常寄存器读取、状态记录 |
| AD7490 | `User/User_Driver/AD7490/ad7490.c` | 多片 ADC SPI 读取 |
| PLL | `User/User_Driver/PLL/pll.c` | AD9528/AD9528_1 注册、配置、读写 |
| 自检/产测 | `User/Self_Test/Self_Test.c` | Flash/CAN/PLL/电压电流测试入口 |

## 7. 为嵌入式软件工程师服务的补强工作

这些是建议后续真正补代码或补实验数据的方向。它们不属于当前已完成能力，文档中应明确标为“可补强”。

### 7.1 裸机 Profiling 与任务执行预算

当前基础：

- 已有 Timer 计数。
- 主循环任务边界清晰。
- CRC 等部分任务已经状态机化。

建议补强：

- 给每个主循环任务增加 `min/max/avg` 耗时统计。
- 统计主循环平均周期、最大周期和 jitter。
- 为 Flash、CRC、AD、Ethernet 等任务设置单轮执行预算。
- 提供 shell/菜单查询接口。

价值：

- 把“裸机超级循环”升级成“可观测、可约束的裸机实时调度”。
- 这是嵌入式软件工程师非常硬的工程能力。

### 7.2 通信可靠性增强

建议补强：

- UART FIFO 增加高水位、溢出计数、最大连续接收字节数。
- UART 帧头错误后从整体 reset 优化为滑动重同步。
- CAN 多帧增加接收超时，避免 FSM 卡半包。
- CAN 重复帧只更新 bitmap，不重复增加接收计数。
- 做漏帧、乱序、重复帧、异常字节插入专项测试。

价值：

- 把“能收发”升级成“异常流量下可恢复”。
- 对 CAN/RS422 这种现场链路尤其有说服力。

### 7.3 Flash/CRC/升级后台化

建议补强：

- Flash 写入改成后台状态机。
- CRC 分块大小可配置，测试 512B/1KB/2KB/4KB。
- Flash 擦写、CRC 校验期间持续响应通信任务。
- 增加进度、错误码、超时状态回传。

价值：

- 把“能升级”升级成“升级过程中仍保持通信实时性”。
- 这类优化很容易拿到数据：总耗时、单轮阻塞时间、通信丢帧率。

### 7.4 DirectC/JTAG 可观测性

当前基础：

- CAN 触发 `dp_top()`。
- 已打印总耗时和 Flash 读取次数。

建议补强：

- 增加阶段耗时：读镜像、IDCODE、擦除、编程、校验。
- 调整 DirectC page buffer size，测试读取次数和总耗时。
- 把 DirectC 返回码映射成可读错误诊断。
- 记录最近一次 JTAG 重构结果，支持远程查询。

价值：

- 把“能调用 DirectC”升级成“能远程诊断重构过程”。
- 这是远程维护类项目很有分量的补强点。

### 7.5 Ethernet 队列诊断与背压

建议补强：

- 统计 RX/TX 队列峰值深度。
- 队列满时记录丢包原因。
- TX 侧增加 pending 队列或重试策略。
- 压测 UDP 突发包，记录队列峰值和丢包率。

价值：

- 把“有以太网队列”升级成“突发网络流量下可观测、可定位”。
- 适合和裸机 Profiling 一起形成系统级稳定性数据。

## 8. 含金量排序建议

如果给人讲，不建议从所有外设逐个报菜名。建议按含金量排序：

1. **M3 + PL 异构协同架构**：这是项目底座，区别于普通 MCU 工程。
2. **多链路遥测遥控网关**：体现通信复杂度和协议适配能力。
3. **Flash/IAP/CRC 在线维护体系**：体现远程升级和镜像完整性。
4. **DirectC/JTAG 互助重构**：体现跨器件在线编程和远程恢复能力。
5. **主备/TMR/MRAM 可靠性控制**：体现高可靠状态管理。
6. **CAN/UART/Ethernet 数据通路优化空间**：体现嵌入式软件工程师可继续做深的方向。
7. **AD/PLL/自检/产测**：作为板级控制和现场维护支撑。

## 9. 边界与注意事项

- CMSIS、MSS 驱动、CoreIP 驱动、DirectC 是厂商组件，不能写成自研。
- 可以强调“基于厂商驱动完成板级适配、抽象封装、业务集成、远程触发和诊断能力建设”。
- 当前 `sf2_core_i2c_init()` 在 `main.c` 中被注释，INA3221/PMBus 是否量产启用需要结合硬件和构建配置确认。
- CDCM6208 驱动存在，但主初始化中配置被注释，可能是历史方案或备用芯片。
- 具体 SmartFusion2 器件型号未在应用层直接固定，准确型号应以 Libero 工程/硬件设计为准。
- 部分源码中文注释存在编码混用，正式文档和代码注释建议统一 UTF-8。

## 10. 后续可继续整理的专题文档

如果后续要把这个项目继续打磨成一套完整材料，建议拆成这些专题：

- `PL寄存器地图.md`：地址、方向、位定义、触发方式、回读状态。
- `Flash分区与镜像映射.md`：文件类型、slot、region、CS、起始地址、长度。
- `遥测遥控命令表.md`：CAN/422/ETH 命令码、参数、响应、错误码。
- `主备切换状态机.md`：状态、触发条件、GPIO、MRAM、PL 寄存器、异常场景。
- `升级与IAP流程.md`：文件传输、Flash 擦写、CRC、IAP、失败恢复。
- `DirectC_JTAG互助重构流程.md`：触发命令、镜像来源、JTAG 接口、错误码、耗时统计。
- `裸机任务调度与耗时统计.md`：主循环任务、周期、最大耗时、优化数据。


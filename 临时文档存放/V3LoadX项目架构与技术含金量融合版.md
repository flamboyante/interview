# V3LoadX 项目架构与技术含金量融合版

> 本文融合 `project_overview_interview.md` 与 `CortexM3工程简历亮点与优化计划总结.md`。重点记录这个 MCU 工程“做了什么、用了什么外设、项目怎么分层、技术含金量在哪里、当前还存在哪些可优化点”。不展开面试话术。

## 1. 项目总体定位

`V3LoadX` 是一个运行在 Microchip/Microsemi SmartFusion2 Cortex-M3 上的 BMU 管理控制固件。它不是单一外设 Demo，也不是只做主备切换的局部模块，而是一个板级管理控制系统：负责多链路通信接入、遥测遥控处理、远程升级、Flash 镜像管理、JTAG 互助重构、电压电流监控、主备冗余、TMR/三模控制、PL 寄存器协同和现场调试维护。

从工程形态看，它属于“Cortex-M3 裸机高可靠通信控制与在线维护系统”。M3 侧承担协议、状态机、任务调度和控制决策；FPGA fabric/PL 侧承担 CoreIP 外设、共享寄存器、硬件状态、CRC/TMR/采样/主备相关逻辑。二者通过 fabric 地址空间和 PL 寄存器接口配合。

## 2. 硬件平台与运行环境

- SoC/MCU：Microchip/Microsemi SmartFusion2 系列。
- 内核：Arm Cortex-M3，工程配置为 `-mcpu=cortex-m3`。
- 工程工具：Libero 2023.1 导出的 firmware 工程，Eclipse/SoftConsole GNU Arm 工程。
- 系统形态：裸机超级循环，无 RTOS；通过主循环轮询、状态机、外设回调、Timer 中断协作。
- CMSIS：`CMSIS/m2sxxx.h`、`CMSIS/system_m2sxxx.c`、`CMSIS/startup_gcc/startup_m2sxxx.S`。
- 链接脚本：
  - Debug：`CMSIS/startup_gcc/debug-in-microsemi-smartfusion2-envm.ld`
  - Release：`CMSIS/startup_gcc/production-smartfusion2-execute-in-place.ld`
- 主频与总线：
  - Cortex-M3：128 MHz
  - APB0：64 MHz
  - APB1/APB2：32 MHz
  - FIC0：64 MHz
  - FIC1/FIC64：128 MHz
- 系统配置：
  - MDDR/FDDR 不由 Cortex-M3 配置。
  - SERDES0 由 Cortex-M3 配置。

## 3. 外设资源

### 3.1 SmartFusion2 MSS 原生外设

- MSS GPIO：主备握手、对端 BMU 状态输入输出、Flash 控制脚、JTAG/DirectC 相关 GPIO 等。
- MSS CAN：作为一条 CAN 通道注册到统一 CAN 抽象层，逻辑上为 `CAN_0`。
- MSS SPI：用于 BMU 本地 SPI Flash、IAP 镜像访问和系统服务相关 Flash 操作。
- MSS Ethernet MAC：以太网接入，支持 MAC 收发、RX 回调、软件队列缓存、状态统计。
- MSS Timer1/Timer2：裸机时间基准、任务耗时统计、周期任务、延时和中断回调。
- MSS System Services：读取 design version，执行 SmartFusion2 IAP authenticate/program/verify。
- MSS UART：调试、控制台、部分清理/输出逻辑。
- MSS NVM/HPDMA 等厂商驱动也在工程内，但主业务重点集中在通信、Flash、MAC、GPIO、Timer、System Services。

### 3.2 FPGA fabric CoreIP 外设

`BMU_hw_platform.h` 是 fabric 侧外设地址地图，说明 M3 通过 FIC 访问 PL 内部 CoreIP。

- CoreSPI：
  - `CORESPI_PLL`：访问 AD9528、AD9528_1、预留 CDCM6208。
  - `CORESPI_AD7490_0/1`：两路 ADC SPI，总共挂 6 片 AD7490。
  - `CORESPI_FLASH_0` 到 `CORESPI_FLASH_8`：多路外部 SPI Flash。
- CoreUARTapb：
  - `COREUARTAPB_C0_SELFMG_0`
  - `COREUARTAPB_C0_KA_RX_0`
  - `COREUARTAPB_C1_KA_TX_0`
  - `COREUARTAPB_C0_SC_0`
  - `COREUARTAPB_C1_BMU`
  - `COREUARTAPB_C1_PLP0`
- CoreCAN：
  - `CORECAN_BASE_ADD`，注册为统一 CAN 抽象层的 `CAN_1`。
- CoreI2C：
  - `COREI2C_PMBUS0/1/2`，工程内有 CoreI2C 适配层和 INA3221 电流采样驱动。
- CoreGPIO：
  - `COREGPIO_OUT_ADDR`
  - `COREGPIO_OUT_DATA`
  - `COREGPIO_IN_DATA`
  - 用于封装 `pl_read_reg()` / `pl_write_reg()`，这是 M3 与 PL 交互的关键通道。
- CoreMRAM：
  - `COREMRAM_AHB_C0_0`，主备参数、Flash 选择、TMR 状态等具有 MRAM 持久化语义。
- SERDES：
  - `SERDES_IF2_C0_0`，系统配置中 SERDES0 由 Cortex-M3 配置。

## 4. 软件架构分层

### 4.1 基础层

- `CMSIS/`：Cortex-M3 启动、异常向量、SmartFusion2 片上寄存器定义、系统时钟初始化。
- `hal/`：NVIC、寄存器访问、Cortex-M3 HAL。
- `drivers/`：Microchip/Microsemi 厂商 MSS 驱动，以及 CoreSPI/CoreUART/CoreI2C/CoreGPIO/CoreCAN 等 fabric CoreIP 驱动。
- `drivers_config/`：Libero 生成的时钟、SERDES、系统配置文件。

### 4.2 板级驱动层

位置主要在 `User/User_Driver/`：

- CAN 抽象与适配：`can_dev`、`sf2_mss_can`、`sf2_core_can`。
- UART 封装：CoreUART 初始化、收发、超时接收、FIFO 帧提取。
- I2C 适配：CoreI2C adapter、PMBus/INA3221 支持。
- ADC：AD7490 多片多通道电压采样。
- PLL：AD9528、AD9528_1、CDCM6208 驱动与寄存器读写。
- PL：`pl_read_reg()` / `pl_write_reg()`。
- Power：电源相关控制。
- MasterBackupSwitch：主备 BMU 状态机。
- shell：本地调试命令入口。

### 4.3 业务应用层

位置主要在 `User/APP/`：

- `RemCT_StateMachine`：主循环业务调度中心。
- `RemCT_Can`：CAN 遥测遥控命令处理。
- `RemCT_422`：RS422/串口遥测遥控命令处理。
- `RemCT_ETH`：以太网遥测遥控命令处理。
- `RemCT_CPU`：与 CPU/下游模块的版本、升级、系统寄存器等交互。
- `AD`：电压电流监测与异常上报。
- `CRC`：Flash CRC 任务。
- `IAP`：SmartFusion2 在线升级。
- `Reset`：整站软复位任务。
- `tmr`：TMR/三模控制。
- `Menu`：菜单和调试入口。
- `UartRecv`：PLP0/BMU UART 接收处理。

### 4.4 协议、存储与组件层

- `User/IpConfig/IP`：MAC/VLAN/IPv6/UDP 解析、构造、checksum、BMU 自定义网络包封装。
- `User/IpConfig/Flash`：Flash 分区、文件类型映射、SPI Flash 操作、BMU Flash 枚举。
- `User/IpConfig/Tod`：TOD 时间解析和 SPI/UART 时间相关处理。
- `User/component/DirectC`：Microchip DirectC/JTAG 编程组件，工程对其做硬件适配和业务触发。
- `User/component/CRC`：CRC32 基础算法。
- `User/Self_Test`：自检、产测、Flash/CAN/PLL/电压电流等测试入口。

## 5. 启动链路与主循环

入口是 `main.c`，核心初始化函数是 `Fdd_Bmu_Init()`。

主要启动步骤：

1. 关闭 watchdog。
2. 初始化服务层：日志、控制台、delay。
3. 读取 PL 槽位号 `0x201` 和 PL 版本 `0x100`。
4. 初始化 PLL SPI，并注册 AD9528、AD9528_1。
5. 初始化多路 CoreUART，SC/SELFMG/KA_TX/KA_RX 主要使用 921600 波特率。
6. 初始化 CoreUART 设备层、BMU UART、MSS CAN、CoreCAN。
7. 初始化 MSS SPI Flash 和 CoreSPI Flash。
8. 初始化 MSS GPIO，配置 Flash 控制、主备握手、对端 BMU 状态等 IO。
9. 按槽位选择 CAN 地址并初始化 BMU CAN。
10. 初始化 AD7490 电压采样。
11. 初始化 Ethernet MAC。
12. 初始化 Timer1。
13. 延时后配置 AD9528_1 和 AD9528。
14. 初始化 MSS System Services，读取 design version。
15. 初始化 PLP0 串口接收、YCYK 帧、CRC 任务、UART FIFO。
16. 进入 `while (1)`，循环执行 `Fdd_RemCT_StateMachine()`。

`Fdd_RemCT_StateMachine()` 是业务调度中心，循环执行：

- CAN 遥控遥测处理。
- RS422 遥控遥测处理。
- Ethernet 遥控遥测处理。
- PLP0 串口接收任务。
- AD 电压/电流监控。
- Flash 枚举；枚举完成后触发主备切换处理和 TMR 初始化。
- Flash 擦除任务。
- 软复位任务。
- CRC 后台任务。
- 本地菜单/shell。
- CAN 恢复逻辑。
- BMU 间 UART 帧处理。
- TMR 监控任务。

## 6. 核心功能模块

### 6.1 多链路通信与遥测遥控

这是整个工程最核心的业务。上行命令可能来自 CAN、RS422、以太网、UART 等通道，进入后统一做帧解析、校验、命令码分发、本地处理或透传。

链路能力：

- CAN A/B：由 `Fdd_RemCT_Can()` 轮询；底层同时支持 MSS CAN 和 CoreCAN。
- RS422：由 `Fdd_RemCT_422()` 处理 SC、KA_TX、KA_RX、SELFMG 等通道。
- Ethernet：由 MSS MAC RX callback 入队，主循环从队列取包处理，网络层支持 MAC/VLAN/IPv6/UDP。
- PLP0/CPU 透传：BMU 不处理的命令会封装转发给 CPU/PLP0。

命令能力覆盖：

- 遥测快变量、慢变量。
- CAN 总线状态查询、切换、恢复。
- 时间广播。
- 电源配置与断电查询。
- 启动区选择。
- Flash CRC 启动与结果查询。
- Flash 轮转/镜像选择。
- 电压获取、电流监控控制。
- BMU 寄存器读写。
- TMR 配置。
- JTAG 互助重构。
- 主备 BMU 切换。
- 日志生成。
- 软复位。

技术含金量：

- 多种物理链路接入同一套业务命令，通道层和业务层解耦。
- 低带宽 CAN 和较高带宽以太网/422 同时存在，需要处理分片、拼包、超时、错误统计和转发。
- BMU 既是命令执行端，也是网关/转发节点，对协议边界和异常路径要求高。

### 6.2 CAN 抽象与双 CAN 支持

工程定义了统一 `can_dev` 抽象，向上提供 `start/stop/reset/config/filter/send/recv/status`，向下适配两种 CAN 控制器：

- `sf2_mss_can.c`：SmartFusion2 MSS CAN，注册为 `CAN_0`。
- `sf2_core_can.c`：fabric CoreCAN，注册为 `CAN_1`。

上层 `RemCT_Can.c` 通过统一接口收发，不直接依赖具体 CAN 控制器。

技术含金量：

- 屏蔽 MSS 原生外设与 fabric CoreIP 的差异。
- 一个业务层可以同时处理双 CAN 链路。
- 支持 CAN 扩展帧、自定义 ID 编码、过滤器配置、错误计数、状态查询和总线恢复。
- CAN 多帧接收侧具备 FSM、bitmap、补帧/异常统计等基础能力，适合低带宽总线传输大包。

### 6.3 RS422/UART FIFO 帧处理

RS422/UART 侧负责 SC、KA_TX、KA_RX、SELFMG 等多通道数据接入。工程中已有：

- CoreUARTapb 初始化与波特率配置。
- 软件 FIFO。
- 1ms 定时搬运。
- 多通道帧缓存。
- 状态机帧解析。
- 4 字节对齐写 FIFO。
- 帧头同步、APID 识别、长度校验、checksum/CRC 校验。
- 异常帧统计和恢复入口。

RS422 协议支持：

- 复位。
- 心跳。
- 文件传输开始、数据、结束、异常终止。
- 固件版本查询。
- 软件版本查询。
- 星历分发。
- 自升级预处理/开始/结果查询。

技术含金量：

- 高频串口接收不能长期阻塞主循环，因此 FIFO、批量搬运、帧状态机是必要设计。
- 多通道串口统一处理，适合做链路冗余和通道转发。
- 文件传输和普通控制命令复用同一帧处理路径，异常处理复杂度较高。

### 6.4 Ethernet 与 IPv6/UDP 适配

以太网使用 MSS Ethernet MAC。工程中不仅调用 MAC 驱动，还实现了自定义网络协议栈适配：

- MAC 收包回调。
- RX 软件队列。
- TX buffer 状态管理。
- MAC/VLAN/IPv6/UDP 解析。
- IPv6 UDP checksum。
- BMU 自定义包构造和发送。
- 队列满、丢包、发送失败等统计基础。

技术含金量：

- MAC 驱动收包和上层协议处理解耦，避免在回调里做重业务。
- 支持 VLAN/IPv6/UDP，说明不是简单裸 UDP Demo。
- 以太网与 422/CAN 进入同一遥测遥控业务体系，承担高速链路入口和转发出口。

### 6.5 Flash 分区、镜像与远程升级

工程支持多模块、多分区、多 Flash 片映射。`flash.h` 中定义了 CPU、FPGA、BMU 等不同类型镜像和分区。

主要镜像类型：

- FPGA
- SCP
- BBPS
- BBPKA
- BMU

BMU 分区：

- `bmu_dir`
- `bmu_update`
- `bmu_golden`
- `bmu_iap`

CPU 分区：

- `cpu_app`
- `cpu_cfg`
- `cpu_os`
- `cpu_exc`
- `cpu_ata2`

关键机制：

- 文件子类型中用 bitfield 表示槽位、区号、主备属性。
- 根据文件类型映射到 SPI 控制器、片选、起始地址、长度和镜像类别。
- 支持 422/以太网文件传输。
- 目标是 BMU 时本地擦写；目标是 CPU/其他模块时透传。
- BMU 本体可通过 SmartFusion2 System Services 执行 IAP。

技术含金量：

- 分区管理不是单地址写入，而是面向多模块、多槽位、多镜像类型的映射系统。
- 存在 update/golden/iap 等恢复语义，具备远程维护和异常恢复基础。
- 升级过程要同时处理通信帧、Flash 擦写、ACK、状态回传、CRC 校验和主循环实时性。

### 6.6 IAP 在线升级

`User/APP/IAP/iap.c` 使用 SmartFusion2 MSS System Services：

- `MSS_SYS_PROG_AUTHENTICATE`
- `MSS_SYS_PROG_PROGRAM`
- `MSS_SYS_PROG_VERIFY`

流程中通过 MSS SPI 选中外部 Flash，并从指定偏移触发系统服务执行编程动作。

技术含金量：

- 这是 BMU 自身固件在线更新能力，不同于普通外部模块文件透传。
- 涉及厂商安全/认证/编程流程、Flash 镜像位置、启动区维护和失败恢复。
- 对无人值守设备或远程设备维护价值较高。

### 6.7 DirectC/JTAG 互助重构

这是工程中含金量较高、容易被忽略的一块。DirectC 是 Microchip/Microsemi 的器件编程组件，工程做的是集成适配和远程触发。

相关路径：

- `User/component/DirectC/dpuser.c`
- `User/component/DirectC/dpcom.c`
- `User/component/DirectC/JTAG`
- `User/component/DirectC/G3Algo/G4Algo/G5Algo/RTG4Algo`
- `User/APP/RemCT_Can/RemCT_Can.c` 中的 `CAN_CMD_JTAG_HELP 0x9A13` 和 `ycyk_can_proc_jtag_help()`

主要能力：

- BMU 作为嵌入式 JTAG Host。
- 通过 CAN 遥控命令触发 JTAG 互助重构。
- 通过 GPIO/PL 适配 JTAG TMS/TDI/TDO 时序。
- DirectC 从外部 Flash 分页读取镜像。
- 执行目标器件编程、校验、错误码返回和状态回传。

技术含金量：

- IAP 是更新 M3 自己，DirectC/JTAG 是 BMU 帮其他 FPGA/配置器件在线编程，二者能力边界不同。
- 涉及厂商编程算法、JTAG 时序适配、Flash 分页读取、远程命令触发和错误诊断。
- 适合体现跨器件维护、远程重构、现场恢复能力。

注意边界：

- DirectC 算法库是厂商组件，不应描述为自研。
- 工程价值在于移植适配、硬件抽象、远程触发流程、镜像读取、状态回传和诊断。

### 6.8 CRC 校验

CRC 校验分两类：

- BMU 本地 Flash：M3 分块读取 SPI Flash，使用 CRC32 软件算法计算。
- SCP/BBPS/BBPKA/PLP0/PLP1 等外部模块：M3 通过 PL 寄存器下发 Flash 起始地址、长度和启动位，由 PL 侧完成 CRC，M3 读取结果寄存器。

相关路径：

- `User/APP/CRC/crc_task.c`
- `User/component/CRC/crc32.c`
- `User/IpConfig/Flash`
- `User/User_Driver/pl`

技术含金量：

- CRC 已经做成后台状态机，不是一次性长时间阻塞。
- 本地软件 CRC 与 PL 硬件 CRC 分工明确。
- CRC 任务会避免升级中冲突，并支持启动、复位、停止、结果查询。
- 适合保障镜像完整性和远程升级可靠性。

### 6.9 电压电流与健康监测

电压采样：

- 使用 2 路 CoreSPI 控制 6 片 AD7490。
- 每片 AD7490 16 通道。
- 覆盖 BBPS、BBPKA、SCP、PLP0、PLP1、BMU、TRX、SSD、PLL、OCXO 等电源轨。

电流/过流监控：

- PL 侧寄存器提供异常状态、PG、采样值。
- M3 侧 `ad_monitor_task()` 周期读取异常寄存器。
- 记录时间戳、PG、电压值、电流换算值。
- 支持阈值读取/设置、异常状态查询和打印上报。
- INA3221 I2C 电流采样驱动也存在，CoreI2C 初始化当前在 `main.c` 中被注释，实际是否启用需结合硬件和构建确认。

技术含金量：

- 不是单路 ADC 读取，而是多片 ADC、多模块电源轨统一监控。
- M3 与 PL 协同处理过流状态、锁存、清除和上报。
- 健康监测和遥测命令体系打通，故障可以远程查询。

### 6.10 PLL 与时钟芯片控制

相关路径：

- `User/User_Driver/PLL/pll.c`
- `ad9528.c`
- `ad9528_1.c`
- `cdcm6208.c`

能力：

- 通过 CoreSPI 控制 AD9528、AD9528_1。
- CDCM6208 驱动保留，主初始化中当前未启用配置。
- 支持 PLL 寄存器读写、配置下发、配置打印。
- 启动时延时后依次配置 AD9528_1 和 AD9528。

技术含金量：

- BMU 负责板级关键时钟树初始化和可调试控制。
- PLL 配置关系到下游模块通信、采样或高速接口稳定性。
- 提供 shell 读写能力，便于现场定位时钟配置问题。

### 6.11 PL 寄存器协同

`pl_read_reg()` / `pl_write_reg()` 是 M3 与 PL 之间最核心的控制接口。

PL 寄存器承载的功能包括：

- 槽位号、PL 版本。
- Flash 选择、Flash 轮转、启动区选择。
- MRAM 更新触发。
- 主备状态。
- TMR 开关、强制模式、校验组状态。
- CRC 硬件任务启动与结果。
- 电源、PG、电流异常、采样值。
- 系统复位、寄存器读写控制。

技术含金量：

- 这是典型 MCU + FPGA 协同设计，M3 侧负责协议和控制，PL 侧负责寄存器、硬件状态和部分计算/保护逻辑。
- 大量业务不是纯软件实现，而是跨软硬件边界的状态机。
- 调试时既要理解 C 代码，也要理解 PL 寄存器协议。

### 6.12 主备 BMU 冗余切换

相关路径：

- `User/User_Driver/MasterBackupSwitch/MasterBackupSwitch.c`
- `User/User_Driver/MasterBackupSwitch/MasterBackupSwitch.h`

主要机制：

- 主状态：`0x55`
- 备状态：`0xAA`
- GPIO5/6/15/16 用于本端/对端状态和配置握手。
- BMU 间 UART 帧同步参数和切换请求/应答。
- PL/MRAM 寄存器保存主备标志、Flash 选择、三模状态。
- 上电初始化与运行时切换是不同场景。
- 切换动作会更新 PL 寄存器、GPIO 输出、CAN 处理策略和 AD9528 相关动作。

技术含金量：

- 主备切换不是简单拉 GPIO，而是完整状态机。
- 要处理本端 MRAM 状态、对端状态、槽位优先级、运行时强制切换、参数同步和冲突场景。
- 切换会影响通信策略、上电控制、Flash/MRAM 状态和 PL 侧模式。

### 6.13 TMR/三模控制

相关路径：

- `User/APP/tmr/tmr.c`

能力：

- 打开/关闭三模。
- 强制打开 SCP、BBPS、BBPKA、PLP0、PLP1 等模块。
- 清除不可恢复校验组标志。
- 根据 Flash robin 状态清理对应组。
- 三模状态写入 MRAM，保证重启后恢复。
- 提供状态、结果、监控打印。

技术含金量：

- TMR 与 MRAM、Flash 轮转、PL 寄存器、主备状态有关，是可靠性控制链路的一部分。
- 软件负责状态管理和控制触发，PL 负责实际三模/校验组逻辑。

### 6.14 自检、产测与现场调试

相关路径：

- `User/Self_Test`
- `User/User_Driver/shell`
- `User/APP/Menu`
- `User/service`

能力：

- Flash 测试。
- CAN 测试。
- PLL 测试。
- 电压、电流测试。
- 上下电测试。
- PL 寄存器读写。
- PLL 寄存器读写。
- Flash 目录读写。
- 网络地址配置/显示。
- AD 监控打印。
- 版本显示。

技术含金量：

- 工程不仅有运行态业务，也有现场维护与产测入口。
- 对复杂板卡来说，调试命令、寄存器读写、状态打印是定位问题的重要工具。
- 自检/产测覆盖通信、Flash、电源、时钟等关键路径。

## 7. 典型数据流

### 7.1 CAN 遥控命令

1. `Fdd_RemCT_Can()` 轮询 CAN A/B。
2. 通过统一 `can_recv_msg()` 从 MSS CAN 或 CoreCAN 接收。
3. `ycyk_can_task()` 做多帧接收、拼包、校验和协议解析。
4. `ycyk_process_can()` 按命令码分发。
5. BMU 本地可处理的命令直接执行，如状态、电压、CRC、TMR、Flash、主备、复位、寄存器读写。
6. BMU 不处理的命令封装后转发到 CPU/PLP0。
7. 结果通过 CAN 或其他链路回传。

### 7.2 RS422/以太网文件升级

1. RS422 FIFO 或 MAC RX 队列收到完整业务帧。
2. 校验帧头、长度、APID、IRCode、checksum。
3. 文件传输开始命令解析文件类型、槽位、区号、主备属性。
4. 查 Flash 映射，确定 SPI 控制器、片选、起始地址和分区。
5. 若目标为 BMU，本地擦除并写 Flash。
6. 若目标为 CPU/其他模块，通过统一网络封装透传。
7. 文件传输结束后返回重构状态或触发后续 IAP/CRC。

### 7.3 DirectC/JTAG 互助重构

1. CAN 收到 JTAG 互助重构命令。
2. BMU 设置 DirectC action code。
3. DirectC 通过适配层访问 JTAG TMS/TDI/TDO。
4. 镜像数据从外部 Flash 分页读取。
5. 执行目标器件 ID、擦除、编程、校验等流程。
6. 记录执行结果和错误码。
7. 通过遥测遥控通道回传状态。

### 7.4 AD 异常监控

1. `ad_monitor_task()` 周期运行。
2. 读取 PL 侧异常状态寄存器。
3. 异常存在时读取 PG、电压值、电流换算值。
4. 写 PL 寄存器清除/确认异常锁存。
5. 更新本地异常状态。
6. 后续通过 CAN/遥测查询或调试打印输出。

### 7.5 CRC 校验

1. 上位命令启动 CRC。
2. BMU 判断文件类型。
3. BMU 分区：M3 分块读取 Flash 并计算 CRC32。
4. 其他模块分区：M3 配置 PL 侧 CRC 寄存器，启动硬件 CRC。
5. 主循环持续运行 CRC 任务或查询 PL 状态。
6. 结果通过命令查询返回。

## 8. 技术含金量汇总

### 8.1 异构 SoC 协同

SmartFusion2 同时包含 Cortex-M3 MSS 和 FPGA fabric。工程不是只用 M3 外设，而是大量使用 fabric CoreSPI/CoreUART/CoreCAN/CoreI2C/CoreGPIO，并通过 PL 寄存器完成跨软硬件协同。

价值点：

- 理解 MCU 与 FPGA fabric 地址映射。
- 理解 FIC 总线访问 fabric 外设。
- 软件状态机和 PL 寄存器协议配合。
- 控制类业务与硬件保护/计算逻辑协同。

### 8.2 多链路协议网关

CAN、RS422、Ethernet、UART 同时接入同一业务体系，BMU 能本地处理，也能透传给下游 CPU/PLP。

价值点：

- 多物理链路统一协议分发。
- 分片/拼包/校验/异常统计。
- 通信链路冗余和恢复。
- 低带宽总线和高速以太网并存。

### 8.3 在线升级与镜像管理

工程支持 BMU 自身 IAP、其他模块镜像透传、多分区 Flash 映射、CRC 校验、golden/update/iap 等分区语义。

价值点：

- 远程维护能力。
- 多镜像版本管理。
- 失败恢复基础。
- 升级过程与实时通信并发。

### 8.4 DirectC/JTAG 远程重构

BMU 可以作为嵌入式 JTAG Host，对其他器件进行在线编程或重构。

价值点：

- 跨器件维护能力。
- 厂商编程组件移植适配。
- Flash 分页读取镜像。
- JTAG 时序抽象。
- 远程触发和状态诊断。

### 8.5 高可靠控制

主备 BMU、MRAM 状态持久化、TMR/三模、CRC、Flash 轮转、电压电流异常监控共同构成可靠性设计。

价值点：

- 异常复位后状态恢复。
- 主备冲突与运行时切换处理。
- 镜像完整性校验。
- 电源异常检测和上报。
- TMR 状态保持。

### 8.6 裸机工程能力

工程采用裸机超级循环，任务包括通信、Flash、CRC、AD、TMR、主备、菜单、复位等。

价值点：

- 协作式调度。
- 非阻塞状态机。
- Timer 中断时间基准。
- 任务切片和后台化。
- 复杂业务下的主循环响应控制。

## 9. 当前工程问题与可优化点

### 9.1 文档和编码问题

- 部分源码注释存在编码混用，中文显示为乱码。
- 建议统一 UTF-8 或确认原始代码页后批量转换。
- 协议命令、PL 寄存器、Flash 分区、主备状态建议形成正式表格文档。

### 9.2 裸机实时性可观测性不足

现状：

- 主循环任务很多，但目前没有系统化的任务耗时统计。
- 部分任务如 Flash、CRC、AD、网络包处理可能影响通信响应。

建议：

- 为每个主循环任务增加 `min/max/avg` 执行时间统计。
- 统计主循环周期、最大抖动、超时次数。
- 对低频长任务设置单轮执行预算。
- 将统计数据接入 shell 或遥测。

可观测指标：

- 主循环平均周期和最大周期。
- 单任务最大耗时。
- 通信最大响应延迟。
- Flash/CRC 期间通信丢帧率。

### 9.3 RS422/UART 接收可靠性

现状：

- 已有 FIFO 和状态机，但异常帧后的重同步策略可以继续增强。

建议：

- 增加 FIFO 高水位、溢出计数、最大连续接收字节数。
- 将头错误后的整体 reset 优化为滑动重同步。
- 给帧解析增加单轮最大处理字节数，防止大包长时间占用主循环。
- 做满速压测和异常字节插入测试。

### 9.4 Flash/CRC 后台化

现状：

- CRC 已经有分块状态机。
- Flash 擦除已有任务入口。
- Flash 写入仍有进一步后台化空间。

建议：

- CRC 分块大小可配置，测试 512B/1KB/2KB/4KB。
- Flash 写入改造成后台状态机。
- 增加进度、错误码、超时状态回传。
- 升级期间持续响应 CAN/RS422/ETH。

### 9.5 DirectC/JTAG 可观测性

现状：

- DirectC 流程已有接入，但分阶段耗时和错误诊断可以加强。

建议：

- 统计 DirectC 阶段耗时：读镜像、IDCODE、擦除、编程、校验。
- 统计 Flash page 读取次数。
- 调整 page buffer size 做耗时对比。
- 将 DirectC 返回码映射为可读诊断。

### 9.6 Ethernet 队列与背压

现状：

- 已有 RX 队列和 TX buffer 状态。

建议：

- 增加 RX/TX 队列深度峰值统计。
- 队列满时记录丢包原因。
- TX 侧增加 pending 队列或重试策略。
- 压测 UDP 突发包下队列峰值和丢包率。

### 9.7 CAN 多帧 FSM 健壮性

建议：

- 增加多帧接收超时，避免 FSM 卡在半包。
- 重复帧只更新 bitmap，不重复增加接收计数。
- 增加乱序、漏帧、重复帧专项测试。
- 限制补帧最大次数，避免异常流量拖慢主循环。

### 9.8 外设启用状态需确认

需要结合硬件设计和编译配置确认：

- 具体 SmartFusion2 器件型号。
- `sf2_core_i2c_init()` 当前在 `main.c` 被注释，INA3221/PMBus 是否量产启用不确定。
- CDCM6208 驱动存在，但启动配置当前注释，可能是历史方案或备用芯片。
- DirectC 支持多代器件算法，实际目标器件和镜像格式应以硬件工程/配置文件为准。

## 10. 推荐补充的工程化文档

后续如果继续完善文档，建议分成以下几份：

- `PL寄存器地图.md`：寄存器地址、读写方向、位定义、关联模块。
- `Flash分区与镜像映射.md`：文件类型、slot、region、CS、起始地址、长度、用途。
- `遥测遥控命令表.md`：CAN/422/ETH 命令码、参数、响应、错误码。
- `主备切换状态机.md`：状态、触发条件、GPIO、MRAM、PL 寄存器、异常场景。
- `升级与IAP流程.md`：文件传输、Flash 擦写、CRC、IAP、失败恢复。
- `DirectC_JTAG互助重构流程.md`：触发命令、镜像来源、JTAG 接口、错误码、耗时统计。
- `裸机任务调度与耗时统计.md`：主循环任务、周期、最大耗时、优化数据。

## 11. 快速索引

| 方向 | 主要路径 |
| --- | --- |
| 入口与初始化 | `main.c` |
| 主循环调度 | `User/APP/RemCT_StateMachine/RemCT_StateMachine.c` |
| CAN 业务 | `User/APP/RemCT_Can/RemCT_Can.c` |
| CAN 驱动抽象 | `User/User_Driver/can` |
| RS422 业务 | `User/APP/RemCT_422` |
| UART FIFO | `User/APP/RemCT_422/Uart_Fifo_Frame.c` |
| Ethernet MAC | `User/APP/RemCT_ETH/eth/eth.c` |
| IPv6/UDP 封装 | `User/IpConfig/IP` |
| Flash 映射 | `User/IpConfig/Flash` |
| IAP | `User/APP/IAP/iap.c` |
| CRC | `User/APP/CRC/crc_task.c` |
| DirectC/JTAG | `User/component/DirectC` |
| AD7490 电压采样 | `User/User_Driver/AD7490` |
| AD 监控 | `User/APP/AD/ad_monitor.c` |
| INA3221 电流采样 | `User/User_Driver/INA3221` |
| PLL | `User/User_Driver/PLL` |
| PL 寄存器 | `User/User_Driver/pl` |
| 主备切换 | `User/User_Driver/MasterBackupSwitch` |
| TMR/三模 | `User/APP/tmr/tmr.c` |
| 自检/产测 | `User/Self_Test` |
| shell/菜单 | `User/User_Driver/shell`、`User/APP/Menu` |

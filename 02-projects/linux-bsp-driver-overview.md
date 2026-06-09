# Linux BSP / Driver 项目总览

## 定位

Linux 方向是当前第二证据线，核心不是“纯内核专家”，而是：

```text
围绕 PCIe、SPI、UART 和 UIO 的 Linux BSP/驱动接口适配、用户态控制层、状态管理、告警上报和系统集成维护。
```

其中 PCIe 是主要证据，SPI/UART/UIO 是 BSP 覆盖面和设备侧系统集成能力。

## PCIe 模块

### 当前已知事实

- 用户与同事共同编写、维护过 PCIe endpoint 相关驱动。
- 可公开参考的雏形代码路径：`/home/flamboy/linux-learning/labs/week-pcie/endpoint.c`。
- 该文件来自内部工程的搬运整理版本，不能等同于完整内部工程。
- 该驱动不是完全从 0 新写，而是在前人基础上持续改造维护形成的完整内核驱动；用户表示整个文件都理解，并在维护过程中逐步改过。
- 除内核侧驱动外，用户态也做了较多配套逻辑，包括初始化、整合、状态管理、处理、上报、告警、分发和维护。

### 当前可支撑的能力表达

- PCIe 设备驱动开发与维护
- BAR/MMIO 资源访问
- 内核态与用户态接口联调
- 用户态状态管理和告警上报
- PCIe 模块在设备系统中的初始化、维护和异常处理
- 在既有驱动基础上进行工程化改造、问题定位和长期维护

### 待确认边界

后续需要明确：

- `probe/remove`、BAR 资源记录、MMIO 访问、中断、MSI、ioctl、mmap 等分别改造维护到什么程度。
- 用户态状态管理和告警分发的模块边界。
- 典型问题定位案例：链路异常、BAR 访问异常、中断异常、状态上报异常等。

## SPI / UART 模块

### 当前已知事实

- SPI 和 UART 模块主要是在现有架构基础上开发和维护。
- 使用 Linux 内核驱动接口完成设备访问、功能实现和问题定位。
- 这部分不应作为最强主证据，但可以体现 BSP 覆盖面。

### 当前可支撑的能力表达

- Linux 外设接口使用和维护
- SPI/UART 通信链路开发与调试
- 基于现有驱动框架完成设备侧功能实现
- 外设通信异常定位

### 待确认边界

- SPI/UART 是否涉及内核驱动改动，还是主要通过用户态接口调用。
- 是否涉及设备树、驱动参数、ioctl、read/write、termios、spidev 等接口。
- 是否有自定义协议、状态机或异常恢复逻辑。

## UIO / 寄存器访问

### 当前已知事实

- 存在部分自定义 UIO 相关内容。
- 部分功能通过 UIO 直接调用寄存器实现。
- 内部代码暂无法提供。

### 当前可支撑的能力表达

- 理解用户态访问 MMIO 寄存器的基本方式。
- 能围绕寄存器状态完成设备控制、状态采集和功能集成。
- 具备内核接口与用户态业务逻辑结合的经验。

### 风险边界

- 不能在没有证据时描述为完整自研 UIO 框架。
- 面试中需要能解释为什么使用 UIO、暴露了哪些资源、如何避免越界访问和并发问题。

## 当前学习补强建议

优先级如下：

1. PCIe 驱动关键路径：设备匹配、BAR/MMIO、MSI/中断、ioctl/mmap、用户态访问。
2. 用户态控制层：初始化流程、状态机、告警上报、异常恢复。
3. UIO：资源暴露、mmap、寄存器访问边界、并发和权限风险。
4. SPI/UART：标准接口、配置路径、异常定位和协议处理。

## 个人学习证据

`/home/flamboy/linux-learning` 中存在较多个人 Linux 驱动学习实验，可作为补强证据线。当前已观察到的实验方向包括：

- 字符设备
- ioremap/MMIO
- 同步机制
- 事件/等待
- 中断
- 设备树/overlay
- PCIe endpoint 雏形驱动和测试应用

这些材料的作用不是替代工作项目，而是证明用户在系统补齐 Linux BSP/Driver 基础。

### 已有文档可复用范围

当前不需要从零重做 Linux 驱动学习。`/home/flamboy/linux-learning/docs` 已经覆盖一条较完整的学习主线：

- QEMU/Linux 启动与内核构建：`docs/week1-build-qemu-linux/`
- Kbuild 与外部模块：`docs/week2-kbuild/`
- 字符设备：`docs/week3-char-device/`
- ioctl/mmap：`docs/week4-ioctl-mmap/`
- 并发与同步：`docs/week5-concurrency-sync/`
- 事件、中断、bottom half：`docs/week6-event-driven-irq/`、`docs/week6.1-irq-bottom-half/`
- platform driver 与设备树：`docs/week7-platform-dt/`
- DTB/overlay 深挖：`docs/week7.5-virt-dtb-deep-dive/`
- resource/MMIO：`docs/week8-resource-mmio/`、`docs/week8.5-pl031-mmio-deep-dive/`
- DMA 接口层：`docs/week9-dma-interface/`
- PCIe 专项：`docs/week-pcie/`

后续重点不是重复实验，而是把已有材料转成面试可用的链路图、问题定位记录和口径边界。

### 复习优先级

1. PCIe 专项：`pcie_fpga_register_access_interview.md`、`week-pcie_endpoint_notes.md`、`week-pcie_pcie_access_chain_experiment.md`。
2. MMIO/资源路径：Week8 与 Week8.5。
3. platform driver/设备树：Week7 与 Week7.5。
4. ioctl/mmap/字符设备：Week3 与 Week4。
5. 中断/bottom half/同步：Week5、Week6、Week6.1。
6. DMA 接口层：Week9，重点理解地址、cache、ownership handoff。

## 当前最需要补的材料

1. `linux-pcie-ownership.md`：区分主写、共同维护、只了解的部分。
2. `linux-pcie-flow.md`：PCIe 从驱动加载到用户态读取状态的链路图。
3. `linux-uio-note.md`：为什么用 UIO、暴露什么寄存器、风险如何控制。
4. `linux-issue-record-01.md`：一个 PCIe 或外设通信问题定位记录。

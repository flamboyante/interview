# Linux 面试导向学习计划

## 定位

Linux 方向不是当前唯一主线，而是第二证据线。目标不是包装成纯 Linux 内核专家，而是补齐：

```text
Linux BSP / Driver / PCIe / MMIO / UIO / 用户态控制层
```

对外推荐表达：

```text
参与并维护 PCIe endpoint 内核驱动及用户态控制链路，熟悉 BAR/MMIO、中断/MSI、ioctl/mmap、状态管理和告警上报；同时通过系统化 Linux 驱动实验补齐字符设备、platform driver、设备树、MMIO、DMA、同步和中断机制。
```

## 总体策略

Linux 准备分三类：

1. 复习：已有 `/home/flamboy/linux-learning/docs` 不重做，按面试优先级回顾。
2. 包装：把工作项目和学习实验整理成可讲链路。
3. 新学：只补当前证据链缺口，不泛学内核。

## 第一阶段：PCIe 主证据打穿

目标：面试官问 PCIe 时，能从硬件枚举讲到用户态状态上报。

### 复习材料

- `/home/flamboy/linux-learning/docs/week-pcie/pcie_fpga_register_access_interview.md`
- `/home/flamboy/linux-learning/docs/week-pcie/week-pcie_endpoint_notes.md`
- `/home/flamboy/linux-learning/docs/week-pcie/week-pcie_pcie_access_chain_experiment.md`
- `/home/flamboy/linux-learning/labs/week-pcie/endpoint.c`

### 必须能讲清

- PCIe RC / EP / Switch / BDF 的关系。
- 设备上电、链路训练、配置空间、PCI core 枚举。
- `pci_driver` 匹配后 `probe()` 做什么。
- BAR 是什么，驱动如何拿到并映射。
- MMIO 访问和 DMA 访问的区别。
- MSI/中断如何申请、触发和处理。
- `ioctl` / `mmap` / UIO / VFIO 的适用边界。
- 用户态如何完成初始化、状态采集、告警上报和分发。

### 需要补的文档

- `02-projects/linux-pcie-flow.md`
  - 驱动加载 -> 设备匹配 -> BAR/MMIO -> MSI/中断 -> ioctl/mmap -> 用户态状态管理 -> 告警上报。
- `02-projects/linux-pcie-ownership.md`
  - 哪些是工作中改造维护，哪些是共同维护，哪些是个人补强理解。
- `02-projects/linux-issue-record-01.md`
  - 至少 1 个 PCIe 或用户态状态链路问题定位记录。

## 第二阶段：BSP 基础链路复习

目标：补齐 Linux BSP/Driver 常见基础，不被问到基础链路时卡住。

### 复习顺序

1. 字符设备、`file_operations`
   - `docs/week3-char-device/`
2. `ioctl` / `mmap`
   - `docs/week4-ioctl-mmap/`
3. platform driver / 设备树
   - `docs/week7-platform-dt/`
   - `docs/week7.5-virt-dtb-deep-dive/`
4. resource / MMIO / ioremap
   - `docs/week8-resource-mmio/`
   - `docs/week8.5-pl031-mmio-deep-dive/`
5. 中断 / bottom half / 同步
   - `docs/week5-concurrency-sync/`
   - `docs/week6-event-driven-irq/`
   - `docs/week6.1-irq-bottom-half/`
6. DMA 接口层
   - `docs/week9-dma-interface/`

### 必须能讲清

- `platform_device` 和 `platform_driver` 如何通过设备树匹配。
- `reg` 如何变成 `IORESOURCE_MEM`，再到 `ioremap`。
- `copy_from_user` / `copy_to_user` 为什么必要。
- `mmap` 映射 BAR 或内存时要注意什么。
- ISR 和 bottom half 的边界。
- `mutex`、`spinlock`、`atomic`、`waitqueue` 的使用场景。
- DMA 地址、物理地址、CPU 虚拟地址为什么不能混。

## 第三阶段：UIO / SPI / UART 覆盖面补强

目标：这些不是主证据，但要能支撑 BSP 覆盖面。

### UIO

需要能讲清：

- 为什么使用 UIO。
- UIO 暴露哪些资源。
- 用户态如何通过 `mmap` 访问寄存器。
- 并发、权限、越界访问和寄存器误写风险。

需要补的文档：

- `02-projects/linux-uio-note.md`

### SPI / UART

需要能讲清：

- SPI/UART 在工作中是基于现有架构开发维护。
- 使用标准内核接口、已有驱动封装或 UIO 访问寄存器分别是什么路径。
- SPI 关注片选、模式、时钟、读写时序。
- UART 关注波特率、帧格式、阻塞/非阻塞、缓冲和异常帧处理。

不建议作为最强卖点，但可以作为 Linux BSP 覆盖面。

## 第四阶段：新学补短板

只补以下缺口：

1. PCIe 中断/MSI 和用户态通知链路。
2. UIO 的安全边界和工程适用场景。
3. DMA API 的基本语义，尤其是 cache ownership handoff。
4. 设备树 + resource + ioremap 的完整调用链。
5. 一个可讲的问题定位记录。

暂不优先：

- 深入调度器。
- 深入内存管理全链路。
- 文件系统。
- 网络协议栈深挖。
- eBPF、容器、虚拟化等非当前主线内容。

## 两周执行计划

### 第 1 周：PCIe 链路复习与包装

输出：

- `linux-pcie-flow.md`
- `linux-pcie-ownership.md`

检查点：

- 能用 3 分钟讲清 PCIe 从上电到用户态读寄存器。
- 能讲清 `endpoint.c` 的主要职责和工程妥协。
- 能说清工作项目和个人实验的边界。

### 第 2 周：BSP 基础和 UIO 补强

输出：

- `linux-uio-note.md`
- `linux-issue-record-01.md`

检查点：

- 能讲清 platform driver + DT + resource + ioremap 链路。
- 能讲清 UIO 为什么用、怎么用、风险是什么。
- 至少准备 1 个真实或实验复现的问题定位案例。

## 面试表达边界

稳妥表达：

```text
工作中参与并维护 PCIe endpoint 内核驱动和用户态控制链路，能够讲清驱动加载、BAR/MMIO、中断/MSI、ioctl/mmap 和用户态状态管理流程；同时通过 QEMU/Linux 实验系统补齐 platform driver、设备树、MMIO、DMA、同步和中断基础。
```

不建议表达：

```text
从 0 独立完成完整 Linux PCIe 驱动栈。
精通 Linux 内核。
深入掌握全部 Linux 驱动模型。
```


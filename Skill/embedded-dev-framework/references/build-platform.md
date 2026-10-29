# 构建约定、故障兜底与平台差异

配合主文件 SKILL.md 使用。

## 构建约定与故障兜底

- C 标准 `-std=gnu11`；浮点按需 `-mfpu`/`-mfloat-abi`，不要默认开。
- Debug：`-O0 -g3`（本项目，调试时变量不被优化掉）；想要更紧凑可用 `-Og -g3`，但单步与变量查看都不如 `-O0`。Release：`-Os -g`；链接 `--specs=nano.specs -Wl,--gc-sections`。
- `HardFault_Handler` 里点灯/翻转一个引脚，别写空循环还不给信号。
- `configASSERT` 打开，失败时停在一个可观测的位置。
- 看门狗只在一个最高优先级任务里喂；喂狗任务自己挂死等于系统挂死，所以它只做"检查其他任务的计数器"。
- `printf` 重定向到串口时注意阻塞与线程安全：只在一个线程里打印。

### 平台差异与数据一致性

- **M0 / M0+**：没有位带，没有 `LDREX/STREX`；对齐的 32 位读写仍是原子的，临界区只有 PRIMASK 一种。
- **M3 / M4**：可用位带别名区做单比特原子操作（但优先用 BSRR）；`__LDREX/__STREX` 可以做无锁计数。
- **M7 / H7（有 D-Cache）**：DMA 与外设缓冲必须放在非 cache 区（MPU 配 `Device` 或 `Non-cacheable`），或者逐次维护：
  - 发送前：`SCB_CleanDCache_by_Addr(buf, len)`；
  - 接收后：`SCB_InvalidateDCache_by_Addr(buf, len)`；
  - 句柄表、常量表在 `.rodata`，不受影响；
  - 忘了维护的典型症状：DMA 收到的数据看起来是"上一包的"。
- **MPU**：把外设寄存器区、DMA 缓冲、栈区配成对应属性；栈溢出时立刻 MemManage，而不是静默踩坏别的变量。
- 这些差异**只允许出现在 BSP 与启动文件里**；上层代码不得出现 `SCB_*`、`__LDREX`、`__set_PRIMASK`。

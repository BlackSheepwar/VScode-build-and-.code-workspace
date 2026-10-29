# TASK、OS 与 CubeMX 生成物边界

配合主文件 SKILL.md 使用。

## TASK 与线程

- 线程由 **CubeMX 的 `.ioc`** 或平台层 `task_init.c` 创建；`TASK/` 里只放线程体。
- 一个任务一个 `.c`，**平铺**，不再分子目录；线程体不需要 `.h`（除非有第二个调用者）。
- 线程体结构固定：

```c
void Task_Key_Scan(void *argument)
{
    const uint32_t freq   = osKernelGetTickFreq();
    const uint32_t period = (freq * TASK_KEY_SCAN_PERIOD_MS) / 1000u;
    uint32_t       next   = osKernelGetTickCount();
    (void)argument;

    for (;;) {
        next += period;
        (void)osDelayUntil(next);   /* ← 只有这几行和 OS 有关 */
        Svc_Key_Logic_Tick((uint16_t)TASK_KEY_SCAN_PERIOD_MS);  /* ← 一步动作 */
    }
}
```

- **用 `osDelayUntil` 不用 `osDelay`**：相对延时会被抢占累积误差，绝对节拍钉在整数倍上。
- 换算 tick **先乘后除**：`(freq * period_ms) / 1000u`。写成 `freq / 1000u * period_ms`，在 tick 100 Hz/500 Hz 的配置上会先截断成 0，任务变成空转不延时。
- 优先级：扫描/采集类 `osPriorityNormal`，命令/日志类 `osPriorityBelowNormal`；不要给业务任务 `REALTIME`。
- **优先级数值受内核配置钳制**：FreeRTOS 的 `configMAX_PRIORITIES` 是上限，CMSIS-RTOS2 的 `osPriorityNormal` 是 24——`configMAX_PRIORITIES` 小于 25 时它会被钳到最高可用值，**不报错**。两种做法选一种：把 `configMAX_PRIORITIES` 设到 ≥ 56（CubeMX 默认就是），或者只用 `osPriorityLow/Low1/BelowNormal/Normal` 这几档并接受钳制。
- 判据：`grep configMAX_PRIORITIES Core/Inc/FreeRTOSConfig.h` 的值必须大于你用到的最大 `osPriority`，否则"高优先级"只是心理安慰。
- 线程栈、优先级、名字全部在 `.ioc` 里配；代码里的 `osThreadAttr_t` 只在"没有 CubeMX"时手写。

## OS 从哪来

| 情况 | OS 源码 | 谁创建线程 | `TASK/` 里的 OS 代码 |
|---|---|---|---|
| CubeMX 自带 FreeRTOS | `Middlewares/Third_Party/FreeRTOS`（只读） | `freertos.c` 的 `MX_FREERTOS_Init()` | 0 行，只有同名强定义覆盖 `__weak` |
| 自己移植 FreeRTOS | `User/Third_Party/FreeRTOS/` | `User/TASK/task_init.c` | 只有循环骨架 |
| 换 RT-Thread | `User/Third_Party/RTT/` | `rt_application_init()` 或 `task_init.c` | 只有循环骨架 |
| 裸机调度器 | `User/Common/Common_Sched/` | `Common_Sched_Start()` | 循环变成"注册周期回调" |

移植 FreeRTOS 的硬要求：

- 目录只放官方源码，不改一个字；唯一属于你的是 `FreeRTOSConfig.h` 与 `heap_N.c` 的选择。
- **`FreeRTOSConfig.h` 只能有一份**（CubeMX 的 `Core/Inc/` 或移植目录），两份时先找到哪份取决于 include 顺序。
- `port.c` 的 `vPortSVCHandler`/`xPortPendSVHandler`/`xPortSysTickHandler` 与启动文件弱符号是同一个符号：要么在 `FreeRTOSConfig.h` 里 `#define vPortSVCHandler SVC_Handler` 三连，要么在 `stm32f1xx_it.c` 里转发。两处都定义 = 重复定义。
- 移植版没有 CMSIS-RTOS2 包装：`osDelay(10)` → `vTaskDelay(pdMS_TO_TICKS(10))`。若希望 `TASK/` 一字不改，在 `Common/Common_Os/` 只包四个原语（延时、投递、接收、取 tick）；**只有一个 OS 时不要包**。

判据：把 `TASK/` 换到另一个 OS，只需改 `for (;;)` 那几行；要连带改 `Svc_*` 函数体，说明 OS 调用漏到了下层。

### CubeMX 生成物与手写代码的边界

- 生成物 = `Core/`、`Drivers/`、`Middlewares/`、根 `CMakeLists.txt`、`cmake/stm32cubemx/`。**只读**；手写代码只允许写在 `/* USER CODE BEGIN ... END */` 之间。
- `.ioc` 改完（加任务、加队列、开外设）必须**重新生成并 review diff**：`freertos.c` 的生成区（`KEY_EVT_QHandle`、`KEY_EVT_Q_attributes`、`osMessageQueueNew(...)` 调用）会被整体重写，只有 USER CODE 区能存活。
- **所有 OS 对象配在 `.ioc`**：线程、队列、信号量、互斥量都在 CubeMX 里配；BSP/Device/Service 里不允许 `osThreadNew` / `osMessageQueueNew` / `osSemaphoreNew` / `osMutexNew`，也不允许绕成 FreeRTOS 原生 `xTaskCreate` / `xQueueCreate` / `xSemaphoreCreateMutex` / `xSemaphoreCreateMutexStatic`（同一个错误换层皮，还绕开了 CubeMX 的堆/静态度量）。BSP 只 `extern` 句柄、绑定句柄或使用句柄。
- **初始化分两段**：不需要 OS 对象的初始化放 `USER CODE BEGIN Init`；需要信号量/互斥量的 `BSP_xxx_Init()` 放 `USER CODE BEGIN RTOS_SEMAPHORES`（生成对象之后、队列/线程之前），否则拿到的是 `NULL`。
- 生成句柄定义在 `freertos.c`，`main.h` 不会声明信号量；消费模块在自己的 `.c` 里 `extern` 即可，不要改生成头文件。
- **没有 CubeMX 时只有一个 OS 初始化文件**：`User/TASK/task_init.c` 的 `Task_Init()` 先 `osKernelInitialize()`，再统一创建全部队列/信号量/互斥量/线程；BSP 的 `Init()` 只绑定或接收句柄。判据：`grep -rn "osMutexNew\|osSemaphoreNew\|osMessageQueueNew\|osThreadNew" User/` 在 CubeMX 工程应为 0。
- 提交前跑一次：`git diff --stat Core cmake/`，出现的改动应当全部对应你这次在 `.ioc` 里做的事；出现"意外删除"说明有人动过生成区。
- 重新生成后立刻编译：生成区被覆盖的典型报错是 `undefined reference to KEY_EVT_QHandle`。

### `.ioc` 里加 OS 对象（可以手改，不必点界面）

`.ioc` 是纯文本键值表，手改键再让 CubeMX 生成，等价于在图形界面里配。**两处都要写**：`FREERTOS.IPParameters` 里加上这个键名，下面再写对应的 `FREERTOS.<键名>=...`。只写后者、漏了 IPParameters，CubeMX 会当这个对象不存在。

| 对象 | 键 | 值语法 |
|---|---|---|
| 任务 | `FREERTOS.Tasks01` | `名字,优先级,栈字,入口函数,As weak,NULL,Static,名字Buffer,名字ControlBlock` |
| 队列 | `FREERTOS.Queues01` | `名字,深度,元素类型,0,Static,名字Buffer,名字ControlBlock` |
| 互斥量 | `FREERTOS.Mutexes01` | `名字,Dynamic,NULL`（实测：写成 `名字,Static,名字ControlBlock` 时 CubeMX 依旧生成 `osMutexNew` 动态分配、控制块名不生效——互斥量只有动态一种，别指望静态） |
| 二值信号量 | `FREERTOS.BinarySemaphores01` | `名字,Static,名字ControlBlock,Available`；动态把中间换成 `Dynamic,NULL` |
| 计数信号量 | `FREERTOS.CountingSemaphores01` | `名字,上限,Static,名字ControlBlock,初始值` |

- 同类多个对象写在同一行，用 `;` 连接（对照 `Tasks01` 的写法）。`名字Buffer` / `名字ControlBlock` 是静态分配所需的存储区**变量名**，CubeMX 会在 `freertos.c` 顶部替你定义，不用自己写。
- 生成顺序是固定的：`USER CODE BEGIN Init` → 建互斥量/递归互斥量 → `USER CODE BEGIN RTOS_MUTEX` → 建队列 → `USER CODE BEGIN RTOS_QUEUES` → 建线程。所以：只用互斥量/信号量的 `BSP_xxx_Init()` 放 **`RTOS_MUTEX`**（放 `Init` 里拿到的是 `NULL`）；要用队列的放 **`RTOS_QUEUES`**。
- 生成后在 `freertos.c` 里得到 `osMutexId_t IIC_LOCKHandle;`；消费模块在自己的 `.c` 里 `extern osMutexId_t IIC_LOCKHandle;` 即可使用 `osMutexAcquire/osMutexRelease`——**超时参数是 tick**，用 `pdMS_TO_TICKS()` 换算，别把毫秒直接塞进去。
- 生成的互斥量是 `osMutexNew(&attr)`（属性里只有 `.name`）：非递归、带优先级继承（底层 `xSemaphoreCreateMutex`，要求 `configUSE_MUTEXES=1`）。要在中断里给信号用二值信号量，不要用互斥量。
- 键写错时 CubeMX 通常静默丢掉这个对象，症状就是编译期 `undefined reference to XXXHandle`：先回头看 `IPParameters`。
- 顺序纪律：发现缺 OS 对象 → 改 `.ioc` → **请用户用 CubeMX 重新生成**（或 `STM32CubeMX -q` 脚本模式）→ 再编译。不许在模块里先建一个顶着用。
- 无 GUI 重新生成（实测可用，本机 CubeMX 在 `D:\a3\CubeMX`）：写一个 ASCII 脚本文件，三行 `config load <绝对路径>.ioc` / `project generate` / `exit`，执行 `STM32CubeMX.exe -q <脚本>`。启动器进程会立刻退出，真正的 JVM 在后台继续生成：盯 `Core/Src/freertos.c` 的修改时间等它落地（约 10~30 s），别把"启动器退出"当成"生成完成"。失败时看 `C:\Users\<用户>\.stm32cubemx\STM32CubeMX.log` 末尾。生成仍会保留所有 `USER CODE` 段。

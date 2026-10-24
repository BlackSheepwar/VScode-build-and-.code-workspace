# TASK、OS 与 CubeMX 生成物边界

配合主文件 SKILL.md 使用。章内保留了原编号（§七、§十三）。

## 七、TASK 与线程

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

## 十三、OS 从哪来

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
- 所以队列与线程**二选一**：要么在 `.ioc` 里配（推荐），要么整段放进 USER CODE 区自己建。**不要手抄进生成区**，下次生成必丢。
- 提交前跑一次：`git diff --stat Core cmake/`，出现的改动应当全部对应你这次在 `.ioc` 里做的事；出现"意外删除"说明有人动过生成区。
- 重新生成后立刻编译：生成区被覆盖的典型报错是 `undefined reference to KEY_EVT_QHandle`。

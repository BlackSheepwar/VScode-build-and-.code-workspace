---
name: embedded-project-scaffold
description: 嵌入式 C 工程分层与命名规范：User/ 下 BSP/Device/Service/APP/TASK + Common/Third_Party，句柄化硬件、CMake 库边界卡越层、任务由 CubeMX 或平台层创建。
---

# 嵌入式项目分层与命名规范

适用：裸机 / FreeRTOS / RT-Thread 的 Cortex-M 类工程，构建用 CMake。
目标：**换 MCU 只改 BSP，换 OS 只改 TASK 的循环骨架，换需求只改 APP。**

## 零、三条总原则

1. **依赖单向**：`TASK → APP → Service → Device → BSP → HAL`。
   `Common` 与 `OS` 是横向层，谁都能用它们，它们不反向依赖任何人。
2. **句柄化**：一个硬件资源一个句柄，一个设备一个句柄；函数收句柄指针，不收裸参数。
3. **越层由编译期拦**：靠 CMake 的 `PRIVATE` / `PUBLIC`，不靠自觉。

### 东西该放哪（判据表）

| 东西 | 层 | 判据（一句话） |
|---|---|---|
| 读/写 GPIO 电平、I2C 一次事务、UART 搬运字节 | BSP | 只碰寄存器与 HAL，不知道接的是什么 |
| 灯条、按键、陀螺仪、OLED 面板 | Device | 有"这台设备"的概念，仍不知道业务 |
| 消抖、长按、滤波、帧格式、重发 | Service | 跨设备，或带时间轴、带流程 |
| 帧结构 + 解包状态机 | Service | 决定"这串字节算不算一帧" |
| CRC 计算、位操作、定点数学、错误码 | Common | 纯函数，不碰硬件也不碰时间 |
| 档位、模式、菜单、协议命令 | APP | 产品行为，Service 不该知道 |
| 线程体、周期、优先级 | TASK | 只回答"多久跑一次、跑哪个一步函数" |
| 别人写的库（ringfifo、字体、RTOS 源码） | Third_Party | 只读区，一个字都不改 |

## 一、目录树

```
User/
├── CMakeLists.txt              # 汇总：add_subdirectory 各层 + 挂到工程
├── Common/                     # 纯软件，可以拿到 PC 上编译
│   ├── Common_Error/
│   └── Common_Crc/
├── Third_Party/                # 第三方只读区，不写自己的逻辑
│   ├── RingFifo/
│   └── FreeRTOS/               # 移植的 RTOS（CubeMX 平台则为空）
├── BSP/
│   ├── BSP_GPIO/               # bsp_gpio.c / bsp_gpio.h
│   ├── BSP_UART/
│   └── BSP_Board/              # 仅无 CubeMX 平台：统筹各 Init 顺序
├── Device/
│   ├── Dev_Key/
│   ├── Dev_Led_Bar/
│   └── Dev_Gyro/
├── Service/
│   ├── Svc_Key_Logic/
│   └── Svc_Proto/
├── APP/
│   ├── APP_Level/
│   └── APP_Key_Cmd/
└── TASK/
    ├── task_key_scan.c
    └── task_key_cmd.c
```

`Core/`、`Drivers/`、`Middlewares/` 是 CubeMX 生成区：**只读**，唯一例外是 `/* USER CODE BEGIN/END */` 之间。

## 二、层职责

| 层 | 管什么 | 禁止什么 |
|---|---|---|
| BSP | 句柄定义、引脚/外设事务、单位换算成"电平/字节/微秒" | 禁止 include 任何 `Dev_`/`Svc_`/`APP_` 头；禁止业务判断 |
| Device | 设备句柄、设备能力（`Set`/`Read`/`Is_Pressed`） | 禁止 include Service/APP；禁止延时、重试、去抖 |
| Service | 时间轴、流程、跨设备组合 | 禁止 include APP/TASK；禁止知道引脚号 |
| APP | 业务状态与命令表 | 禁止建线程、禁止直接操作寄存器 |
| TASK | 线程体、周期、优先级、队列接收 | 禁止业务逻辑（≤5 行 OS 调用 + 调下层） |
| Common | 纯函数 | 禁止 include HAL 与任何上层 |
| Third_Party | 别人的代码 | 禁止在里面写业务 |

**允许跳级**：APP 可以直接调 Device（比如"满档"就是 `Dev_Led_Bar_Set()`），Service 不是必经之路。
**禁止反向**：Device 调 Service、BSP 调 Device、任何层调 `Task_*`。

## 三、句柄模型

三类句柄，命名必须能看出是哪类：

| 层 | 句柄 | 谁实例化 | 例子 |
|---|---|---|---|
| BSP | `bsp_gpio_t`（一个引脚） | BSP 模块自己的 `.c`（引脚号只写这一处） | `bsp_gpio_pa0`、`bsp_gpio_pb1` |
| Device | `dev_key_t` / `dev_led_bar_t`（一台设备） | Device 模块自己的 `.c`，只引用 BSP 句柄 | `dev_key_inc`、`dev_led_bar` |
| Service | `svc_*_t`（一份状态，不必单例） | 用它的 APP 或 TASK | `svc_proto_t` |

硬规则：

- **引脚号只出现在 BSP 的 `.c` 里**：用到的引脚在 `bsp_gpio.c` 里一引脚一行，Device 只引用句柄、不写端口号。换板子时只改 BSP 一个文件，设备与业务一行不动。
- 设备句柄只存"下层句柄指针 + 差异量（极性、地址）"，**不存端口号**。
- 同一种设备可多份：加第二台灯条 = 新增一组引脚句柄 + 一个 `Dev_Led_Bar_Set()`；不改任何函数。
- 句柄默认 `const`，实例化后不可变，编译期进 `.rodata`，不占 RAM。
- 函数传 `const bsp_gpio_t *`；指针为 `NULL` 要能安全返回。
- **BSP 头文件不许 include 芯片厂商头**（`stm32f1xx_hal.h`、`esp_gpio.h`、`nrf_gpio.h`）。句柄里需要硬件类型时用 `void *`（或 `uintptr_t`），`(GPIO_TypeDef *)` 这类转换只出现在 BSP 的 `.c` 里。
  - 理由：HAL 的 `GPIO_TypeDef` 是匿名 struct 的 typedef，没法前置声明；一旦写进头文件，厂商头就会顺着 `bsp_gpio.h → dev_led_bar.h → APP` 一路传染，最后"上层不碰 HAL"只剩口号。
  - 判据（可执行）：只给 `User/` 下的头目录、**不给** HAL 路径，`app_level.c`、`dev_led_bar.c` 仍能用 `arm-none-eabi-gcc -M` 通过；只有 BSP 的 `.c` 与用 OS 原语的 Service 会失败。

## 四、初始化顺序与所有权

**唯一一条初始化链，谁都不许自己另起一条：**

```
BSP_Board_Init()        ← 无 CubeMX 平台才需要；有 CubeMX 时引脚由 MX_GPIO_Init() 配
        ↓
Dev_Key_Init()、Dev_Led_Bar_Init()、Dev_Gyro_Init()      ← 设备自检/上电时序
        ↓
Svc_Key_Logic_Init()、Svc_Proto_Init()                   ← 服务状态与队列
        ↓
APP_Level_Init()、APP_Key_Cmd 的表（静态数据，无需 Init）
        ↓
创建线程（CubeMX 的 freertos.c / 平台 task_init.c）
```

- 这条链写在**一个地方**：有 CubeMX 时是 `Core/Src/freertos.c` 的 `USER CODE BEGIN Init`（在 `MX_FREERTOS_Init()` 最前面，早于建队列与建线程）；没有时是 `User/TASK/task_init.c` 的 `Task_Init()`。
- 有 CubeMX 时 `Dev_*_Init()` 通常退化为空：引脚模式已由 `MX_GPIO_Init()` 配好，设备层没有要初始化的东西；只有需要上电时序、自检、复位芯片（陀螺仪、OLED）的设备才真要写 `Init`。
- 所有初始化必须在**第一个线程启动前**完成；线程里不许再初始化全局状态。
- `BSP_Board_Init()` 只调 BSP，**不许**调 Device —— 依赖方向不允许。
- 初始化函数返回 `int`，0 成功、负值失败；失败要有一个地方停下来（点灯/断言），不要静默继续。

## 五、中断与并发契约

1. **中断里只允许调 BSP 的"搬字节"函数**（push 一个字节、置一个标志）。禁止在 ISR 里调 Service/APP、禁止 `malloc`、禁止日志输出。
2. ISR 与任务共享的变量必须 `volatile`；读写宽度 ≤ 4 字节（Cortex-M 对齐的 32 位读写是原子的），超过 4 字节要临界区。
3. 环形缓冲默认**单生产者单消费者**：ISR 里只 push，任务里只 pop；多写者要另外加临界区，不能假设 ringfifo 自带。
4. 临界区只保护"几条指令"：关中断期间不许调用可能阻塞的函数，也不许等外设。
5. RTOS 里中断优先级不能高于 `configMAX_SYSCALL_INTERRUPT_PRIORITY`，否则 ISR 里调 `FromISR` 版本会挂死。
6. 服务里的状态变量默认只被"一个线程"访问；要跨线程就明确写注释，由队列或临界区保护。
7. **可变全局量禁止跨模块写**：`extern` 出去的变量只许读；要共享状态就走队列、函数接口，或者留在模块 `static` 里由它自己的函数改。唯一例外是初始化阶段（单线程、启动前）的写。
   判据：`grep -n "extern" User/` 出来的每个变量，除句柄与常量表外都不该有第二次写入。

## 六、时间基准、栈与堆

**时间基准**（每层只用一种）：

| 层 | 用什么 |
|---|---|
| BSP / Device | `HAL_GetTick()`（超时、重试） |
| Service | **只收 `elapsed_ms` 参数**，自己不看时钟 |
| TASK | OS 节拍：`osKernelGetTickCount()` / `osDelayUntil()` |
| 裸机 | 一个 1 ms 的 `SysTick` 计数，别和 `HAL_GetTick()` 各算各的 |

**栈与堆**：

- 每个线程栈从 **1024 字节（256 字）起步**；跑起来用 `uxTaskGetStackHighWaterMark` 实测，留 30% 余量再收紧。128 字只能给"只调两个函数"的线程。
- 动态分配集中在初始化阶段；ISR、Service、循环里禁止 `malloc`。
- 静态优先：`xTaskCreateStatic` / `osThreadNew` + `StaticTask_t`，避免堆碎片。
- 禁止在中断里用 `printf`/浮点格式化；需要日志就投递到队列，由低优先级线程打印。

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

## 八、Service 判据（四问）

四个问题里**有一个"是"才建 Service**，否则 APP 直接调 Device：

1. 要时间轴吗？（闪烁、呼吸、连发、超时重发）
2. 有第二个使用者吗？（两个 APP 模块都要按键事件）
3. 是与产品无关的策略吗？（去抖、CRC、协议解析）
4. 能脱离硬件与线程单独测吗？

四个都"否" → 别建 Service，直接 `Dev_*`。
判例：8 颗灯"显示档位"是 APP 直接调 `Dev_Led_Bar_Set()`；将来要做"流水灯"才建 `Svc_Led_Effect`。

## 九、APP 判据

- APP 只做两件事：**保存业务状态**、**把命令映射成动作**。
- 命令映射用**配置表**，不用 `switch` 套 `switch`：

```c
typedef struct {
    uint8_t  key_id;
    uint8_t  event;
    void   (*handler)(void);
} app_cmd_entry_t;
```

- 模块名说清楚它在管什么业务：`APP_Level`（档位）、`APP_Screen`（界面）、`APP_Mode`（模式）。
  不要叫 `APP_Main`（没有 `main()`，名字无意义）、不要叫 `APP_App`。
- APP 不许 include `FreeRTOS.h` / `cmsis_os2.h`。

## 十、Common 与 Third_Party

- `Common/`：**自己写的纯软件**（错误码、CRC、位操作、定点数学、环形缓冲）。不许 include HAL、不许 include 上层。可以在 PC 上编译并单测。
- `Third_Party/`：**别人写的代码**，只读。不改造、不格式化、不在里面加业务。
- 用法：第三方库 → 配一个**适配器**放在需要它的层。
  - ringfifo（纯数据结构）→ 直接在 BSP/Service 里调；
  - 协议栈（nanomodbus）→ `Service/Svc_Modbus/` 适配传输与超时；
  - OLED 面板驱动 → `Device/Dev_Oled/` 适配，驱动里唯一的硬件调用（`HAL_I2C_Master_Transmit`）改成 `Dev_Oled_Write()`。
- 第三方库的 CMake 必须 `-w`，否则 `-Werror` 会把别人的警告变成你的错误。
- 第三方目标用 `PRIVATE` 链接到适配层，别让别人的类型泄漏到上层头文件。
- 每个第三方目录写一个 `README.md`：来源 URL、版本、本地改了什么。

### 测试与 CI 的位置

- `tests/` 放**仓库根**，不进 `User/`（`User/` 只放手写固件代码）：
  - `tests/common/`：Common 单测，宿主 gcc 直接编；
  - `tests/service/`：Service 单测，用 `tests/shim/` 顶替 OS 与 HAL；
  - `tests/shim/`：`cmsis_os2.h`、`stm32f1xx_hal.h` 的假实现（只提供用到的类型与函数）。
- 测试与固件编**同一份 `.c`**，不复制源码；改固件必须让测试跟着编译失败，才会有人去改。
- CI 两步就够：`cmake --build build/Debug`（固件必须过 `-Werror`）+ 宿主编译并运行 `tests/`。
- 固件里不放测试代码；`#ifdef HOST_TEST` 只允许出现在 shim 里，不允许污染 `User/`。

## 十一、命名

| 对象 | 规则 | 例子 |
|---|---|---|
| 层目录 | 多词缩写全大写；单词只首字母大写 | `BSP` `APP` `OS` `TASK` `Dev` `Svc` `Common`（层目录仍写全称 `Device` `Service`） |
| 模块目录 | 层前缀 + `_` + PascalCase | `BSP/BSP_GPIO`、`Device/Dev_Key`、`Service/Svc_Key_Logic` |
| 文件 | 模块目录名全小写 | `dev_key.c`、`svc_key_logic.c` |
| 扩展名 | 永远小写 | `.c` `.h`（`.C` 会被当 C++） |
| 公共函数 | 模块目录名原样 + PascalCase 动作 | `BSP_GPIO_Write`、`Dev_Led_Bar_Set`、`Svc_Key_Logic_Tick` |
| 内部函数 | `static` + 同上规则 | `Svc_Key_Logic_Send_Event` |
| 类型 | `snake_case_t` | `dev_led_bar_t`、`key_event_t` |
| 宏 | 全大写 + 模块前缀 | `SVC_KEY_LONG_PRESS_MS` |
| 保护宏 | `__模块目录名_H__` | `__Dev_Key_H__` |
| 全局实例 | **不带 `g_`**（本项目选择，非通例） | `dev_key_inc`、`bsp_gpio_pa0` |
| 文件内静态量 | `s_` | `s_level` |

> `g_` 前缀是团队偏好，不是嵌入式通例。换工程时统一改一处（本规范选择不带）。

## 十二、CMake

- 每层一个 `CMakeLists.txt`，每模块一个 `STATIC` 库；`PUBLIC` 传头目录，`PRIVATE` 藏下层。
- 例外：模块只有 1 个 `.c` 且只被 1 处使用时，可并入上层库，但要注释说明。
- 头文件里出现的类型，其所在库必须 `PUBLIC` 链接（否则上层看到 incomplete type）。
- `-Wall -Wextra -Werror` 只对手写代码；第三方 `-w`。

**`-w` 怎么才算真的压住 `-Werror`**：GCC 是"后出现的选项生效"，`-w` 必须排在 `-Werror` 之后。CMake 里 `CMAKE_C_FLAGS` 的选项排在目标选项之前，所以 `target_compile_options(<第三方目标> PRIVATE -w)` 一定在后、一定生效；但如果有人用 `add_compile_options(-Werror)` 加在手写库上并借用了 `INTERFACE` 传播，顺序就可能反过来。别猜，编译后去看生成的重排选项文件，确认 `-w` 在最后：

```powershell
Get-Content build/Debug/CMakeFiles/freertos.dir/flags.make
```

更稳的写法是只对第三方源文件关警告：

```cmake
set_source_files_properties(
    ${THIRD_PARTY_SRC} PROPERTIES COMPILE_OPTIONS "-w"
)
```

```cmake
add_library(dev_key STATIC Dev_Key/dev_key.c)
target_include_directories(dev_key PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/Dev_Key)
target_link_libraries(dev_key PUBLIC stm32cubemx PRIVATE bsp_gpio)
```

- 任务体用 **OBJECT 库**再 `target_sources(frame PRIVATE $<TARGET_OBJECTS:task_objects>)`：静态库里的强符号没人引用会被链接器丢掉，`__weak` 空实现反而赢。
- 根 `CMakeLists.txt` 若是 CubeMX 生成文件，不要改它；在 `User/CMakeLists.txt` 末尾加 `target_link_libraries(${CMAKE_PROJECT_NAME} <新库>)`（注意与生成文件一样用无关键字签名，混用会报错）。

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

## 十四、没有 CubeMX 的平台

- 每个 BSP 模块自带 `BSP_xxx_Init()`（时钟、模式、上下拉都在里面），句柄同时实例化。
- 由一个 BSP 层文件统筹顺序：`User/BSP/BSP_Board/bsp_board.c` 的 `BSP_Board_Init()` 按依赖顺序调各 `BSP_xxx_Init()`，对外只暴露这一个函数。
- 引脚号写在 BSP 模块自己的 `.c` 里，Device 只引用句柄（与 §三 一致）。
- 有 CubeMX 时**不建** `BSP_Board`：引脚由 `MX_GPIO_Init()` 配好，BSP 只做"句柄 → 端口"的翻译。

## 十五、构建约定与故障兜底

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

## 十六、快速开始（15 分钟）

1. `User/CMakeLists.txt` 建 `USER_WARN_FLAGS`，`add_subdirectory()` 各层。
2. `BSP/BSP_GPIO/`：句柄结构 + `Read/Write/Toggle`。
3. 在 `BSP/BSP_GPIO/bsp_gpio.c` 里加引脚句柄（引脚号写这里）；在 `Device/Dev_Led_Bar/dev_led_bar.c` 里实例化设备句柄、引用这些引脚。
4. `Service/Svc_Key_Logic/`：表驱动消抖，投递事件到队列。
5. `APP/APP_Level/`：业务状态 + `Step()`。
6. `APP/APP_Key_Cmd/`：命令表 `{key_id, event, handler}`。
7. `.ioc` 里配两个线程（扫描 10 ms、命令阻塞）+ 一个队列；生成 `freertos.c`。
8. `TASK/` 写两个线程体；`Core/Src/freertos.c` 的 `USER CODE BEGIN Init` 里串初始化（早于建队列与建线程）→ 编译。

## 十七、模块模板

```c
/* xxx.h */
#ifndef __Dev_Xxx_H__
#define __Dev_Xxx_H__

#include <stdint.h>
#include "bsp_gpio.h"

typedef struct {
    const bsp_gpio_t *pin;      /* 下层句柄 */
    uint8_t           active_low;
} dev_xxx_t;

extern dev_xxx_t dev_xxx;

int      Dev_Xxx_Init(void);
uint8_t  Dev_Xxx_Read(void);

#endif /* __Dev_Xxx_H__ */
```

```c
/* xxx.c */
#include <stddef.h>
#include "dev_xxx.h"

/* 引脚句柄由 BSP 提供，Device 一行端口号都不写 */
dev_xxx_t dev_xxx = { &bsp_gpio_pa0, 0u };

int Dev_Xxx_Init(void) { return 0; }
```

## 十八、反模式

- 在 `Device` 里写 `for (;;)` 或 `osDelay`（不是它的职责，也测不了）
- 在 `BSP` 里判断"档位"或按键长按
- 在 `Device`/`APP` 里写 `GPIOA`、`GPIO_PIN_x`（引脚号只许出现在 BSP 的 `.c` 里）
- `Service` 里 `#include "app_level.h"`（反向依赖）
- `APP` 直接 `#include "FreeRTOS.h"`
- `TASK` 里写业务（改需求要动任务文件）
- 一个 `.c` 里 800 行、既有协议又有业务
- 两份 `FreeRTOSConfig.h`
- 第三方代码里改 bug（下次升级全丢）

## 十九、自检清单

- [ ] `User/` 下只有 BSP / Device / Service / APP / TASK（`Common`、`Third_Party` 按需创建，可为空）
- [ ] 每个模块目录名是 `层前缀_模块`，文件名是它的小写
- [ ] 每个硬件都有 BSP 句柄，每台设备都有 Device 句柄
- [ ] 引脚号只出现在 BSP 的 `.c` 里，Device 不写端口号
- [ ] 设备句柄不存端口号，句柄都是 `const`
- [ ] 头文件里出现的类型，其库用 `PUBLIC` 链接
- [ ] 没有反向依赖（Device 调 Service、BSP 调 Device、下层调 `Task_*`）
- [ ] 初始化只有一条链，且在线程启动前完成
- [ ] ISR 里只调 BSP 搬字节，共享变量 `volatile` 且 ≤ 4 字节
- [ ] 线程用 `osDelayUntil`，栈 ≥ 256 字并实测水位
- [ ] `TASK/` 每个文件 OS 调用只在 `for (;;)` 骨架里
- [ ] 只有一个地方创建线程，一个 `.ioc`，一份 `FreeRTOSConfig.h`
- [ ] 第三方代码不在 `-Werror` 范围、有 README 记来源
- [ ] 第三方目标的编译选项里 `-w` 排在最后（查 `flags.make`，不靠猜）
- [ ] `.ioc` 改完重新生成过，`git diff --stat Core cmake/` 里没有意外删除
- [ ] 跨模块 `extern` 变量只读，除句柄与常量表外没有第二处写入
- [ ] 用到的最大 `osPriority` 小于 `FreeRTOSConfig.h` 里的 `configMAX_PRIORITIES`
- [ ] 有 D-Cache 的平台，DMA 缓冲做了 clean / invalidate，或放在非 cache 区
- [ ] （可选）`tests/` 在仓库根，CI 同时跑"固件构建"与"host 单测"；不上 CI 的小工程可跳过这条

## 二十、迁移旧工程

1. 先建层目录与空的 `CMakeLists.txt`，让工程能编译过（空库也要能 link）。
2. 从最底层往上搬：BSP → Device → Service → APP；每搬一层编译一次。
3. 句柄化放在搬 Device 时做，不要最后补。
4. 改名用脚本 + 一次全仓库 `grep` 校验；PowerShell 的 `-ne` 大小写不敏感，只改大小写时要用 `-cne`。
5. Windows 下改目录大小写必须两段式 `[System.IO.Directory]::Move`（先 `X_tmp` 再目标名），否则 `Remove-Item` 会误删。

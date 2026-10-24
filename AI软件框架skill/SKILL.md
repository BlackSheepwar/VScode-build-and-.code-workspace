---
name: embedded-project-scaffold
description: 嵌入式 C 工程分层与命名规范（完整版）：User/ 下 BSP/Device/Service/APP/TASK，硬件与设备都句柄化、引脚号只写 BSP、CMake 库边界卡越层、初始化唯一链、任务由 CubeMX 或平台层创建。主文件是判据与硬规则，细节拆在 references/（CMake 四行模板与 PUBLIC/PRIVATE、TASK 与 OS 边界、反模式清单、平台与构建差异）。
---

# 嵌入式项目分层与命名规范

适用：裸机 / FreeRTOS / RT-Thread 的 Cortex-M 类工程，构建用 CMake。
一句话：**换 MCU 只改 BSP，换 OS 只改 TASK 的循环骨架，换需求只改 APP。**

## 这份规范怎么用

主文件只放"每次写代码都要遵守"的判据与硬规则；细节按需查 `references/`，不必整套读完：

| 我要做什么 | 读哪个文件 |
|---|---|
| 建工程、加 `.c`/模块/层/任务、改链接、报"找不到头文件" | `references/cmake.md` — 三个 CMakeLists 的职责、四行模块模板、PUBLIC/PRIVATE 判据、TASK 的 OBJECT 库、验证手法 |
| 写线程体、换 RTOS、`.ioc` 重新生成之后 | `references/task-os.md` — TASK 与线程、OS 从哪来、CubeMX 生成物边界 |
| 提交前自查、code review | `references/checks.md` — 反模式与自检清单 |
| 配编译选项、查 HardFault、换芯片/Cache/MPU | `references/build-platform.md` — 构建约定、内存与故障配置、平台差异 |

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

## 七、Service 判据（四问）

四个问题里**有一个"是"才建 Service**，否则 APP 直接调 Device：

1. 要时间轴吗？（闪烁、呼吸、连发、超时重发）
2. 有第二个使用者吗？（两个 APP 模块都要按键事件）
3. 是与产品无关的策略吗？（去抖、CRC、协议解析）
4. 能脱离硬件与线程单独测吗？

四个都"否" → 别建 Service，直接 `Dev_*`。
判例：8 颗灯"显示档位"是 APP 直接调 `Dev_Led_Bar_Set()`；将来要做"流水灯"才建 `Svc_Led_Effect`。

## 八、APP 判据

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

## 九、Common 与 Third_Party

- `Common/`：**自己写的纯软件**（错误码、CRC、位操作、定点数学、环形缓冲）。不许 include HAL、不许 include 上层。可以在 PC 上编译并单测。
- `Third_Party/`：**别人写的代码**，只读。不改造、不格式化、不在里面加业务。
- 用法：第三方库 → 配一个**适配器**放在需要它的层。
  - ringfifo（纯数据结构）→ 直接在 BSP/Service 里调；
  - 协议栈（nanomodbus）→ `Service/Svc_Modbus/` 适配传输与超时；
  - OLED 面板驱动 → `Device/Dev_Oled/` 适配，驱动里唯一的硬件调用（`HAL_I2C_Master_Transmit`）改成 `Dev_Oled_Write()`。
- 第三方库的 CMake 必须 `-w`，否则 `-Werror` 会把别人的警告变成你的错误（细节见 `references/cmake.md` 的 §12.8）。
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

## 十、命名

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

## 十一、没有 CubeMX 的平台

- 每个 BSP 模块自带 `BSP_xxx_Init()`（时钟、模式、上下拉都在里面），句柄同时实例化。
- 由一个 BSP 层文件统筹顺序：`User/BSP/BSP_Board/bsp_board.c` 的 `BSP_Board_Init()` 按依赖顺序调各 `BSP_xxx_Init()`，对外只暴露这一个函数。
- 引脚号写在 BSP 模块自己的 `.c` 里，Device 只引用句柄（与 §三 一致）。
- 有 CubeMX 时**不建** `BSP_Board`：引脚由 `MX_GPIO_Init()` 配好，BSP 只做"句柄 → 端口"的翻译。

## 十二、快速开始（15 分钟）

1. `User/CMakeLists.txt`：`set(USER_WARN_FLAGS -Wall -Wextra -Werror)`（必须在 `add_subdirectory` 之前）→ 五层 `add_subdirectory()` → 末尾给主目标 `target_link_libraries`（plain 签名）。每层先用 `references/cmake.md` 的四行模板建第一个模块，让空库也能 link 通。
2. `BSP/BSP_GPIO/`：句柄结构 + `Read/Write/Toggle`。
3. 在 `BSP/BSP_GPIO/bsp_gpio.c` 里加引脚句柄（引脚号写这里）；在 `Device/Dev_Led_Bar/dev_led_bar.c` 里实例化设备句柄、引用这些引脚。
4. `Service/Svc_Key_Logic/`：表驱动消抖，投递事件到队列。
5. `APP/APP_Level/`：业务状态 + `Step()`。
6. `APP/APP_Key_Cmd/`：命令表 `{key_id, event, handler}`。
7. `.ioc` 里配两个线程（扫描 10 ms、命令阻塞）+ 一个队列；生成 `freertos.c`。
8. `TASK/` 写两个线程体；`Core/Src/freertos.c` 的 `USER CODE BEGIN Init` 里串初始化（早于建队列与建线程）→ 编译。

## 十三、模块模板

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

## 十四、迁移旧工程

1. 先建层目录与空的 `CMakeLists.txt`，让工程能编译过（空库也要能 link）。
2. 从最底层往上搬：BSP → Device → Service → APP；每搬一层编译一次。
3. 句柄化放在搬 Device 时做，不要最后补。
4. 改名用脚本 + 一次全仓库 `grep` 校验；PowerShell 的 `-ne` 大小写不敏感，只改大小写时要用 `-cne`。
5. Windows 下改目录大小写必须两段式 `[System.IO.Directory]::Move`（先 `X_tmp` 再目标名），否则 `Remove-Item` 会误删。
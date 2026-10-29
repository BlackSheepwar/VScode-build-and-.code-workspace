---
name: embedded-dev-framework
description: 嵌入式 C 开发框架唯一规范（架构 + 注释合一，与其他 C 规范冲突时以本技能为准）。架构：User/ 下 TASK/APP/Service/Device/BSP 分层、句柄与不透明类型（前置声明，减少下层向上层暴露）、层内单一 CMakeLists 与 PUBLIC/PRIVATE 边界、初始化唯一链、中断并发契约、命名规则。注释：RC_Dog 风格 Doxygen，文件头、80 列横幅、每个函数（含 static）的 @brief/@param/@retval 逐值枚举，.c 与 .h 各写一份。写或改 .c/.h、建工程/加模块、配 CMake、做 code review 时加载；细节按需查 references/。
---

# 嵌入式开发框架规范

适用：裸机 / FreeRTOS / RT-Thread 的 Cortex-M 类工程，构建用 CMake。

一句话：**换 MCU 只改 BSP，换 OS 只改 TASK 的循环骨架，换需求只改 APP。**

## 这份规范怎么用

主文件只放"每次写代码都要遵守"的判据与硬规则；细节按需查 `references/`，不必整套读完。

| 我要做什么 | 读哪个文件 |
|---|---|
| 建工程、加 `.c`/模块/层/任务、改链接、报"找不到头文件" | `references/cmake.md`：三个 CMakeLists 的职责、四行模块模板、PUBLIC/PRIVATE 判据、TASK 的 OBJECT 库、验证手法 |
| 写线程体、换 RTOS、`.ioc` 重新生成之后 | `references/task-os.md`：TASK 与线程、OS 从哪来、CubeMX 生成物边界、`.ioc` 里加 OS 对象 |
| 提交前自查、code review | `references/checks.md`：反模式清单（架构 + 注释）与自检清单 |
| 配编译选项、查 HardFault、换芯片 / Cache / MPU | `references/build-platform.md`：构建约定、内存与故障配置、平台差异 |

写注释不用查文件，本文件末尾「注释规范」一节已经是全部规则。

## 总原则

1. **依赖单向**：`TASK → APP → Service → Device → BSP → HAL`。
   `Common` 与 `OS` 是横向层，谁都能用它们，它们不反向依赖任何人。
2. **句柄化**：一个硬件资源一个句柄，一个设备一个句柄；函数收句柄指针，不收裸参数。
3. **越层由编译期拦**：靠 CMake 的 `PRIVATE` / `PUBLIC`，靠不透明类型，不靠自觉。

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

## 目录树

```
User/
├── CMakeLists.txt              # 汇总：一行一层 add_subdirectory + 挂到工程
├── Common/                     # 纯软件，可以拿到 PC 上编译
│   ├── CMakeLists.txt          # 本层唯一的 CMakeLists：本层所有模块库都写在这里
│   ├── Common_Error/           # 模块目录里只有 .c/.h，没有 CMakeLists.txt
│   └── Common_Crc/
├── Third_Party/                # 第三方只读区，不写自己的逻辑
│   ├── CMakeLists.txt
│   ├── RingFifo/
│   └── FreeRTOS/               # 移植的 RTOS（CubeMX 平台则为空）
├── BSP/
│   ├── CMakeLists.txt
│   ├── BSP_GPIO/               # bsp_gpio.c / bsp_gpio.h
│   ├── BSP_IIC/
│   ├── BSP_UART/
│   └── BSP_Board/              # 仅无 CubeMX 平台：统筹各 Init 顺序
├── Device/
│   ├── CMakeLists.txt
│   ├── Dev_Key/
│   ├── Dev_Led_Bar/
│   └── Dev_Gyro/
├── Service/
│   ├── CMakeLists.txt
│   ├── Svc_Key_Logic/
│   └── Svc_Proto/
├── APP/
│   ├── CMakeLists.txt
│   ├── APP_Level/
│   └── APP_Key_Cmd/
└── TASK/
    ├── CMakeLists.txt
    ├── task_key_scan.c
    └── task_key_cmd.c
```

`Core/`、`Drivers/`、`Middlewares/` 是 CubeMX 生成区：**只读**，唯一例外是 `/* USER CODE BEGIN/END */` 之间。

### 一条硬规则：一层一个 CMakeLists，模块不建自己的 CMakeLists

- `User/` 下**每个层目录只有一个 `CMakeLists.txt`**，本层的每个模块库（一个硬件、一台设备、一个服务、一个 APP 模块）都写成这个文件里的一段：一个 `add_library` + 四行模板。
- 模块目录（`BSP_GPIO/`、`Dev_Key/`、`Common_Ringbuf/`）里**只有 `.c` / `.h`**，不许出现 `CMakeLists.txt`，也不许 `add_subdirectory(<模块目录>)`。
- `add_subdirectory()` 只出现在 `User/CMakeLists.txt`，一行一层，它是"收集各层 CMake"的唯一位置。
- 理由：模块级 CMakeLists 会让"有哪些库、谁链谁"散落在目录树里，改链接要逐层翻目录；层内一个文件才能一眼读完本层的库边界，也让 `-Werror`、模块命名、PUBLIC/PRIVATE 这套判据只有一个落点。
- 判据（可执行）：`Get-ChildItem -Recurse -Filter CMakeLists.txt User` 只应列出**"顶层 1 条 + 每个层目录 1 条"**；出现 `User/BSP/BSP_GPIO/CMakeLists.txt` 就是违规。
- 加模块 = 建模块目录放 `.c/.h` → 在**本层** CMakeLists 里加一段四行模板 → 用它的层加链接 → 重新 Configure。**不新建 CMakeLists。**

## 层职责

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

## 句柄与不透明类型：减少下层向上层暴露

这一节是本规范的核心。**头文件只给名字，不给布局。** 上层能拿到指针、能取地址，但编译器不让它碰成员、不让它算大小。

### 三类句柄

| 层 | 句柄 | 谁实例化 | 例子 |
|---|---|---|---|
| BSP | `bsp_gpio_t` / `bsp_iic_t` | BSP 模块自己的 `.c`（引脚号只写这一处） | `bsp_gpio_pa0`、`bsp_iic1` |
| Device | `dev_key_t` / `dev_led_bar_t` | Device 模块自己的 `.c`，只引用 BSP 句柄 | `dev_key_inc`、`dev_led_bar_0` |
| Service | `svc_*_t`（一份状态，不必单例） | 用它的 APP 或 TASK | `svc_proto_t` |

**句柄纪律（五条）**

- **引脚号只出现在 BSP 的 `.c` 里**：用到的引脚在 `bsp_gpio.c` 里一引脚一行，Device 只引用句柄、不写 `GPIOA` / `GPIO_PIN_x`。换板子时只改 BSP 一个文件，设备与业务一行不动。
- 设备句柄只存"下层句柄指针 + 差异量（极性、地址）"，**不存端口号**。
- 同一种设备可多份：加第二台灯条 = 新增一组引脚句柄 + 一个 `Dev_LED_Bar_Set()` 调用，不改任何函数。
- 句柄默认 `const`，实例化后不可变，编译期进 `.rodata`，不占 RAM。
- 函数收 `const bsp_xxx_t *`；实参为 `NULL` 要能安全返回（设置函数直接返回，查询函数给保守值），或者显式断言并在注释里写明"不可为 NULL"。

### 头文件只给名字，布局留在 .c

```c
/* bsp_iic.h —— 头文件里只有"名字" */
#ifndef __BSP_IIC_H__
#define __BSP_IIC_H__

#include <stdint.h>              /* 只要 C 标准头，HAL 与 OS 一概不引 */

/* 前置声明：tag 在 .c 里补全，头文件不写布局 */
typedef struct bsp_iic bsp_iic_t;

/* 不完整类型可以 extern：只用于取地址 / 传指针 */
extern bsp_iic_t bsp_iic1;

int BSP_IIC_Write(bsp_iic_t *bus, uint8_t addr,
                  const uint8_t *data, uint16_t len);

#endif /* __BSP_IIC_H__ */
```

```c
/* bsp_iic.c —— 布局只在这里出现 */
#include "bsp_iic.h"
#include "stm32f1xx_hal.h"        /* HAL 只出现在 .c */
#include "cmsis_os2.h"            /* OS 类型只出现在 .c */

extern I2C_HandleTypeDef hi2c1;
extern osMutexId_t IIC_LOCKHandle;   /* 由 freertos.c 生成并定义 */

struct bsp_iic {                  /* ← 只是 tag，不再写 typedef */
    void             *hal;        /* HAL 句柄，类型不外泄 */
    osMutexId_t      *lock;       /* 存"句柄变量的地址"，不是句柄值 */
    volatile uint32_t waiter;
    volatile int32_t  result;
    uint8_t           initialized;
};

bsp_iic_t bsp_iic1 = {            /* 实例定义，一次配好且进 .rodata */
    .hal  = &hi2c1,
    .lock = &IIC_LOCKHandle,
};
```

### 这么写为什么合法（机制）

- C 的 struct tag 在**独立的名字空间**：`typedef struct bsp_iic bsp_iic_t;` 只是给一个"尚未完整的" tag 起别名。**typedef 记录的是类型引用，不复制布局。**
- `.c` 里补 `struct bsp_iic { ... };` 就补全了这个 tag，所有写 `bsp_iic_t` 的地方自动生效，头文件一个字都不用改。
- 因此上层编译时看到的是**不完整类型**，语言层面直接封死：

| 上层能做 | 上层不能做 | 报错 |
|---|---|---|
| `bsp_iic_t *p;` 声明指针 | `bsp_iic_t v;` 按值定义 | `storage size of 'v' isn't known` |
| `&bsp_iic1`、`func(&bsp_iic1)` | `sizeof(bsp_iic_t)` | `invalid use of incomplete type` |
| `extern` 声明该类型对象 | `p->initialized` 访问成员 | `invalid use of incomplete typedef` |

- 这就是"用编译期强制分层"：不是靠注释写一句"上层不要访问成员"，而是**写了也编不过**。

三条收益：

| 收益 | 具体表现 |
|---|---|
| 头文件零上层依赖 | `bsp_iic.h` 只 include `<stdint.h>`，`osMutexId_t`/`I2C_HandleTypeDef` 只出现在 `.c`。APP 链接 BSP 不会把 `stm32f1xx_hal.h` 拖进依赖树；换芯片时上层不用重编 |
| 布局可演进 | 加字段、换内部实现、换 HAL 版本，只要公共函数签名不变，上层源码一行不用改 |
| 强制走接口 | 上层想读状态只能要求下层加只读函数，不能伸手 |

判据（可执行）：`User/` 下（`Third_Party/` 除外）的 `.h` 里，`grep -rn "stm32f1xx_hal.h\|FreeRTOS.h\|cmsis_os2.h" User --include=*.h` 应为 0。

### 运行时创建的对象：存句柄地址，不存句柄值

场景：互斥量/信号量/队列由 CubeMX 在 `freertos.c` 里 `osMutexNew()` 创建，**句柄值在运行时才存在**，编译期拿不到，写不进静态初始化器。

但 `&IIC_LOCKHandle` 是**链接期常量**（句柄变量本身的地址是固定的），可以放进 `.rodata` 的初始化器。于是实例定义处一次配好，`Init()` 退化为校验：

```c
if (*bus->lock == NULL) {          /* 句柄还没被 osMutexNew 创建 */
    return BSP_IIC_ERR_STATE;
}
```

- 收益：不存在"半初始化"中间态；`Init()` 幂等（重复调用结果一样）；所有权清晰——句柄归 `freertos.c` 所有，BSP 只借地址，**不创建、不删除**。
- 代价：多一次解引用；`lock`（指针）与 `*lock`（句柄值）是两层，**两个都要判 NULL**。
- 反例：结构体里写 `osMutexId_t lock;`（按值）再在 `Init()` 里赋值——实例定义处是 `NULL`，谁先调谁先炸；两次 `Init()` 还会覆盖第一次的句柄。

### 什么时候只能 `void *`

- 厂商类型是**匿名 struct 的 typedef**，没有 tag 名 → 没法写 `struct XXX *`，也就无法前置声明。
  - 例：`typedef struct { ... } GPIO_TypeDef;`（`Drivers/CMSIS/Device/ST/STM32F1xx/Include/stm32f103xb.h`）
  - 判据：`grep -rn "typedef struct *{" Drivers/` 命中的类型都只能 `void *`
- 反过来，**有 tag 名**的 HAL 类型可以前置声明，白白拿回类型检查：
  - `typedef struct __I2C_HandleTypeDef { ... } I2C_HandleTypeDef;` → 头文件写 `struct __I2C_HandleTypeDef *hal;` 即可
  - 判据：`grep -n "typedef struct __" Drivers/` 命中的都能这么写
- `void *` 的代价是丢类型检查，收口办法：
  - 强转只出现在 BSP 的 `.c` 里，且集中在极少数函数（本项目是 `BSP_IIC_FindBus()` 名字→句柄、`BSP_IIC_StartHal()` `void *`→`I2C_HandleTypeDef *`）
  - 任何头文件里都不许出现 `(GPIO_TypeDef *)` / `(I2C_HandleTypeDef *)`

### 取成员 vs 提供只读接口

```c
/* 错：上层想算"档位数"就直接读成员 */
int16_t span = (int16_t)dev_led_bar_0.count + 1;
/* error: invalid use of incomplete typedef 'dev_led_bar_t' */
```

```c
/* 对：设备层提供只读函数，实现里才碰成员 */
uint8_t Dev_LED_Bar_Count(const dev_led_bar_t *bar);   /* return bar ? bar->count : 0u; */
```

- 判据：**上层需要的每个属性，都对应一个只读函数**；结构体布局永不出现在头文件。
- 收益：`count` 以后改成"由引脚数推算"或"运行时探测"，上层一行不用改——这才是接口稳定性。
- 顺带：`extern` 出去的变量只许读（见「中断与并发契约」）；结构体也不 `extern` 可写实例。

### 五类真实错误（都踩过）

1. `typedef bsp_gpio { ... };` —— 缺 `struct`，尾部还无别名
   → `error: expected '=', ',', ';', 'asm' or '__attribute__' before '{' token`
2. `typedef struct bsp_gpio { ... };` —— 尾部无别名
   → `warning: useless storage class specifier in empty declaration`（`-Werror` 下就是错误）
3. 以为 C 像 C++ 那样能裸写 tag 当类型名：`bsp_gpio x;` → `error: unknown type name 'bsp_gpio'`。
   同一份代码 `g++` 能过、`gcc` 不能——**C 的 tag 名不是类型名**。
4. 头文件 `extern dev_led_bar_t dev_led_bar_0;`，`.c` 里定义的是 `dev_led_bar` → 编译期不报，链接期 `undefined reference to 'dev_led_bar_0'`
5. 头文件为了拿 `GPIO_TypeDef` 而 `#include "stm32f1xx_hal.h"` → HAL 顺着 `bsp_gpio.h → dev_led_bar.h → APP` 一路传染，"上层不碰 HAL"只剩口号

### 例外：什么时候可以完整暴露

- 小型、稳定的**值类型**可以完整定义在头文件：`key_event_t`、错误码枚举、`app_cmd_entry_t` 这类配置表项。
  判据：布局会不会变？上层需不需要按值持有或 `sizeof`（如数组元素）？两个都"否"就走不透明。
- `static` 实例只在本文件用，头文件不需要 `extern`，天然没有暴露问题。
- 需要跨模块共享的常量**表**必须是完整类型（数组元素要算大小），此时用 `extern const T TABLE[]`，且类型本身不含可变的实现细节。

## 初始化顺序与所有权

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

- 这条链写在**一个地方**：有 CubeMX 时是 `Core/Src/freertos.c` 的 `USER CODE BEGIN Init`；没有时是 `User/TASK/task_init.c` 的 `Task_Init()`。
- **有 CubeMX 时，OS 对象一律配在 `.ioc`，由 `freertos.c` 生成**：任务、队列、信号量、互斥量都不许在 BSP/Device/Service 里用 `osThreadNew` / `osMessageQueueNew` / `osSemaphoreNew` / `osMutexNew` 创建，**也不许改用 FreeRTOS 原生 API**（`xSemaphoreCreateMutexStatic`、`xQueueCreate`、`xTaskCreate`、`xEventGroupCreate`）——那是同一个错误换了层皮，还多绕开了 CubeMX 的堆/静态度量。模块只 `extern` 生成句柄并使用。
  - 缺对象时的顺序固定：**先改 `frame.ioc` → 让用户用 CubeMX 重新生成 → 再编译**。不许"先自己建一个跑通，回头再挪"，挪的过程中固件一直是坏的，而且八成忘了挪。
- **需要 OS 对象的驱动要分两段初始化**：不需要句柄的纯软件/设备初始化放 `USER CODE BEGIN Init`；需要信号量/互斥量的 `BSP_xxx_Init()` 必须放 `USER CODE BEGIN RTOS_MUTEX`（要用队列的放 `USER CODE BEGIN RTOS_QUEUES`）——CubeMX 先执行 Init 段，再创建 mutex/queues，放早了句柄还是 `NULL`。
- **没有 CubeMX 时，所有 OS 对象集中在一个文件**：`User/TASK/task_init.c` 的 `Task_Init()` 先 `osKernelInitialize()`，再统一创建全部队列/信号量/互斥量/线程；BSP 的 `Init()` 只绑定或接收句柄。
- 有 CubeMX 时 `Dev_*_Init()` 通常退化为空：引脚模式已由 `MX_GPIO_Init()` 配好。只有需要上电时序、自检、复位芯片（陀螺仪、OLED）的设备才真要写 `Init`。
- 所有初始化必须在**第一个线程启动前**完成；线程里不许再初始化全局状态。
- `BSP_Board_Init()` 只调 BSP，**不许**调 Device —— 依赖方向不允许。
- 初始化函数返回 `int`，0 成功、负值失败；失败要有一个地方停下来（点灯/断言），不要静默继续。
- 判据：`grep -rn "BSP_IIC_Init\|BSP_.*_Init" Core/Src/*.c` 只应命中一处调用；本项目曾同时出现在 `freertos.c` 的 RTOS_MUTEX 段和 `main.c` 的生成区外（后者会被 CubeMX 重生成删掉，必须删）。

## 中断与并发契约

1. **中断里只允许调 BSP 的"搬字节"函数**（push 一个字节、置一个标志）。禁止在 ISR 里调 Service/APP、禁止 `malloc`、禁止日志输出。
2. ISR 与任务共享的变量必须 `volatile`；读写宽度 ≤ 4 字节（Cortex-M 对齐的 32 位读写是原子的），超过 4 字节要临界区。
3. 环形缓冲默认**单生产者单消费者**：ISR 里只 push，任务里只 pop；多写者要另外加临界区，不能假设 ringfifo 自带。
4. 临界区只保护"几条指令"：关中断期间不许调用可能阻塞的函数，也不许等外设。
5. RTOS 里中断优先级不能高于 `configMAX_SYSCALL_INTERRUPT_PRIORITY`，否则 ISR 里调 `FromISR` 版本会挂死。
6. 服务里的状态变量默认只被"一个线程"访问；要跨线程就明确写注释，由队列或临界区保护。
7. **可变全局量禁止跨模块写**：`extern` 出去的变量只许读；要共享状态就走队列、函数接口，或者留在模块 `static` 里由它自己的函数改。唯一例外是初始化阶段（单线程、启动前）的写。
   判据：`grep -n "extern" User/` 出来的每个变量，除句柄与常量表外都不该有第二次写入。
8. **唤醒与数据落盘的顺序**：ISR 里先写结果、再通知（`bus->result = ...` 在 `osSemaphoreRelease`/`vTaskNotifyGiveFromISR` 之前），否则任务被唤醒时读到的是旧值。`volatile` 只保证不被优化掉，不保证顺序——真需要顺序屏障时用 `__DMB()`。

## 时间基准、栈与堆

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

## Service 与 APP 的判据

### Service：四问，有一个"是"才建

1. 要时间轴吗？（闪烁、呼吸、连发、超时重发）
2. 有第二个使用者吗？（两个 APP 模块都要按键事件）
3. 是与产品无关的策略吗？（去抖、CRC、协议解析）
4. 能脱离硬件与线程单独测吗？

四个都"否" → 别建 Service，直接 `Dev_*`。判例：8 颗灯"显示档位"是 APP 直接调 `Dev_Led_Bar_Set()`；将来要做"流水灯"才建 `Svc_Led_Effect`。

### APP：只做两件事

- **保存业务状态**、**把命令映射成动作**。
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
- **APP 层的"薄"要能解释**：如果只有一个调用者、状态不跨触发源共享、也无法脱离硬件验证，这层就是"预支的层次"。留着可以，但要明确它在等第二个触发源；否则应合并进 TASK。判据见 `references/checks.md`。

## Common、Third_Party 与测试

- `Common/`：**自己写的纯软件**（错误码、CRC、位操作、定点数学、环形缓冲）。不许 include HAL、不许 include 上层。可以在 PC 上编译并单测。
- `Third_Party/`：**别人写的代码**，只读。不改造、不格式化、不在里面加业务。
- 用法：第三方库 → 配一个**适配器**放在需要它的层。
  - ringfifo（纯数据结构）→ 直接在 BSP/Service 里调；
  - 协议栈（nanomodbus）→ `Service/Svc_Modbus/` 适配传输与超时；
  - OLED 面板驱动 → `Device/Dev_Oled/` 适配，驱动里唯一的硬件调用（`HAL_I2C_Master_Transmit`）改成 `Dev_Oled_Write()`。
- 第三方库的 CMake 必须 `-w`，否则 `-Werror` 会把别人的警告变成你的错误（细节见 `references/cmake.md`）。
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

## 命名

| 对象 | 规则 | 例子 |
|---|---|---|
| 层目录 | 多词缩写全大写；单词只首字母大写 | `BSP` `APP` `OS` `TASK` `Dev` `Svc` `Common`（层目录仍写全称 `Device` `Service`） |
| 模块目录 | 层前缀 + `_` + PascalCase | `BSP/BSP_GPIO`、`Device/Dev_Key`、`Service/Svc_Key_Logic` |
| 文件 | 模块目录名全小写 | `dev_key.c`、`svc_key_logic.c` |
| CMake 目标名 | 模块目录名全小写，**层前缀一起带** | `dev_key`、`dev_led_bar`、`svc_key_logic` |
| 扩展名 | 永远小写 | `.c` `.h`（`.C` 会被当 C++） |
| 公共函数 | 模块目录名原样 + PascalCase 动作短语 | `BSP_GPIO_Write_Bits`、`Dev_Led_Bar_Set`、`Svc_Key_Logic_Tick` |
| 内部函数 | `static` + 同上规则 | `Svc_Key_Logic_Send_Event` |
| 类型 | `snake_case_t` | `dev_led_bar_t`、`key_event_t` |
| 宏 | 全大写 + 模块前缀 | `SVC_KEY_LONG_PRESS_MS` |
| 保护宏 | `__模块目录名_H__` | `__Dev_Key_H__` |
| 全局实例 | **不带 `g_`**（本项目选择，非通例） | `dev_key_inc`、`bsp_gpio_pa0` |
| 文件内静态量 | `s_` | `s_level` |

### 函数名四条硬规则

- 层前缀按层名原样拼：多词缩写全大写（`BSP_` `APP_` `OS_`），单词语缩写首字母大写（`Dev_` `Svc_`）；模块名里的完整单词首字母大写（`Dev_Led_Bar_`）。
- 动词在前、宾语在后：`Write_Bits` `Is_Pressed` `Mem_Read`；不要 `Bits_Write`。
- 禁止 `bsp_gpio_write_bits()` 这类全小写下划线写法；`User/TASK/` 下的任务入口由 CubeMX `.ioc` 生成，是唯一例外。
- 改名后逐处核对：`@param` 名、`@brief` 里的调用示例、CMake 注释里提到的旧函数名一并更新。

> `g_` 前缀是团队偏好，不是嵌入式通例。换工程时统一改一处（本规范选择不带）。

> CMake 目标名与 `target_link_libraries` 里的名字必须**逐字**一致（含大小写）。对不上时 CMake configure 不报错，只在编译期变成 `fatal error: xxx.h: No such file or directory`；改过名字还必须重新 Configure。排查步骤见 `references/cmake.md` §12.11、§12.12。

## 没有 CubeMX 的平台

- 每个 BSP 模块自带 `BSP_xxx_Init()`（时钟、模式、上下拉都在里面），句柄同时实例化。
- 由一个 BSP 层文件统筹顺序：`User/BSP/BSP_Board/bsp_board.c` 的 `BSP_Board_Init()` 按依赖顺序调各 `BSP_xxx_Init()`，对外只暴露这一个函数。
- 引脚号写在 BSP 模块自己的 `.c` 里，Device 只引用句柄（见「句柄与不透明类型」）。
- 有 CubeMX 时**不建** `BSP_Board`：引脚由 `MX_GPIO_Init()` 配好，BSP 只做"句柄 → 端口"的翻译。

## 快速开始（15 分钟）

1. `User/CMakeLists.txt`：`set(USER_WARN_FLAGS -Wall -Wextra -Werror)`（必须在 `add_subdirectory` 之前）→ 一行一层的 `add_subdirectory()`（Common / Third_Party / BSP / Device / Service / APP / TASK）→ 末尾给主目标 `target_link_libraries`（plain 签名）。**每个层目录各建一个 `CMakeLists.txt`**，模块库写在各层自己的那一个文件里；每层先用 `references/cmake.md` 的四行模板建第一个模块，让空库也能 link 通。
2. `BSP/BSP_GPIO/`：不透明句柄 + `Read/Write/Toggle`。
3. 在 `BSP/BSP_GPIO/bsp_gpio.c` 里加引脚句柄（引脚号写这里）；在 `Device/Dev_Led_Bar/dev_led_bar.c` 里实例化设备句柄、引用这些引脚。
4. `Service/Svc_Key_Logic/`：表驱动消抖，投递事件到队列。
5. `APP/APP_Level/`：业务状态 + `Step()`。
6. `APP/APP_Key_Cmd/`：命令表 `{key_id, event, handler}`。
7. `.ioc` 里配两个线程、队列、需要的信号量/互斥量；生成 `freertos.c`。
8. `TASK/` 写两个线程体；`Core/Src/freertos.c` 的 `USER CODE BEGIN Init` 串不需要 OS 对象的初始化，需要信号量/互斥量的 BSP 绑定放 `USER CODE BEGIN RTOS_MUTEX`（早于建线程）→ 编译。

## 模块模板

```c
/* dev_xxx.h */
#ifndef __Dev_Xxx_H__
#define __Dev_Xxx_H__

#include <stdint.h>                /* 连 bsp_gpio.h 都不用：布局在 .c，头文件只给名字 */

typedef struct dev_xxx dev_xxx_t;          /* 布局在 .c，头文件不写 */

extern dev_xxx_t dev_xxx;                  /* 不完整类型：只能取地址/传指针 */

int     Dev_Xxx_Init(void);
uint8_t Dev_Xxx_Read(const dev_xxx_t *dev);
uint8_t Dev_Xxx_Count(const dev_xxx_t *dev);   /* 上层要的属性都走只读函数 */

#endif /* __Dev_Xxx_H__ */
```

```c
/* dev_xxx.c */
#include <stddef.h>
#include "dev_xxx.h"
#include "bsp_gpio.h"            /* BSP 类型只在这里出现 */

struct dev_xxx {
    const bsp_gpio_t *pin;       /* 下层句柄，一行端口号都不写 */
    uint8_t           active_low;
};

dev_xxx_t dev_xxx = { &bsp_gpio_pa0, 0u };

int Dev_Xxx_Init(void) { return 0; }
```

## 迁移旧工程

1. 先建层目录与空的 `CMakeLists.txt`，让工程能编译过（空库也要能 link）。
2. 从最底层往上搬：BSP → Device → Service → APP；每搬一层编译一次。
3. 句柄化放在搬 Device 时做，不要最后补；**同时把结构体从 `.h` 挪进 `.c`**，一步到位，否则永远挪不完。
4. 改名用脚本 + 一次全仓库 `grep` 校验；PowerShell 的 `-ne` 大小写不敏感，只改大小写时要用 `-cne`。
5. Windows 下改目录大小写必须两段式 `[System.IO.Directory]::Move`（先 `X_tmp` 再目标名），否则 `Remove-Item` 会误删。

---

# 注释规范（RC_Dog Doxygen 风格）

## 适用范围与优先级

- 适用于本项目所有手写的 `.c` / `.h`。
- 与任何其他「C 注释规范」冲突时，**以本节为准**（分区横幅宽度、步骤标记、行内注释三种写法最容易撞车）。
- 代码可以变，注释结构不能随手破坏：改代码时按「修改已有代码时」同步。

## 文件头

```c
/**
 * @file bsp_gpio.c
 * @brief GPIO 通用 BSP 层实现（直接操作 BSRR/IDR）
 * @author 李嘉图
 * @date 2026-10-21
 *
 * @note 只做寄存器级读写，不含任何业务语义；上层不要直接 include 本文件。
 */
```

- `@file` 必须带后缀
- `@brief` 一句话说清"这个文件是什么"
- `@author` 固定 `李嘉图`，`@date` 固定 `YYYY-MM-DD`
- `@note` 写设计取舍、前提、坑；没有实质内容就整行删掉，不留空标签

## 分区横幅（满行 80 列）

```c
/*==============================================================================
 * 头文件包含
 *============================================================================*/
```

精确宽度：

| 行 | 内容 | 字符数 |
| --- | --- | --- |
| 第 1 行 | `/*` + 78 个 `=` | 80 |
| 第 2 行 | 空格 + `*` + 空格 + 分区名 | 不定，≤ 76 |
| 第 3 行 | 空格 + `*` + 76 个 `=` + `*/` | 80 |

常用分区名：头文件包含、宏定义与常量、类型定义、内部变量、内部函数、接口实现、初始化、查询。

`#include` 段内再分两小节：

```c
// 固定包含
#include <stdint.h>
#include "bsp_gpio.h"
// 功能包含
#include "stm32f1xx_hal.h"
```

## 函数注释（核心）

注释块紧贴函数上方，中间不留空行。**`.h` 的声明和 `.c` 的定义各写一份，两份内容一致。**

```c
/**
 * @brief 按位写 GPIO 输出（一次写 BSRR，原子更新）
 * @param port      端口基址，如 GPIOA
 * @param mask      要写入的位掩码
 * @param high_bits 掩码内需要输出高电平的位
 * @retval 无
 * @note 高电平位写 BSRR 低 16 位，低电平位写高 16 位，同一拍生效。
 */
void BSP_GPIO_Write_Bits(GPIO_TypeDef *port, uint16_t mask, uint16_t high_bits);
```

```c
/**
 * @brief 设置进度条点亮颗数
 * @param bar 灯条句柄，不可为 NULL
 * @param lit 目标颗数，超出范围自动夹紧到 [0, Dev_LED_Bar_Count(bar)]
 * @retval 无
 */
void Dev_LED_Bar_Set(const dev_led_bar_t *bar, uint8_t lit);
```

规则：

- `@brief` 必写，说清"做什么"，不要写成"设置"/"处理"这种空话
- 每个参数一个 `@param`，名字与签名完全一致，说明另起一列对齐
- 返回值用 `@retval` **逐值枚举**：

```c
/**
 * @brief 判断按键是否处于按下状态
 * @param btn 按键句柄，不可为 NULL
 * @retval 1: 已按下（已按 active_low 换算成逻辑电平）
 * @retval 0: 未按下
 */
uint8_t Dev_Button_Is_Pressed(const button_t *btn);
```

- `void` 返回值的函数**不写 `@retval`**（写 `@retval 无` 也算合规，但全项目统一一种——本项目统一写 `@retval 无`）
- 有副作用、时序要求、线程安全、调用时机时加 `@note`
- 无参数无返回值的最小写法：一行 `@brief` 即可

```c
/**
 * @brief 按键服务初始化，清零所有按键状态与事件队列
 */
void Svc_Key_Logic_Init(void);
```

## 结构体、枚举、宏、静态变量

结构体与枚举定义上方写 `@brief`，成员用行内 `/* */` 右对齐成一列：

```c
/**
 * @brief 灯条设备句柄（由设备模块的 .c 静态实例化）
 */
typedef struct dev_led_bar {
    const bsp_gpio_t *pin;        /* 引脚句柄，极性由 active_low 决定 */
    uint8_t           first;      /* 起始引脚号（仅 BSP 层可见的差异量） */
    uint8_t           count;      /* 灯珠数量 */
    uint8_t           active_low; /* 1: 低电平点亮 */
} dev_led_bar_t;
```

```c
/**
 * @brief 按键事件
 */
typedef enum {
    KEY_EVENT_NONE = 0, /* 无事件 */
    KEY_EVENT_PRESSED,  /* 消抖后的按下沿 */
    KEY_EVENT_REPEAT,   /* 长按连发 */
    KEY_EVENT_RELEASED  /* 消抖后的抬起沿 */
} key_event_t;
```

- 有语义的宏、`static` 变量、`const` 句柄也要一句话 `@brief` 或行内 `/* */`
- 构造宏（`LED_BAR_DEFINE`、`BUTTON_DEFINE`）必须说明每个参数与生成对象的生命周期
- 注意：不透明类型（`typedef struct dev_led_bar dev_led_bar_t;`）的**布局注释写在 `.c` 的结构体定义上方**，头文件那份只写一句"布局见 .c"

## 函数内部步骤

多步骤函数用步骤标记分段：

```c
/* ---------- 查重 ---------- */
/* ---------- 容量检查 ---------- */
/* ---------- 注册 ---------- */
```

步骤多于 3 个带编号：

```c
/* ---------- 1. 采样 ---------- */
/* ---------- 2. 消抖 ---------- */
/* ---------- 3. 判定按下沿/抬起沿 ---------- */
```

单行说明用 `// 说明`（独立成行）或行尾 `/* 说明 */`，右对齐成一列。禁止 `//======` 和 `/*---*/` 这类假分隔线。

## 注释语言

一律简体中文，技术名词（BSRR、IDR、句柄、消抖、原子）保留英文原文。不要中英混杂写半句。

## 修改已有代码时

1. **新增**函数/结构体/变量：`.h` 与 `.c` 各补一份完整注释，`@brief/@param/@retval` 保持一致。
2. **只改实现、接口不变**：只更新 `.c` 里该函数的 `@note`，`@param` 不动。
3. **改接口**：`.h` 和 `.c` 的 `@brief/@param/@retval` 同步改，别只改一处。
4. **结构调整**：新单元补完整注释，废弃的整块删掉，不留死注释。
5. **不要顺手重排代码**：只加注释、只改注释；除非明确要求，否则不动逻辑。

## 反面例子（速览）

```c
// 错误：只有 .h 有注释，.c 的定义上方光秃秃
void Dev_LED_Bar_Set(const dev_led_bar_t *bar, uint8_t lit)
{
```

```c
/**
 * @brief 设置
 */                                    /* 错误：@brief 没写清做什么 */
void Dev_LED_Bar_Set(const dev_led_bar_t *bar, uint8_t lit);  /* 错误：缺 @param */
```

```c
/**
 * @brief 读取按键
 * @param btn 按键
 * @retval 按下状态                      /* 错误：@retval 必须逐值枚举 0/1 */
 */
```

```c
/*========================= 头文件包含 =========================*/  /* 错误：横幅不是 80 列 */
/* ── 采样 ── */                                                  /* 错误：步骤标记格式不对 */
```

更多见 `references/checks.md` 的「注释反模式」。

## 提交前自检

**架构**

- [ ] `User/` 下只有 BSP / Device / Service / APP / TASK（`Common`、`Third_Party` 按需创建，可为空）
- [ ] 每个模块目录名是 `层前缀_模块`，文件名是它的小写；CMake 库名 = 模块目录名全小写且带层前缀
- [ ] 一层一个 `CMakeLists.txt`，模块目录里没有 `CMakeLists.txt`，也没有 `add_subdirectory(<模块目录>)`
- [ ] 每条 `PUBLIC` 都能指到头文件里的一个下层类型名；只有 `.c` 用到的写 `PRIVATE`
- [ ] 每个头文件里**没有结构体布局**（不透明类型），`extern` 的对象是不完整类型
- [ ] `User/` 下的 `.h` 里没有 `stm32f1xx_hal.h` / `FreeRTOS.h` / `cmsis_os2.h`
- [ ] 上层没有任何 `->成员` 访问下层结构体；需要属性就有对应的只读函数
- [ ] 引脚号只出现在 BSP 的 `.c` 里，设备句柄不存端口号，句柄都是 `const`
- [ ] 没有反向依赖（Device 调 Service、BSP 调 Device、下层调 `Task_*`）
- [ ] 初始化只有一条链，且在线程启动前完成；需要 OS 句柄的 `Init()` 在 `USER CODE BEGIN RTOS_MUTEX/RTOS_QUEUES`
- [ ] ISR 里只调 BSP 搬字节，共享变量 `volatile` 且 ≤ 4 字节
- [ ] 线程用 `osDelayUntil`，栈 ≥ 256 字并实测水位
- [ ] CubeMX 工程里 `grep -rn "osMutexNew\|osSemaphoreNew\|osMessageQueueNew\|osThreadNew\|xSemaphoreCreate\|xQueueCreate\|xTaskCreate" User/` 为 0
- [ ] 跨模块 `extern` 变量只读；`.ioc` 改完重新生成过，`git diff --stat Core cmake/` 没有意外删除

**注释**

- [ ] 每个 `.c`、`.h` 都有文件头（@file/@brief/@author/@date）
- [ ] `.c` 里每个函数定义上方都有注释块，不是只有 `.h` 有；`static` 函数也有
- [ ] 每个非 void 函数都有 `@retval`，且逐值枚举
- [ ] 每个参数都有 `@param`，名字与签名完全一致
- [ ] 函数名符合 `模块目录名_PascalCase`，没有 `bsp_gpio_write_bits()` 这类全小写形式
- [ ] 分区横幅三行宽度对齐（80 / ≤76 / 80）
- [ ] 多步骤函数有 `/* ---------- 步骤 ---------- */`
- [ ] 没有空 `@note`、复制错的参数名、TODO 残留、注释里过期的旧函数名
- [ ] 结构体/枚举成员注释右对齐成一列

完整清单（含 CMake、CubeMX、Cache 等条目）见 `references/checks.md`。

---
name: embedded-project-scaffold
description: 嵌入式 C 工程分层与命名规范：User/ 下 BSP/Device/Service/APP/TASK，硬件与设备都句柄化，引脚号只写 BSP，CMake 库边界卡越层，任务由 CubeMX 或平台层创建。
---

# 嵌入式 C 工程分层规范（速查版）

适用：Cortex-M + CMake，裸机 / FreeRTOS / RT-Thread。
目标：**换 MCU 只改 BSP，换 OS 只改 TASK 骨架，换需求只改 APP。**

## 0. 三条总原则

1. **依赖单向**：`TASK → APP → Service → Device → BSP → HAL`。`Common`、`OS` 是横向层，谁都能用，它们不反向依赖。
2. **句柄化**：一个引脚、一台设备、一份状态各一个句柄，函数收句柄指针，不收裸参数。
3. **越层由编译期拦**：靠 CMake 的 `PRIVATE` / `PUBLIC`，不靠自觉。

## 1. 东西放哪

| 内容 | 层 | 判据 |
|---|---|---|
| 读写电平、I2C 一次事务、UART 搬字节 | BSP | 只碰寄存器与 HAL，不知道接的是什么 |
| 灯条、按键、陀螺仪、OLED 面板 | Device | 有"这台设备"的概念，仍不知道业务 |
| 消抖、长按、滤波、帧格式、重发 | Service | 跨设备，或带时间轴、带流程 |
| CRC、位操作、定点数学、错误码 | Common | 纯函数，不碰硬件也不碰时间 |
| 档位、模式、菜单、协议命令 | APP | 产品行为，Service 不该知道 |
| 线程体、周期、优先级 | TASK | 只回答"多久跑一次、跑哪个一步函数" |
| 别人的库 | Third_Party | 只读区，一个字不改 |

## 2. 目录树

```
User/
├── CMakeLists.txt          # add_subdirectory 各层 + 挂到工程
├── Common/                 # 按需：纯软件，可上 PC 编译
├── Third_Party/            # 按需：别人的代码，只读
├── BSP/BSP_GPIO/           # bsp_gpio.c / bsp_gpio.h
├── Device/Dev_Key/         # dev_key.c / dev_key.h
│   └── Dev_Led_Bar/
├── Service/Svc_Key_Logic/  # svc_key_logic.c + svc_key_logic_cfg.h
├── APP/APP_Level/          # app_level.c
│   └── APP_Key_Cmd/
└── TASK/task_key_scan.c    # 一个任务一个 .c，平铺，不分子目录
```

`Core/`、`Drivers/`、`Middlewares/`、根 `CMakeLists.txt` 是生成区：**只读**，唯一例外是 `/* USER CODE BEGIN/END */` 之间。

## 3. 命名

| 对象 | 规则 | 例 |
|---|---|---|
| 层目录 | 多词缩写全大写 / 完整单词首字母大写 | `BSP`、`APP`、`OS`、`Device`、`Service` |
| 模块目录 | `层前缀_模块` | `BSP_GPIO`、`Dev_Key`、`Svc_Key_Logic`、`APP_Level` |
| 文件 | 模块目录名全小写 | `dev_key.c`、`svc_key_logic.c` |
| 公共函数 | 模块目录名 + `_` + PascalCase 动作 | `BSP_GPIO_Write`、`Svc_Key_Logic_Tick` |
| 类型 | `snake_case_t` | `dev_key_t`、`bsp_gpio_t` |
| 宏 / 枚举值 | 全大写 | `KEY_EVENT_DOWN`、`BSP_GPIO_DIR_IN` |
| 保护宏 | `__模块目录名_H__` | `__Dev_Key_H__` |
| 全局实例 | 不带 `g_`（项目选择） | `bsp_gpio_pa0`、`dev_key_inc` |
| 文件内静态量 | `s_` | `s_level` |

## 4. 句柄

| 层 | 句柄 | 谁实例化 |
|---|---|---|
| BSP | `bsp_gpio_t`（一个引脚） | **BSP 自己的 `.c`**，引脚号只写这一处 |
| Device | `dev_key_t` / `dev_led_bar_t`（一台设备） | Device 自己的 `.c`，只引用 BSP 句柄 |
| Service | `svc_*_t`（一份状态） | 用它的 APP / TASK |

硬规则：

- Device 句柄只存"下层句柄指针 + 差异量（极性、地址）"，**不存端口号**。
- 句柄默认 `const`，进 `.rodata`，不占 RAM；指针参数为 `NULL` 要能安全返回。
- 同种设备可多份：加第二台灯条 = 加一组 BSP 句柄 + 一个设备句柄，函数一行不改。
- **BSP 头文件不许 include 厂商头**：句柄里需要硬件类型时用 `void *`，`(GPIO_TypeDef *)` 只出现在 BSP 的 `.c`。理由：HAL 的 `GPIO_TypeDef` 是匿名 struct 的 typedef，无法前置声明；写进头文件就会一路传染到 APP。

## 5. 初始化（只有一条链）

```
BSP_Board_Init()            ← 仅无 CubeMX 的平台
      ↓
Dev_Key_Init()、Dev_Led_Bar_Init()      ← 只做上电时序/自检，有 CubeMX 时常为空
      ↓
Svc_Key_Logic_Init()                     ← 服务状态
      ↓
APP_Level_Init()                         ← 业务状态
      ↓
创建线程（CubeMX 的 freertos.c / 平台 task_init.c）
```

- 写在**一处**：有 CubeMX 时是 `Core/Src/freertos.c` 的 `USER CODE BEGIN Init`（在建队列与建线程之前）；没有时是 `User/TASK/task_init.c`。
- 必须早于第一个线程；线程里不许再初始化全局状态。
- 初始化返回 `int`，0 成功、负值失败；失败要停下（点灯/断言），不要静默继续。

## 6. 中断与并发

1. ISR 里只调 BSP 的搬字节函数；禁止调 Service/APP、禁止 `malloc`、禁止打印。
2. 共享变量 `volatile` 且 ≤ 4 字节（对齐 32 位读写原子），超过要临界区。
3. 环形缓冲默认单生产者单消费者。
4. 中断优先级不得高于 `configMAX_SYSCALL_INTERRUPT_PRIORITY`。
5. `extern` 出去的变量只许读，除句柄与常量表外没有第二处写入。

## 7. TASK

- 一任务一个 `.c`，平铺，不分子目录；线程体不需要 `.h`（除非有第二个调用者）。
- 线程体只有骨架，业务调下层的"一步函数"：

```c
void Task_Key_Scan(void *argument)
{
    const uint32_t freq   = osKernelGetTickFreq();
    const uint32_t period = (freq * TASK_KEY_SCAN_PERIOD_MS) / 1000u;
    uint32_t       next   = osKernelGetTickCount();
    (void)argument;

    for (;;) {
        next += period;
        (void)osDelayUntil(next);                              /* 只有这几行和 OS 有关 */
        Svc_Key_Logic_Tick((uint16_t)TASK_KEY_SCAN_PERIOD_MS);  /* 一步动作 */
    }
}
```

- 用 `osDelayUntil` 不用 `osDelay`；tick 换算**先乘后除**（`freq / 1000u * ms` 会被截断成 0）。
- 栈 ≥ 256 字（1 KB）起步，跑起来用 `uxTaskGetStackHighWaterMark` 收紧。
- 优先级：扫描/采集 `osPriorityNormal`，命令/日志 `osPriorityBelowNormal`；且必须小于 `configMAX_PRIORITIES`（否则被静默钳制）。

## 8. CMake

- 每层一个 `CMakeLists.txt`，每模块一个 `STATIC` 库；`PUBLIC` 传头目录，`PRIVATE` 藏下层。
- 头文件里出现的类型，其所在库必须 `PUBLIC` 链接，否则上层看到 incomplete type。
- `-Wall -Wextra -Werror` 只对手写代码；第三方加 `-w`（必须排在 `-Werror` 之后）。
- 任务体用 **OBJECT 库**再 `target_sources(frame PRIVATE $<TARGET_OBJECTS:task_objects>)`：静态库没人引用的强符号会被丢掉，`__weak` 空实现反而赢。
- 根 `CMakeLists.txt` 若由 CubeMX 生成，不要改它；在 `User/CMakeLists.txt` 末尾用**无关键字签名**补 `target_link_libraries(${CMAKE_PROJECT_NAME} <库>)`。

## 9. 四个模板

```c
/* BSP：头文件不碰厂商头 */
typedef struct { void *port; uint16_t pin; bsp_gpio_dir_t dir; } bsp_gpio_t;
extern const bsp_gpio_t bsp_gpio_pa0;
uint8_t BSP_GPIO_Read(const bsp_gpio_t *gpio);   /* NULL → 0 */
```

```c
/* Device：只引用句柄 */
typedef struct { const bsp_gpio_t *pin; uint8_t active_low; } dev_xxx_t;
extern dev_xxx_t dev_xxx;
uint8_t Dev_Xxx_Read(void);
/* .c 里：dev_xxx_t dev_xxx = { &bsp_gpio_pa0, 0u }; */
```

```c
/* Service：表驱动 + 一步函数，出口只有一个 */
void Svc_Xxx_Init(void);
void Svc_Xxx_Tick(uint16_t elapsed_ms);
```

```c
/* APP：只有 Init 与 Step，不建线程、不碰内核 */
void APP_Xxx_Init(void);
void APP_Xxx_Step(int8_t delta);
```

## 10. 自检（能跑就别用眼睛看）

```bash
grep -rn "stm32f1xx_hal.h" User/          # 只应命中 BSP/*/*.c
grep -rn "GPIOA\|GPIOB\|GPIOC" User/      # 只应命中 BSP/*/*.c
grep -rn "Task_" User/*/                  # 下层不该出现 Task_ 调用
grep -n "configMAX_PRIORITIES" Core/Inc/FreeRTOSConfig.h
grep -rn "osDelay(" User/TASK/            # 应为空，统一用 osDelayUntil
```

## 11. 深入阅读

项目里有 `docs/embedded-project-scaffold.md` 时再看：CubeMX 生成物边界、M7/H7 的 D-Cache 维护、OS 四种来源与移植要求、反模式、完整自检清单、旧工程迁移步骤。

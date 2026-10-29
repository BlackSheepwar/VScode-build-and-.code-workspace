# 反模式与自检清单

配合主文件 SKILL.md 使用，提交前过一遍。

## 架构反模式

- 在 `Device` 里写 `for (;;)` 或 `osDelay`（不是它的职责，也测不了）
- 在 `BSP` 里判断"档位"或按键长按
- 在 `Device`/`APP` 里写 `GPIOA`、`GPIO_PIN_x`（引脚号只许出现在 BSP 的 `.c` 里）
- `Service` 里 `#include "app_level.h"`（反向依赖）
- `APP` 直接 `#include "FreeRTOS.h"`
- `TASK` 里写业务（改需求要动任务文件）
- 一个 `.c` 里 800 行、既有协议又有业务
- 两份 `FreeRTOSConfig.h`
- 第三方代码里改 bug（下次升级全丢）
- 为了"能看见头"把依赖一路写成 `PUBLIC`（边界就此失效，`-D` 也跟着传染）
- 用 `target_include_directories` 给主目标硬塞下层头目录（链接到位就不需要，塞了等于绕开依赖关系）
- `add_library` 的目标名与 `target_link_libraries` 里写的名字不一致（大小写、前缀）：CMake configure 不报错，只在编译期变成 `fatal error: xxx.h: No such file or directory`
- 改了 `add_library` 的名字只 Build 不重新 Configure（`build.ninja` 里还是旧的 `-l`）
- 一个工程两套库命名习惯（`key` 与 `bsp_gpio` 并存）：链接行与 `lib*.a` 读不出模块是哪个
- 一个硬件 / 一台设备 / 一个模块建一个 `CMakeLists.txt`，再用 `add_subdirectory(<模块目录>)` 拆出去（库清单与链接关系散进目录树，改一条链接要逐层翻；层内一个文件一次读完）
- 为了"四行齐全"给 Common 的纯函数库链 `stm32cubemx`（`-DUSE_HAL_DRIVER` 随之传染，"能在 PC 上单测"这条判据作废）
- CubeMX 工程里在 BSP/Device/Service 自己 `osMutexNew` / `osSemaphoreNew` / `osMessageQueueNew` / `osThreadNew`（对象该由 `.ioc` 生成）
- 同一件事换成 FreeRTOS 原生 API 在模块里建对象（`xSemaphoreCreateMutexStatic` / `xQueueCreate` / `xTaskCreate`）：判据 `grep` 一样为 0，别只看 `os*New`
- 为了"先把功能跑通"，在模块里临时建对象却不去改 `.ioc`：这个过渡态会长期留在代码里，CubeMX 日后一生成就与它打架
- 把需要信号量/互斥量的 `BSP_xxx_Init()` 放在 `USER CODE BEGIN Init`（CubeMX 此时还没创建信号量，句柄是 `NULL`）
- 头文件里写字结构布局、`.c` 里再 `typedef` 一次：上层为了读一个字段就把结构体打开，之后加字段等于改接口
- 上层直接 `dev_led_bar_0.count` 读成员（`error: invalid use of incomplete typedef`）：该在下层加只读函数，不该把结构体公开
- `typedef X { ... };`（缺 `struct`，报 `expected '=', ',', ';', 'asm' or '__attribute__' before '{' token`）或 `typedef struct X { ... };`（尾部无别名，`-Werror` 下报 `useless storage class specifier in empty declaration`）
- 头文件 `extern dev_led_bar_t dev_led_bar_0;`，`.c` 定义成 `dev_led_bar`：编译期不报，链接期 `undefined reference to 'dev_led_bar_0'`
- 为了拿 `GPIO_TypeDef` 在头文件里 include `stm32f1xx_hal.h`：HAL 顺着 include 链传染到 APP，"上层不碰 HAL"只剩口号
- 对匿名 struct 的 typedef（`GPIO_TypeDef`）硬写 `struct XXX *` 前置声明：没有 tag 名，编不过，只能 `void *`
- 运行时创建的句柄按值存进结构体（`osMutexId_t lock;`）再到 `Init()` 里赋值：实例定义处是 `NULL`，先调先炸，两次 `Init()` 还会覆盖
- 运行时句柄只判 `lock != NULL`、不判 `*lock != NULL`：指针非空不等于句柄已创建
- APP 层只有一个调用者、状态不跨触发源共享、也无法脱离硬件验证，却仍然保留（预支的层次）：要么在注释里写明它在等第二个触发源，要么合并进 TASK

## 架构自检清单

- [ ] `User/` 下只有 BSP / Device / Service / APP / TASK（`Common`、`Third_Party` 按需创建，可为空）
- [ ] 每个模块目录名是 `层前缀_模块`，文件名是它的小写
- [ ] 每个硬件都有 BSP 句柄，每台设备都有 Device 句柄
- [ ] 引脚号只出现在 BSP 的 `.c` 里，Device 不写端口号
- [ ] `User/` 下每个 `.h` 只给类型名、不给布局；没有 `stm32f1xx_hal.h` / `FreeRTOS.h` / `cmsis_os2.h`
- [ ] 头文件里声明的实例名与 `.c` 里的定义逐字一致（`nm` 对得上，不是链接期 `undefined reference`）
- [ ] 运行时创建的句柄存的是地址（`osMutexId_t *lock`），且 `lock` 与 `*lock` 两层都判过 `NULL`
- [ ] 设备句柄不存端口号，句柄都是 `const`
- [ ] 每个模块四行模板齐全（`add_library` / `PUBLIC` 头目录 / `target_link_libraries` / `target_compile_options`），库名 = 模块目录名全小写且带层前缀（`Dev_Key` → `dev_key`），且这四行写在**所属层的那一个 `CMakeLists.txt`** 里
- [ ] 一层一个 `CMakeLists.txt`：`Get-ChildItem -Recurse -Filter CMakeLists.txt User` 只列出"`User/CMakeLists.txt` + 每个层目录各一条"，没有 `User/BSP/BSP_GPIO/CMakeLists.txt` 这类模块级文件，也没有 `add_subdirectory(<模块目录>)`
- [ ] 每条链接行里的目标名都能在某个 `add_library` 里逐字找到；改过名字后重新 Configure，`compile_commands.json` 的 `-I` 已含新目录
- [ ] 每条 `PUBLIC` 都能指到头文件里的一个下层类型名；只有 `.c` 用到的写 `PRIVATE`
- [ ] 没有用 `file(GLOB)` 收源文件（新文件不重配不生效，顺序还按字母）
- [ ] 生成文件（根 `CMakeLists.txt`、`cmake/stm32cubemx/`）一个字没改；user 库按层写在各层 `CMakeLists.txt` 里，由 `User/CMakeLists.txt` 用 plain 签名补链到主目标
- [ ] 每个建了的库都有人链接；`$<TARGET_OBJECTS:>` 之外没有依赖是靠"以为会传递"成立的
- [ ] `nm` 确认任务符号在主目标里是 `T`，`freertos.c.obj` 里是 `W`（不是空转的弱函数）
- [ ] 没有反向依赖（Device 调 Service、BSP 调 Device、下层调 `Task_*`）
- [ ] 初始化只有一条链，且在线程启动前完成
- [ ] ISR 里只调 BSP 搬字节，共享变量 `volatile` 且 ≤ 4 字节
- [ ] 线程用 `osDelayUntil`，栈 ≥ 256 字并实测水位
- [ ] `TASK/` 每个文件 OS 调用只在 `for (;;)` 骨架里
- [ ] 只有一个地方创建线程，一个 `.ioc`，一份 `FreeRTOSConfig.h`
- [ ] CubeMX 工程里 `grep -rn "osMutexNew\|osSemaphoreNew\|osMessageQueueNew\|osThreadNew\|xSemaphoreCreate\|xQueueCreate\|xTaskCreate\|osTimerNew\|osEventFlagsNew" User/` 为 0；无框架工程只命中 `task_init.c`
- [ ] 需要信号量/互斥量的 BSP `Init()` 放在 `USER CODE BEGIN RTOS_MUTEX`（只等互斥量/信号量）或 `USER CODE BEGIN RTOS_QUEUES`（要用队列），不需要 OS 对象的才放 `USER CODE BEGIN Init`
- [ ] 第三方代码不在 `-Werror` 范围、有 README 记来源
- [ ] 第三方目标的编译选项里 `-w` 排在最后（查 `flags.make`，不靠猜）
- [ ] `.ioc` 改完重新生成过，`git diff --stat Core cmake/` 里没有意外删除
- [ ] 跨模块 `extern` 变量只读，除句柄与常量表外没有第二处写入
- [ ] 用到的最大 `osPriority` 小于 `FreeRTOSConfig.h` 里的 `configMAX_PRIORITIES`
- [ ] 有 D-Cache 的平台，DMA 缓冲做了 clean / invalidate，或放在非 cache 区
- [ ] （可选）`tests/` 在仓库根，CI 同时跑"固件构建"与"host 单测"；不上 CI 的小工程可跳过这条

## 注释反模式

- 只有 `.h` 有注释，`.c` 的定义上方光秃秃
- `@brief 设置` / `@brief 处理`：没写清做什么
- 有参数却没有 `@param`，或 `@param` 名与签名不一致（改名后没同步）
- `@retval 按下状态`：没有逐值枚举 `1` / `0`
- `@note` 留空标签，或写"暂无"
- 分区横幅不是 80 列（`/*==== 头文件包含 ====*/` 这种一行式、或宽度对不上的三行式）
- 步骤标记写成 `/* ── 采样 ── */`、`//====`、`/*---*/` 这类假分隔线
- 注释里提到已经被改名的旧函数（`APP_Level_Step_Level` 之类），或 CMake/头文件注释里的旧名
- `TODO` 长期残留、注释描述与代码行为相反
- 结构体成员注释没右对齐成一列，或成员顺序与注释顺序错位

## 注释自检清单

- [ ] 每个 `.c`、`.h` 都有文件头（`@file`/`@brief`/`@author`/`@date`）
- [ ] `.c` 里每个函数定义上方都有注释块，不是只有 `.h` 有
- [ ] `static` 函数也有注释块
- [ ] 每个非 void 函数都有 `@retval`，且逐值枚举；void 函数全项目统一（写 `@retval 无` 或不写）
- [ ] 每个参数都有 `@param`，名字与签名完全一致
- [ ] 函数名符合 `模块目录名_PascalCase`，没有 `bsp_gpio_write_bits()` 这类全小写形式
- [ ] 分区横幅三行宽度对齐（80 / ≤76 / 80）
- [ ] 多步骤函数有 `/* ---------- 步骤 ---------- */`，超过 3 步带编号
- [ ] 没有空 `@note`、复制错的参数名、TODO 残留、过期的旧函数名
- [ ] 结构体/枚举成员注释右对齐成一列；不透明类型的布局注释写在 `.c` 的定义处

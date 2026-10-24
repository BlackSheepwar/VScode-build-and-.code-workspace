# 反模式与自检清单

配合主文件 SKILL.md 使用，提交前过一遍。

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
- 为了"能看见头"把依赖一路写成 `PUBLIC`（边界就此失效，`-D` 也跟着传染）
- 用 `target_include_directories` 给主目标硬塞下层头目录（链接到位就不需要，塞了等于绕开依赖关系）

## 十九、自检清单

- [ ] `User/` 下只有 BSP / Device / Service / APP / TASK（`Common`、`Third_Party` 按需创建，可为空）
- [ ] 每个模块目录名是 `层前缀_模块`，文件名是它的小写
- [ ] 每个硬件都有 BSP 句柄，每台设备都有 Device 句柄
- [ ] 引脚号只出现在 BSP 的 `.c` 里，Device 不写端口号
- [ ] 设备句柄不存端口号，句柄都是 `const`
- [ ] 每个模块四行模板齐全（`add_library` / `PUBLIC` 头目录 / `target_link_libraries` / `target_compile_options`），库名全小写
- [ ] 每条 `PUBLIC` 都能指到头文件里的一个下层类型名；只有 `.c` 用到的写 `PRIVATE`
- [ ] 没有用 `file(GLOB)` 收源文件（新文件不重配不生效，顺序还按字母）
- [ ] 生成文件（根 `CMakeLists.txt`、`cmake/stm32cubemx/`）一个字没改，user 库全写在 `User/CMakeLists.txt` 里且用 plain 签名
- [ ] 每个建了的库都有人链接；`$<TARGET_OBJECTS:>` 之外没有依赖是靠"以为会传递"成立的
- [ ] `nm` 确认任务符号在主目标里是 `T`，`freertos.c.obj` 里是 `W`（不是空转的弱函数）
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
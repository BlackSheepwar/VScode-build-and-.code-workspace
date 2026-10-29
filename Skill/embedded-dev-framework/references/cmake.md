# CMake 专题

配合主文件 SKILL.md 使用。本文件有自己的小节号（12.x），与主文件章号无关。

## CMake 的职责与可见性

### 12.1 三个文件的职责，一个都不许串

| 文件 | 谁写 | 只干这些事 |
|---|---|---|
| 根 `CMakeLists.txt` | CubeMX 生成 | **只读**。`# Add user defined libraries` 下面保持为空 |
| `User/CMakeLists.txt` | 手写 | 共用变量 → `add_subdirectory` 各层 → 给主目标补链接。三件事，顺序固定 |
| `User/<层>/CMakeLists.txt` | 手写 | **一层一个文件**，本层每个模块一个库、四行模板（§12.2）。模块目录里不放 CMakeLists |

原因：根文件是 generate-once 的，改了下次重新生成就丢；把 user 库全部收在 `User/` 目录里，重新生成也不受影响。

**一层一个 CMakeLists，不是一个模块一个**：`User/BSP/CMakeLists.txt` 一个文件里写 `bsp_gpio`、`bsp_iic` …… 全部 BSP 库；`User/Device/CMakeLists.txt` 里写全部 Device 库；`User/Common/CMakeLists.txt` 里写全部 Common 库。`add_subdirectory()` 只出现在 `User/CMakeLists.txt`，一行一层。模块目录（`BSP_GPIO/`、`Dev_Key/`、`Common_Ringbuf/`）里只有 `.c` 和 `.h`。

为什么不要模块级 CMakeLists：库清单与链接关系会散到目录树的每一层，改一条链接要顺着目录逐层翻；而且每个模块文件都得重复抄 `CMAKE_CURRENT_SOURCE_DIR`、`USER_WARN_FLAGS` 的可见性前提，抄漏一处就静默失效（`USER_WARN_FLAGS` 是目录作用域变量）。层内一个文件，本层的库边界一次读完。


`User/CMakeLists.txt` 骨架：

```cmake
# 1) 共用变量：必须写在所有 add_subdirectory 之前
set(USER_WARN_FLAGS -Wall -Wextra -Werror)

# 2) 注册各层：CMake 没有任何自动发现机制，漏一行这层就不参与编译
add_subdirectory(BSP)
add_subdirectory(Device)
add_subdirectory(Service)
add_subdirectory(APP)
add_subdirectory(TASK)

# 3) 主目标：只补生成文件里没写的 user 库
target_link_libraries(${CMAKE_PROJECT_NAME}
    app_key_cmd
    app_level
    svc_key_logic
    stm32cubemx
)
```

漏掉某一段的症状（都比较难查）：

| 漏了什么 | 症状 |
|---|---|
| `USER_WARN_FLAGS` 写在 `add_subdirectory` 之后 | `-Werror` 静默失效，一个字都不报（CMake 变量只对**之后**进入的子目录生效） |
| 少一个 `add_subdirectory` | 该层不编译，直到链接期才 `undefined reference` |
| 主目标少链一个库 | 它的 `-I` 到不了主目标的 `.c`：`fatal error: svc_key_logic.h: No such file or directory` |
| 主目标混用关键字签名 | `The plain signature for target_link_libraries has already been used with the target "frame"` |

`${CMAKE_PROJECT_NAME}` 就是顶层 `project()` 的名字（本例 `frame`），与字面量 `frame` 是同一个 target，别硬编码字符串。

### 12.2 模块模板：四行，一个都不能省

四行**写在所属层的那个 CMakeLists.txt 里**（`User/Device/CMakeLists.txt` 一条接一条放本层全部模块），不单独建模块级文件、不用 `add_subdirectory` 拆：

```cmake
add_library(dev_key STATIC
    Dev_Key/dev_key.c
)
target_include_directories(dev_key PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/Dev_Key)
target_link_libraries(dev_key PUBLIC bsp_gpio)
target_compile_options(dev_key PRIVATE ${USER_WARN_FLAGS})
```

- `target_include_directories` 永远 `PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/<模块目录>`：头目录是给上层用的，本模块也用它。
- `target_compile_options` 每个模块都要写。`USER_WARN_FLAGS` 是**普通变量**，靠目录作用域可见，不靠 target 继承；想省事用 `add_compile_options` 会连第三方一起打开警告。
- 库名 = **模块目录名全小写，层前缀一起带上**：`Dev_Key` → `dev_key`、`Dev_LED_Bar` → `dev_led_bar`、`Svc_Key_Logic` → `svc_key_logic`、`BSP_GPIO` → `bsp_gpio`、`Common_Ringbuf` → `common_ringbuf`。
  - 判据：`build/Debug/User/**/lib*.a` 与链接行读起来应当就是目录树，不用回查哪个库是哪个模块。
  - 不带前缀的代价是实打实的：`key` 与 C 库、`LED` 与宏名在链接行里分不清；`led` / `LED` / `dev_led` 三种拼法并存时，`target_link_libraries` 里写错一个字母，CMake 不报错，只在编译期变成"找不到头文件"（见 §12.11）。
  - **一个工程只有一种习惯**，不要 BSP 带前缀、Device 不带。
- 例外：模块只有 1 个 `.c` 且只被 1 处使用，可并入上层库，但要注释说明。
- **唯一可以省的一行是 `target_link_libraries`**，且只对"没有下层可链"的模块：Common 层的纯函数库（`common_ringbuf` 只 include `<stdint.h>` / `<stddef.h>`）、只有头文件的库。省这一行要在 CMakeLists 里写清理由。**不许为了凑四行链 `stm32cubemx`** —— 那会给 Common 带上 `-I` 与 `-DUSE_HAL_DRIVER`，把"本层能在 PC 宿主 gcc 上单独编译"这个判据直接废掉。

### 12.3 `PUBLIC` / `PRIVATE`：判据只有一个，看头文件

**看本模块的 `.h`，不看 `.c`。**

- 公开头里出现了下层的**类型名、宏、函数声明** → `PUBLIC`（否则上层拿到的是 incomplete type）。
- 只有本模块 `.c` 用 → `PRIVATE`。

一条 `PUBLIC` 必须能指到具体的行：

| 依据 | 结论 |
|---|---|
| `dev_led_bar.h:29` 有 `const bsp_gpio_t *const *pins;` | `dev_led_bar` 必须 `PUBLIC bsp_gpio` |
| `svc_key_logic.h:32` 有 `#include "dev_key.h"`、`:60` 有 `const dev_key_t *key;` | `svc_key_logic` 必须 `PUBLIC dev_key`，`PRIVATE` 的话 `app_key_cmd.c` 立刻报 `dev_key.h: No such file or directory` |

反查法（30 秒）：把 `PUBLIC` 临时改成 `PRIVATE` 重新编译，应当在上层某个 `.c` 报 `fatal error: xxx.h: No such file or directory`。**不报错**说明这条要么多余、要么本来就该 `PRIVATE`。

两个最常被误解的事实：

1. **`PUBLIC`/`PRIVATE` 只影响 include 可见性，不影响符号。** STATIC 库的 `PRIVATE` 依赖照样以 `$<LINK_ONLY:...>` 出现在最终链接行，符号照常解析。所以"改成 `PRIVATE` 会链接失败"是误判——真正会失败的是上层编译时找不到头。
2. **`PUBLIC` 会把 `-D` 一起上传。** 从 `stm32cubemx` 拿到的除了 `-I`，还有 `-DUSE_HAL_DRIVER -DSTM32F10x_xx`。也就是说 `PUBLIC stm32cubemx` 不只是"能看见头"，编译期分支也变了。"上层不碰 HAL"只能靠 `PRIVATE` 保证，靠自觉保证不了。

匿名 struct typedef 会逼你写 `PUBLIC`：

```c
typedef struct { void *port; uint16_t pin; uint8_t active_low; } bsp_gpio_t;
```

它没法前置声明，上层头文件里一出现 `bsp_gpio_t *` 就必须能看见定义。两条路：接受 include 传染（本项目选择，Device 头里带 `bsp_gpio.h`），或者给它加 struct tag 后各层头重复 `typedef struct bsp_gpio_s bsp_gpio_t;`（C11 允许重复 typedef，C99 报错）。

`PRIVATE` 是"每一跳"的事：`app_key_cmd --PRIVATE--> app_level` 只是截断传播，`frame` 因此看不见 `app_level.h`，所以 `freertos.c` 要用 `APP_Level_Init()` 时得靠 `app_key_cmd` 的链接行把它带进链接（`$<LINK_ONLY:>`），而 `-I` 不会跟着走。

### 12.4 每层链接什么（STM32 + CubeMX 定式）

| 库 | `PUBLIC` | `PRIVATE` | 理由 |
|---|---|---|---|
| `bsp_gpio` | — | `stm32cubemx` | HAL 只在 `.c`；头文件用 `void *` 挡住 |
| `dev_*` | `bsp_gpio` | — | 头文件里存 `const bsp_gpio_t *` 句柄 |
| `svc_*` | 仅当公开头暴露下层类型 | 下一层设备 + `stm32cubemx` | `.c` 用 `cmsis_os2.h` / 设备函数 |
| `app_*` | 仅当公开头暴露事件/设备类型 | 用到的 `svc_*` / `dev_*` | 业务头通常只有 `uint8_t` |
| `task_objects` | — | `app_*` + `stm32cubemx` + 自己 include 的层 | 线程体 include 什么就链什么 |

判据一句话：**实现全局一次（`.c` 只编一份），声明每个 target 各一次（`-I` 靠链接传递）。**

`stm32cubemx` 在 CubeMX 工程里不是真库，是 `INTERFACE` 库——`-I` 与 `-D` 的袋子，没有 `.a`。谁 include 了 HAL/FreeRTOS 头，谁就要能看见它；没 include 的层不要写（Device 就不需要，即使写了也不报错，只是"APP 不碰 HAL"这条边界被悄悄放宽）。

### 12.5 主目标上只写"真的需要"的

- 生成文件（`cmake/stm32cubemx/CMakeLists.txt`）已经链了 `stm32cubemx`、FreeRTOS 与 STM32_Drivers 的 OBJECT；`User/CMakeLists.txt` 只补它没写的 user 库。
- 主目标为什么需要 `svc_key_logic`：`Core/Src/freertos.c` 里 `sizeof(key_event_msg_t)` 建队列，会 include `svc_key_logic.h`。这个 `-I` 只能靠链接拿，`add_subdirectory` 不传递任何东西。
- 主目标为什么需要 `stm32cubemx`：`freertos.c` 用 HAL 与 FreeRTOS 的头。
- **`$<TARGET_OBJECTS:>` 不传播用法要求，也不建立链接边。** 所以 `task_objects` 里的 `PRIVATE app_key_cmd` 不会让命令表库进链接行——必须自己在主目标上再写一条。实测 `frame.dot` 里 `task_objects → app_key_cmd` 与 `frame → app_key_cmd` 是两条独立的边。
- 同一目标上 plain 与关键字签名不能混；一旦被 plain 用过（生成文件里就是），后面必须全 plain。
- 重复链同一个 INTERFACE 库无成本（CMake 去重），但语义上多余；重复链是"兜底"还是"忘了删"，注释里要写清楚。

### 12.6 TASK 用 OBJECT 库覆盖 `__weak`

```cmake
add_library(task_objects OBJECT task_key_scan.c task_key_cmd.c)
target_link_libraries(task_objects PRIVATE app_level app_key_cmd stm32cubemx)
target_compile_options(task_objects PRIVATE ${USER_WARN_FLAGS})
target_sources(${CMAKE_PROJECT_NAME} PRIVATE $<TARGET_OBJECTS:task_objects>)
```

为什么必须是 OBJECT：

- CubeMX 在 `freertos.c` 生成 `__weak void Task_Key_Scan(void *argument)`，`osThreadNew` 引用的就是这个名字。
- **`__weak` 也算"已定义"**，链接器不会为它去静态库里拉同名强符号；归档成员根本不会被取，任务线程空转，而编译器和链接器**一个错都不报**。
- 目标文件是强制链接的，强定义必定覆盖 `__weak`。
- 自检：`arm-none-eabi-nm` 看 `freertos.c.obj` 是 `W Task_Key_Scan`、`task_objects/.../task_key_scan.c.obj` 是 `T Task_Key_Scan`；`frame.map` 里地址落在 `.text`。若最终 `.elf` 里还是 `W` 那个 0x10 的小函数，任务就是空循环。

不写成 `target_link_libraries(frame PRIVATE task_objects)`：主目标已被生成文件用 plain 签名占用，混用直接 configure 失败。

不把 `.c` 直接塞进 `add_executable(frame ...)`：主目标会失去这些文件自己的 `-I` 与 `-Werror`，还得手抄一排依赖。

### 12.7 加东西的操作手册

| 要加什么 | 改哪里 |
|---|---|
| 已有模块加一个 `.c` | 该模块 `add_library(... <新 .c>)` 加一行 → 重配 |
| 新模块进已有层 | 建目录 + 文件 → 在层的 `CMakeLists.txt` 抄四行模板 → 谁用它谁加链接 → 重配 |
| 新层 | 建目录 + `CMakeLists.txt` → `User/CMakeLists.txt` 加 `add_subdirectory(<层>)`，放在依赖它的层**之前** |
| 新任务 | `.ioc` 加线程（入口名与函数同名、`As weak`）→ `TASK/` 加 `.c` → `task_objects` 加一行 → 重新生成并编译 |
| 新第三方 | 放进 `Third_Party/`，配适配器，`-w`，`PRIVATE` 链接到适配层 |

三条硬规则：

- **没有自动发现。** `add_subdirectory` 是唯一的注册方式；新层不写这一行，它静默不参与编译，直到链接期 `undefined reference`。指向不存在的目录则 configure 直接报 `add_subdirectory given source ... which is not an existing directory`。
- **不要 `file(GLOB)`。** 新增文件不重新 configure 就不生效，而且顺序按字母、依赖关系看不出来。宁可多写一行。
- **建了库就要有人链。** 观察 `build/Debug/User/<层>/lib*.a` 与 `build.ninja` 里的链接行：`lib` 有、链接行没有的库就是死代码（0 字节 `.c` 建的库尤其明显）。

### 12.8 第三方库与 `-w`

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

### 12.9 验证手法：别猜，看生成物

```powershell
# ① 某个 .c 到底拿到了哪些 -I / -D
$j = Get-Content build/Debug/compile_commands.json -Raw | ConvertFrom-Json
($j | Where-Object { $_.file -like '*app_level.c' } | Select-Object -First 1).command

# ② 最终链接行里有哪些 .a 与 .o
Select-String -Path build/Debug/build.ninja -Pattern 'libbsp_gpio'

# ③ 目标之间的依赖边（graphviz，可反复放大看 solid / dashed）
Get-Content build/Debug/frame.dot

# ④ 符号强弱：任务函数必须是 T，不能是 W
arm-none-eabi-nm build/Debug/CMakeFiles/frame.dir/Core/Src/freertos.c.obj   # 期望 W
arm-none-eabi-nm build/Debug/User/TASK/CMakeFiles/task_objects.dir/task_key_scan.c.obj  # 期望 T
```

判断删掉一行有没有副作用的最快办法：**重配 + 构建**。如果编译命令行逐字节没变，`ninja` 会直接说 `ninja: no work to do`——那就是零影响。

### 12.10 常见坑

- 改生成文件（根 `CMakeLists.txt`、`cmake/stm32cubemx/`）：重新生成就丢，且无法 review diff。
- 按模块拆 CMakeLists：`User/BSP/BSP_GPIO/CMakeLists.txt` + `Common/CMakeLists.txt` 里 `add_subdirectory(Common_Ringbuf)`。症状一是库清单散进目录树，二是 `USER_WARN_FLAGS` 这类目录作用域变量在新文件里"看得见却拿不到正确值"（经过额外的 `add_subdirectory` 后变量的继承链更容易断），三是没人链接的空目录 `add_subdirectory` 会直接 configure 失败：`add_subdirectory given source "Common_Ringbuf" which is not an existing directory`／`does not contain a CMakeLists.txt file`。正确做法：删掉模块级文件，把四行模板内联进层 CMakeLists。
- 只在一处写 plain、另一处写 `PRIVATE`：`The plain signature for target_link_libraries has already been used with the target "frame"`。
- 给 0 字节的 `.c` 建库：编出一个空库，没人链接，纯粹浪费一次 `ar`。要么填实现，要么先不建库。
- 以为 `add_subdirectory` 会把子目录的 `target_include_directories` 传给父目录：不会，只传递"链接关系"。
- 以为 `PUBLIC`/`PRIVATE` 管符号：不管，只管 `-I`/`-D`；符号靠链接行。
- 以为 `$<TARGET_OBJECTS:>` 会把 OBJECT 库的依赖一起带过来：不会，只搬 `.o`。
- 找不到头先查 `compile_commands.json` 里该文件的 `-I`，别先怀疑代码。
- 链接顺序不用手管（CMake 按依赖拓扑排序），但别手动往链接行塞 `.a`。
- `build/` 里的陈旧 `.a`（改名/删模块之前留下的）会让人以为模块还在；换结构后清一次 `build`。

### 12.11 目标名对不上，报的却是"找不到头文件"

`target_link_libraries(app_key_cmd PRIVATE dev_led)` 里的 `dev_led` 如果不存在（实际叫 `LED`、`led`、`dev_LED`，或者模块还没建），CMake **configure 不报错**：plain 名字被当成外部库，只在最终链接行变成 `-ldev_led`。后果是整个依赖什么都不传——`-I`、`-D` 都不到。

症状是上层某个 `.c` 报：

```
app_key_cmd.c:27:10: fatal error: dev_led.h: No such file or directory
```

看起来像 include 路径问题，其实是**库名问题**。排查三步：

1. 列出全部目标名，与每条链接行**逐字**比对（大小写也要比）：
   ```powershell
   Select-String -Path User/*/CMakeLists.txt -Pattern 'add_library\('
   ```
2. 看这个 `.c` 到底拿到了哪些 `-I`，缺的那条正是没传进来的依赖：
   ```powershell
   $j = Get-Content build/Debug/compile_commands.json -Raw | ConvertFrom-Json
   ($j | Where-Object { $_.file -like '*app_key_cmd.c' } | Select-Object -First 1).command
   ```
3. 往编译期之后才会露头的版本：`cannot find -ldev_led`，或 `ninja: error: 'libdev_led.a', needed by 'frame.elf', missing`。

第二类同名错是**句柄名**：`Dev_LED_Toggle(&dev_led)` 里的 `dev_led` 是设备句柄，不是库名，实际声明是 `extern const dev_led_t dev_led_1;`。编译器直接报 `undeclared`，但改库名时最容易顺手一起改错。判据：**库名 = 模块目录名，句柄名 = `.c` 里实例化的那个变量**，两者只是巧合同前缀。

### 12.12 改目标名之后必须做的三件事

1. **重新 Configure，不是只 Build。** `build/*/build.ninja` 与 `CMakeCache.txt` 里存着旧目标名和旧的 `-l`，不重配就继续找 `-lLED`。IDE 里点一次生成通常会自动重配；不确定就手动 Re-configure。
2. **认陈旧归档。** `build/*/User/**/lib<旧名>.a` 还躺在原地，`ls`/`nm` 会让人以为没改成功。改名后要么删 `build`，要么确认新 `lib<新名>.a` 已生成且旧的不再被链接行引用。
3. **全仓库搜旧名**，注意词边界与大小写敏感：
   ```powershell
   Select-String -Path User/*/CMakeLists.txt,User/**/*.c,User/**/*.h -Pattern '\b(led_bar|key|led)\b'
   ```
   只改大小写（`LED` → `led`）时，PowerShell 的 `-eq` 不敏感，脚本判断要用 `-cne`。

**不碰 IDE 构建目录的全量验证**（改链接结构后最值得跑一次）：

```powershell
$tc    = "$env:LOCALAPPDATA\stm32cube\bundles\gnu-tools-for-stm32\13.3.1+st.9\bin"
$env:PATH = "$tc;$env:PATH"        # 工具链文件里写的是裸名 arm-none-eabi-gcc，不在 PATH 就 configure 失败
$cm    = "$env:LOCALAPPDATA\stm32cube\bundles\cmake\4.2.3+st.1\bin\cmake.exe"
$ninja = "$env:LOCALAPPDATA\stm32cube\bundles\ninja\1.13.2+st.1\bin\ninja.exe"
$b     = "$env:TEMP\frame_cfg"

& $cm -S . -B $b -G Ninja "-DCMAKE_MAKE_PROGRAM=$ninja" `
     "-DCMAKE_TOOLCHAIN_FILE=$PWD/cmake/gcc-arm-none-eabi.cmake" -DCMAKE_BUILD_TYPE=Debug
& $ninja -C $b
& "$tc\arm-none-eabi-size.exe" "$b\frame.elf"
```

- 用 IDE 自带的 cmake/ninja/gcc，版本与 IDE 一致。**不要**用 `cube-cmake --build`：这台机器上它在 configure 阶段会静默挂住（无输出、不退出）。
- 构建目录放 `%TEMP%`，`build/Debug` 一个字不动，IDE 的增量缓存不受影响。
- 通过标准：`configure exit 0` → `[N/N] Linking C executable frame.elf` → `size` 打出 RAM/FLASH 占比。
- 确认某个库**真的进了固件**（不只是"编过了"）：
  ```powershell
  & "$tc\arm-none-eabi-nm.exe" "$b\frame.elf" | Select-String 'dev_led'
  ```
  期望同时看到 `T Dev_LED_Write`（函数在 `.text`）与 `R dev_led_1`（`const` 设备句柄在 `.rodata`，不占 RAM）。只看到 `T` 没有 `R`，说明那个句柄没被引用、链接器把它丢了。

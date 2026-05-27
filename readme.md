# STM32 外部烧录 — VSCode Tasks 使用指南

通过 VSCode 任务系统实现多项目的**编译 + 烧录**自动化，同时支持 **ST-Link** 和 **J-Link** 两种调试器。

本文档不依赖特定项目，以下所有示例中的 `MyProject` 仅作示意，请替换为你的实际项目名。

## 工作模式

根据你的项目组织方式，选择以下两种模式之一：

### 模式一：单根工作区（单文件夹）

所有项目放在同一个文件夹下，共用一个 `.vscode/tasks.json`。

```
STM32/
├── .vscode/
│   └── tasks.json            ← 所有项目任务写在这里
├── ProjectA/
│   ├── build/Debug/ProjectA.elf
│   └── flash.jlink
├── ProjectB/
│   ├── build/Debug/ProjectB.elf
│   └── flash.jlink
└── ...
```

- 打开 `STM32/` 文件夹即为单根模式
- `Ctrl+Shift+B` 列出当前文件夹下所有项目的 build 任务
- 适合项目在同一仓库或同一目录下的场景

### 模式二：多根工作区（多文件夹）

项目分散在不同目录，通过 `.code-workspace` 文件将它们组合到一起。

```
STM32/
├── 项目工作区.code-workspace   ← 定义包含哪些文件夹
├── ProjectA/
│   ├── .vscode/tasks.json      ← 项目任务写在这里
│   ├── build/Debug/ProjectA.elf
│   └── flash.jlink
└── ProjectB/
    ├── .vscode/tasks.json      ← 项目任务写在这里
    ├── build/Debug/ProjectB.elf
    └── flash.jlink
```

**用法步骤：**

1. 在父目录创建 `.code-workspace` 文件（如 `STM32.code-workspace`）：

```json
{
    "folders": [
        { "name": "ProjectA", "path": "STM32/ProjectA" },
        { "name": "ProjectB", "path": "STM32/ProjectB" }
    ]
}
```

2. 每个项目文件夹各自维护自己的 `.vscode/tasks.json`
3. 双击 `.code-workspace` 文件打开，VSCode 自动合并所有根的 tasks
4. `Ctrl+Shift+B` 列出所有项目所有烧录方式的选项

### 两种模式对比

| | 单根工作区 | 多根工作区 |
|--|-----------|-----------|
| tasks.json 位置 | 根目录 `.vscode/tasks.json`，统一管理 | 每个项目各自的 `.vscode/tasks.json`，各自独立 |
| 配置文件数量 | 一个 JSON 管理所有项目 | 每个项目一个 JSON |
| 打开方式 | 直接打开文件夹 | 双击 `.code-workspace` 文件 |
| 任务来源 | 一个文件全部列出 | 多根自动合并 |
| Ctrl+Shift+B | 列当前根下的 build 任务 | 列所有根下所有 build 任务 |
| 适用场景 | 项目在同一仓库下 | 项目分散在不同路径 |

## 任务命名规范

每个项目固定包含 6 个任务，命名格式：

```
{项目名}: {操作} ({烧录器})
```

| 任务 | 功能 |
|------|------|
| `MyProject: configure` | 运行 `cube-cmake --preset Debug` 生成 build 目录 |
| `MyProject: build` | 编译生成 .elf |
| `MyProject: flash (ST-Link)` | ST-Link 烧录 |
| `MyProject: build + flash (ST-Link)` | 先编译，再 ST-Link 烧录 |
| `MyProject: flash (J-Link)` | J-Link 烧录 |
| `MyProject: build + flash (J-Link)` | 先编译，再 J-Link 烧录 |

> 所有 `build + flash` 组合任务都标记为 `"isDefault": true`，这样 `Ctrl+Shift+B` 会弹出选择菜单，让你挑烧录哪个程序、用什么烧录器。如果觉得选项太多，去掉不需要的 `isDefault` 即可。

## 使用方式

| 操作 | 方法 |
|------|------|
| 选择任务编译+烧录 | `Ctrl+Shift+B` → 从菜单中挑选任务 |
| 运行任意任务 | `Ctrl+Shift+P` → `Tasks: Run Task` → 搜索任务名 |
| 仅编译（不烧录） | 运行 `MyProject: build` |
| 仅烧录（不编译） | 运行 `MyProject: flash (ST-Link)` 或 `flash (J-Link)` |

## 前置工具

| 工具 | 用途 | 配置位置 |
|------|------|----------|
| `cube-cmake` | CMake 构建系统 | 系统 PATH |
| STM32CubeProgrammer CLI | ST-Link 烧录 | tasks.json 中指定路径 |
| JLink.exe | J-Link 烧录 | tasks.json 中指定路径 |

## 烧录参数

### ST-Link

```
STM32_Programmer_CLI.exe -c port=SWD -d <xxx.elf> -v -rst
```

| 参数 | 含义 |
|------|------|
| `-c port=SWD` | SWD 接口连接 |
| `-d <文件>` | 指定 .elf 文件 |
| `-v` | 烧录后校验 |
| `-rst` | 烧录完成自动复位 |

### J-Link

```
JLink.exe -CommanderScript <flash.jlink>
```

每个项目需要一个 `flash.jlink` 脚本，JLink 通过该脚本自动执行烧录流程：

```
device STM32F103C6
si SWD
speed 4000
connect
r
loadfile build/Debug/MyProject.elf
r
g
exit
```

> JLink 的 `loadfile` 直接支持 `.elf`，无需转为 `.hex`。

## 添加新项目

以 `tasks.json` 中的任务为模板，复制以下 6 个任务块，将 `MyNewProject` 替换为实际项目名：

```json
{
    "label": "MyNewProject: configure",
    "type": "shell",
    "command": "cube-cmake",
    "args": ["--preset", "Debug"],
    "options": { "cwd": "${workspaceFolder}" },
    "problemMatcher": []
},
{
    "label": "MyNewProject: build",
    "dependsOn": ["MyNewProject: configure"],
    "type": "shell",
    "command": "cube-cmake",
    "args": ["--build", "${workspaceFolder}/build/Debug", "--"],
    "problemMatcher": []
},
{
    "label": "MyNewProject: flash (ST-Link)",
    "type": "shell",
    "command": "D:\\a3\\STM32CubeProgrammer\\bin\\STM32_Programmer_CLI.exe",
    "args": ["-c", "port=SWD", "-d", "${workspaceFolder}/build/Debug/MyNewProject.elf", "-v", "-rst"],
    "problemMatcher": []
},
{
    "label": "MyNewProject: build + flash (ST-Link)",
    "dependsOn": ["MyNewProject: build", "MyNewProject: flash (ST-Link)"],
    "dependsOrder": "sequence",
    "group": { "kind": "build", "isDefault": true }
},
{
    "label": "MyNewProject: flash (J-Link)",
    "type": "shell",
    "command": "D:\\a3\\JLink_V930\\JLink.exe",
    "args": ["-CommanderScript", "${workspaceFolder}/flash.jlink"],
    "problemMatcher": []
},
{
    "label": "MyNewProject: build + flash (J-Link)",
    "dependsOn": ["MyNewProject: build", "MyNewProject: flash (J-Link)"],
    "dependsOrder": "sequence",
    "group": { "kind": "build", "isDefault": true }
}
```

- **单根模式**：直接将模板追加到根目录下的 `.vscode/tasks.json`
- **多根模式**：将模板写入新项目文件夹自己的 `.vscode/tasks.json`
参考文件.vscode/tasks.json

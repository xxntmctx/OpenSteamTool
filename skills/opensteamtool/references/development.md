# OpenSteamTool 编译构建与调试指南

本篇面向开发者与 AI 助手，指导如何从源码编译 OpenSteamTool、进行本地调试以及分析日志。

---

## 1. 环境准备

- **操作系统**：Windows 10 / 11 (x64)
- **构建工具**：CMake 3.20 或更高版本
- **编译器**：Visual Studio 2022 (具有 MSVC v143 - C++ x64 构建工具组件)
- **编译时网络**：由于 CMakeLists.txt 会利用 `FetchContent` 动态拉取并编译依赖项 (包括 Lua, spdlog, tomlplusplus 等，缓存于 `<项目根目录>/.deps`），确保网络畅通。

---

## 2. 编译与构建

### 快速构建 (推荐)
在项目根目录下，双击或在 PowerShell 中直接运行构建脚本：
```powershell
.\build.bat
```
该脚本会自动调用 CMake 配置并分别生成 Debug 与 Release 构建版本。

### 手动构建步骤
如果你需要自定义构建选项，可以在项目根目录执行：
```bash
# 生成工程配置
cmake -B build -S . -DCMAKE_BUILD_TYPE=Release
# 执行编译
cmake --build build --config Release
```

### 输出产物
构建成功后，生成的二进制文件将输出至 `build/` 目录下：
- **Debug 版**：`build/Debug/OpenSteamTool.dll`, `build/Debug/dwmapi.dll`, `build/Debug/xinput1_4.dll`
- **Release 版**：`build/Release/OpenSteamTool.dll`, `build/Release/dwmapi.dll`, `build/Release/xinput1_4.dll`
- **凭证提取工具**：`build/tools/Release/extract_tickets.exe` (用于在正版电脑上转储游戏的 AppTicket/ETicket)

---

## 3. 本地部署与测试

1. 确保 Steam 客户端已完全退出。
2. 将构建出的 `dwmapi.dll`、`xinput1_4.dll` 和 `OpenSteamTool.dll` 三个文件复制到 Steam 的安装根目录下（与 `steam.exe` 处于同级目录）。
3. 如果需要使用自定义配置，可将项目根目录下的 `opensteamtool.example.toml` 重命名为 `opensteamtool.toml`，并同样放到 Steam 根目录下。
4. 创建 Lua 配置目录：在 Steam 目录下新建路径 `config\lua\`（例如 `C:\Program Files (x86)\Steam\config\lua\`），并将你的解锁 Lua 脚本放在此处。
5. 启动 Steam 客户端。核心劫持 DLL 会自动将 `OpenSteamTool.dll` 载入，控制台或调试输出将开始工作。

---

## 4. 调试日志诊断 (Debug 构建专用)

当使用 **Debug 版本** 部署时，OpenSteamTool 将在 `<Steam安装根目录>\opensteamtool\` 文件夹下自动创建多个功能专用的日志文件。当解锁功能异常时，可以通过这些日志快速定位问题：

| 日志文件名 | 记录模块与来源 | 核心诊断内容 |
| :--- | :--- | :--- |
| **`main.log`** | 通用初始化与配置加载 | DLL 加载流程、Lua 脚本解析错误、模块生命周期 |
| **`ipc.log`** | `Pipe` / `Hooks_IPC` | 拦截到的 Steam 进程 IPC 握手、IClientUser 接口调用细节 |
| **`netpacket.log`** | `Hooks_NetPacket` | WebSocket 网络数据包的序列化/反序列化及修改过程 |
| **`manifest.log`** | `Hooks_Manifest` | 游戏 Manifest 绑定（BuildDepotDependency）的注入和修改 |
| **`decryptionkey.log`** | `Hooks_Decryption` | Depot 自动解密密钥的拦截与注入，以及缓存凭证的加载 |
| **`achievement.log`** | 统计与成就模块 | 玩家成就数据（UserStats）的拉取和 SteamID 欺骗行为 |
| **`pipe.log`** | 命名管道与平台助手 | 游戏进程的库注入 (GameHook)、Denuvo 授权管道握手状态 |
| **`steamui.log`** | SteamUI 渲染模块 | Steam 网页版 UI (CEF) 钩子的诊断与执行状态 |

> [!TIP]
> 日志级别可在 `opensteamtool.toml` 文件的 `[log] level` 字段中配置（可选值：`trace`, `debug`, `info`, `warn`, `error`）。在调试崩溃或 Hook 不生效时，建议将级别设为 `trace`。

---

## 5. 新增内存 Hook 开发规范

如果 Steam 更新了某些 API，或者你想为项目增加新的绕过功能：
1. **Hook 宏规范**：在 `src/Hook/HookMacros.h` 中包含丰富的 MinHook 封装宏。使用 `HOOK_FUNC(FuncName, ReturnType, ...)` 定义你自己的 Hook 函数。
2. **生命周期注册**：在 `src/Hook/` 目录下建立你的 `Hooks_<Feature>.cpp`，并提供 `Install()` 与 `Uninstall()` 函数。
3. **启用挂载**：在 [src/Hook/HookManager.cpp](file:///c:/Users/naitangmao/Documents/GitHub/OpenSteamTool/src/Hook/HookManager.cpp) 的 `CoreHook` 与 `CoreUnhook` 列表里，添加对应 `Hooks_<Feature>::Install()` 和 `Hooks_<Feature>::Uninstall()`，从而使其在线程启动时被自动挂载。

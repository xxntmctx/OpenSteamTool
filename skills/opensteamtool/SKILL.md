---
name: opensteamtool
description: Assist in developing, debugging, configuring and maintaining OpenSteamTool, an open-source Steam unlocker written in C++.
---

# OpenSteamTool 开发者助手指南

本技能卡用于帮助开发者与 AI 助手快速理解、调试与维护 **OpenSteamTool** —— 一个基于 C++ 20 编写的开源 Steam 游戏/DLC 解锁及辅助工具。

## 项目概览

OpenSteamTool 通过在 Windows 环境下劫持 Steam 进程的加载，在运行时动态 Hook 内存，从而实现未拥有游戏的解锁、DLC 的授权、Denuvo/SteamStub 的凭证欺骗、家庭共享绕过以及成就/统计伪造。

## 目录与职责划分

以下是项目核心代码目录结构及各模块的职责说明：

- [src/dwmapi/](file:///c:/Users/naitangmao/Documents/GitHub/OpenSteamTool/src/dwmapi) 和 [src/xinput1_4/](file:///c:/Users/naitangmao/Documents/GitHub/OpenSteamTool/src/xinput1_4): Proxy DLL (劫持 DLL)。通过导出与系统 DLL 相同的函数，在 Steam 启动时诱导 `steam.exe` 加载它们，随后利用 `DllMain` 将核心 DLL `OpenSteamTool.dll` 注入进程。
- [src/Hook/](file:///c:/Users/naitangmao/Documents/GitHub/OpenSteamTool/src/Hook): Hook 管理器与具体拦截逻辑。
  - [HookManager.cpp](file:///c:/Users/naitangmao/Documents/GitHub/OpenSteamTool/src/Hook/HookManager.cpp): 负责所有子 Hook 模块的统一安装和卸载。
  - `Hooks_NetPacket`: WebSocket 网络包拦截点（核心拦截）。
  - `Hooks_IPC`: Steam 跨进程 IPC 拦截（控制 IClientUser/IClientUtils 等核心接口）。
  - `Hooks_Decryption`: 仓库 (Depot) 解密密钥自动注入。
  - `Hooks_Manifest`: 拦截并重写游戏 Manifest 关联逻辑。
- [src/Pipe/](file:///c:/Users/naitangmao/Documents/GitHub/OpenSteamTool/src/Pipe): 命名管道子系统。管理与外部进程（如注入的游戏）之间的加密/握手通信，同时对游戏进程进行检测，管理 Denuvo AppTicket 存储。
- [src/Utils/](file:///c:/Users/naitangmao/Documents/GitHub/OpenSteamTool/src/Utils): 工具类与逻辑支持。
  - `LuaConfig`: 负责初始化 Lua 虚拟机并导出自定义配置函数。
  - `SteamMetadata`: 实现远程 Pattern 特征码、Manifest API 和 Stats 数据的网络拉取。
  - `Config`: `opensteamtool.toml` 文件的读取与热监视。

## 开发核心约束与设计原则

接管本项目开发或进行修改时，必须遵守以下核心约束：

### 1. 规避 Loader Lock（加载器锁）
- **原则**：绝对不能在 `DllMain` 的 `DLL_PROCESS_ATTACH` 阶段直接进行文件 I/O、加载其他 DLL、拉取网络数据或初始化 Detours/MinHook。
- **实现**：在 `DllMain` 中仅使用 `OSTPlatform::Thread::StartDetached` 启动异步工作线程，将所有的核心初始化挂载到 `InitThread` 运行。

### 2. 动态特征码定位（Pattern Matching）
- **原则**：OpenSteamTool 不在代码中硬编码任何 `steamclient64.dll` 和 `steamui.dll` 的内存偏移。
- **实现**：每次启动时计算这两个 DLL 的 SHA-256 哈希值，从云端 `OpenSteam001/steam-monitor` 仓库（`pattern` 分支）拉取对应的匹配 TOML 特征码文件。修改 Hook 点时应对应更新特征码配置或本地测试缓存。

### 3. Git 提交与发布规范
- **非交互式 Git 提交**：在执行 `git commit` 时必须加上关闭签名校验的参数，以防 Pinentry 口令弹窗卡死 CI 或非交互终端：
  ```bash
  git -c commit.gpgsign=false commit -m "commit message"
  ```
- **配置隔离**：本地调试变量应写入 `.dev.vars`，且该文件必须被 git 忽略，绝对不能将敏感密钥（如 API keys, AppTickets 等）直接硬编码或提交至仓库。

## 深入参考文档

为了解更多细节，请阅读以下子文档：

1. [核心架构与原理说明](file:///c:/Users/naitangmao/Documents/GitHub/OpenSteamTool/skills/opensteamtool/references/architecture.md)
   - 详细解析 DLL 劫持原理、特征码拉取匹配机制、WebSocket 与 IPC 包重写，以及 Denuvo/SteamStub 的授权伪造原理。
2. [Lua 配置与 API 指南](file:///c:/Users/naitangmao/Documents/GitHub/OpenSteamTool/skills/opensteamtool/references/lua_api.md)
   - 包含完整的 Lua 脚本函数手册、参数含义、热重载支持及常见解锁 Lua 脚本编写范例。
3. [构建与调试指南](file:///c:/Users/naitangmao/Documents/GitHub/OpenSteamTool/skills/opensteamtool/references/development.md)
   - 如何在 Windows 下进行 CMake 编译、日志诊断分析以及如何添加自定义内存 Hook。

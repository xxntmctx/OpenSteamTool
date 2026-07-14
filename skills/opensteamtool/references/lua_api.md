# OpenSteamTool Lua API 配置与使用指南

OpenSteamTool 的游戏解锁、仓库解密密钥和票据管理逻辑完全由 Lua 脚本定义。本项目包含一个内置的 Lua 虚拟机，能够加载、解析并监视指定的脚本目录。

---

## 1. 核心特性

- **大小写不敏感**：底层的 Lua 元表（Metatable）会拦截全局环境 `_G` 的未知函数查找，并将函数名统一转换为全小写。因此，`addappid`、`AddAppId`、`ADDAPPID` 均等效。
- **文件热重载**：`LuaFileWatcher` 模块会实时监视 Lua 目录。一旦 `.lua` 文件被创建、编辑、覆盖或删除，系统会自动清理之前的缓存并重新解析脚本，**无需重启 Steam**，无需切换离线/在线模式。
- **加载顺序**：默认的加载目录为 `<Steam安装目录>\config\lua\`，并在之后加载 `opensteamtool.toml` 中自定义配置的额外 `paths`。若有冲突，用户自定义目录和主目录的设置优先级更高。

---

## 2. API 函数清单

### `addappid`
- **签名**：`addappid(app_or_depot_id [, unused, depot_key])`
- **参数**：
  - `app_or_depot_id` (integer): 需要解锁的游戏 AppId 或仓库 DepotId。
  - `unused` (integer, 可选): 占位符，通常传 `0`。
  - `depot_key` (string, 可选): 64 字节长的十六进制仓库解密密钥。如果提供，会自动注入用于该 Depot 文件的下载与解密。
- **示例**：
  ```lua
  -- 仅解锁 AppId 1361510
  addappid(1361510)
  -- 解锁 Depot 1361511 并注入解密密钥
  addappid(1361511, 0, "5954562e7f5260400040a818bc29b60b335bb690066ff767e20d145a3b6b4af0")
  ```

---

### `addtoken`
- **签名**：`addtoken(app_id, token_string)`
- **参数**：
  - `app_id` (integer): 需要访问令牌的保护游戏 AppId。
  - `token_string` (string): 十进制的 uint64 访问令牌字符串。
- **示例**：
  ```lua
  addtoken(1361510, "2764735786934684318")
  ```

---

### `setmanifestid`
- **签名**：`setmanifestid(depot_id, gid_string [, size])`
- **参数**：
  - `depot_id` (integer): 目标仓库的 DepotId。
  - `gid_string` (string): 锁定版本的十进制 Manifest GID 字符串。
  - `size` (integer, 可选): 指定的 Manifest 大小。在底层，为了避免因不正确的 size 破坏 Steam 下载器，该值一般在内部被重新校验（或强制保留为原有 size / 重设为 0）。
- **作用**：常用于锁定特定的 Manifest 版本，防止游戏升级，或强制回滚游戏版本。
- **示例**：
  ```lua
  setmanifestid(1361511, "5656605350306673283")
  ```

---

### `setappticket`
- **签名**：`setappticket(app_id, ticket_hex)`
- **参数**：
  - `app_id` (integer): Denuvo / SteamStub 保护的游戏 AppId。
  - `ticket_hex` (string): 提取出的原始所有权 AppTicket 二进制内容的十六进制字符串。
- **作用**：写入本地凭据存储，使 Denuvo/SteamStub 验证能够顺利通过。
- **示例**：
  ```lua
  setappticket(1361510, "1400000008000000...")
  ```

---

### `seteticket`
- **签名**：`seteticket(app_id, eticket_hex)`
- **参数**：
  - `app_id` (integer): 游戏 AppId。
  - `eticket_hex` (string): 加密票据 ETicket 二进制的十六进制字符串。
- **示例**：
  ```lua
  seteticket(1361510, "02000000a12b...")
  ```

---

### `setstat`
- **签名**：`setstat(app_id, steamid_string)`
- **参数**：
  - `app_id` (integer): 游戏 AppId。
  - `steamid_string` (string): 想要拉取成就/统计数据源的 SteamID (十进制 uint64 字符串)。
- **作用**：当该 AppId 未配置 `setstat` 且 `[stats] enable_api = true` 时，OpenSteamTool 会查询 `https://stats.opensteamtool.com/{appid}`，若仍然失败则使用默认内置 SteamID（`76561198028121353`）。
- **示例**：
  ```lua
  setstat(1361510, "76561197960287930")
  ```

---

### `http_get` / `http_post`
- **签名**：
  - `http_get(url [, headers])` -> `(body_string, status_code)`
  - `http_post(url, post_body [, headers])` -> `(body_string, status_code)`
- **参数**：
  - `url` (string): 请求的目标 HTTP/HTTPS 地址。
  - `post_body` (string): POST 请求的载荷内容。
  - `headers` (table, 可选): 请求头表，例如 `{["Accept"]="application/json"}`。
- **示例**：
  ```lua
  local headers = { ["Authorization"] = "Bearer token123" }
  local body, status = http_get("https://api.example.com/get-manifest-code", headers)
  if status == 200 then
      print("Success: " .. body)
  end
  ```

---

## 3. 典型解锁 Lua 脚本范例

这是一个综合配置示例，通常命名为 `game_unlock.lua` 并保存到 `<Steam安装目录>\config\lua\` 下：

```lua
-- 1. 声明并解锁主游戏及 DLC 的 AppId
addappid(1361510) -- 某游戏主程序 AppId
addappid(1361511) -- 某游戏 DLC 1
addappid(1361512) -- 某游戏 DLC 2

-- 2. 注入仓库解密密钥 (Depot Decryption Key)
addappid(1361513, 0, "8a6e87f8a848a6efb08d4b3b27b9c9f0c767e20d145a3b6b4af022f3e8f8101a")

-- 3. 添加游戏的访问令牌 (App Access Token)
addtoken(1361510, "18264583920194857321")

-- 4. 锁定特定 Depot 版本的 Manifest
setmanifestid(1361513, "8945620938210398457")

-- 5. 配置 Denuvo 所有权凭证 (提取的 AppTicket & ETicket)
setappticket(1361510, "140000000800000001000000000000008fc0f8...")
seteticket(1361510, "02000000b050c265df0e7b8f9a...")

-- 6. 重定向成就同步的 SteamID
setstat(1361510, "76561198028121353")
```

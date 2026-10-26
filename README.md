# opencode-workbuddy-auth-jsonimport

在官方插件 **`@magpie-community/opencode-workbuddy-auth@0.1.36`** 基础上，**新增第三种登录方式：「导入登录 JSON」**。其余行为与官方版一致，但用独立的供应商 id（见下），可与官方插件共存。


## 供应商 id（与官方插件共存）

为避免和官方 `@magpie-community/opencode-workbuddy-auth` 冲突，本 fork 用**独立的供应商 id**：

| 本插件 | 官方插件 |
|---|---|
| `workbuddy-import`（显示名 `WorkBuddy (JSON 导入)`） | `workbuddy` |
| `workbuddy-ai-import`（显示名 `WorkBuddy AI (JSON 导入)`） | `workbuddy-ai` |

`authID`、`platform`、接口地址与官方一致，只换了供应商 id 和显示名，因此两者可同时安装。

## 新增：导入登录 JSON

官方版原有两种登录：
1. `… account (browser)` —— 浏览器登录
2. `… desktop's sign-in` —— 读 WorkBuddy 桌面端已登录的文件

本版新增第三种：
3. **`… 导入登录 JSON`** —— 弹出粘贴框（`method: "code"`），把 WorkBuddy 的登录 JSON 贴进去。

适用场景：这台机器没装 WorkBuddy 桌面端、或要把一个登录搬到另一台机器时。

### 粘贴什么

WorkBuddy 桌面端 auth 目录下那个 `.info` 文件的**完整内容**，形如：

```json
{
  "account": { "enterpriseId": "", "nickname": "177****1446", "uid": "..." },
  "auth": { "accessToken": "...", "refreshToken": "...", "domain": "www.codebuddy.cn", "expiresAt": 1795442400, "realm": "cn" }
}
```

也接受只有 token 的扁平对象（`{ "accessToken": "...", "refreshToken": "...", "uid": "...", ... }`）。

文件位置：
- Windows：`%LOCALAPPDATA%\CodeBuddyExtension\Data\Public\auth\workbuddy-desktop.info`（国际版 `workbuddy-desktop-ai.info`）
- macOS：`~/Library/Application Support/CodeBuddyExtension/Data/Public/auth/…`
- Linux：`~/.local/share/CodeBuddyExtension/Data/Public/auth/…`

### 与「读桌面端文件」的区别

- 「读桌面端文件」每次请求都去读 app 的文件，token 由 app 维护，magpie 不续期、不写回。
- 「导入登录 JSON」把粘贴的登录**存成普通 oauth 登录**，magpie 会像浏览器登录一样**自动续期**。
  因此它的 refresh token 归这个账号所有——**同一个登录只在一台机器上导入使用**，别和 app 或别处同时用，先用的会把后用的挤下线。
- WorkBuddy 5.6+ 的加密登录（`$wbEncrypted`）读不了，会明确报错，让你改用浏览器登录。

## 安装

magpie 窗口 → 插件页 → 添加插件 → 填文件夹路径：

    D:\Ai\magpie-plugins\workbuddy-auth-jsonimport

然后在 Providers → WorkBuddy 那一行选「导入登录 JSON」，粘贴登录 JSON。

## 改动说明（相对官方 0.1.36）

`index.mjs` 只加了两段、未改原逻辑：
- 新函数 `parseImport()`（解析粘贴的 JSON）和 `importSignIn()`（第三种登录方式）。
- 在 `auth.methods` 数组末尾加了第三项。
- `_internal` 导出里加了 `importSignIn`、`parseImport`，方便测试。

升级官方版时，把这两段重新应用到新版本即可。

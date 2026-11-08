# opencode-workbuddy-auth-jsonimport

在官方插件 **`@magpie-community/opencode-workbuddy-auth@0.2.20`** 基础上，**新增一种登录方式：「导入登录 JSON」**。其余行为与官方版一致，但用独立的供应商 id（见下），可与官方插件共存。

> 基线版本：官方 `0.2.20`（2026-11-03）。本包版本 `0.2.20-import.2`（import.2 新增**批量导入**）。升级官方版时，把下面「改动说明」里的几处重新应用到新版本即可。

## 供应商 id（与官方插件共存）

为避免和官方 `@magpie-community/opencode-workbuddy-auth` 冲突，本 fork 用**独立的供应商 id**：

| 本插件 | 官方插件 |
|---|---|
| `workbuddy-import`（显示名 `WorkBuddy (JSON 导入)`） | `workbuddy` |
| `workbuddy-ai-import`（显示名 `WorkBuddy AI (JSON 导入)`） | `workbuddy-ai` |

`authID`、`platform`、接口地址与官方一致，只换了供应商 id 和显示名，因此两者可同时安装。

## 登录方式

官方 0.2.20 已有三种登录：

1. `… account (browser)` —— 浏览器登录
2. `… desktop's sign-in` —— 读 WorkBuddy 桌面端已登录的文件
3. `… API key` —— 填在 WorkBuddy 后台生成的 API key

本版新增第四种：

4. **`… 导入登录 JSON`** —— 弹出粘贴框（`method: "code"`），把 WorkBuddy 的登录 JSON 贴进去。

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

**批量导入（import.2 新增）**：直接粘贴一份 `workbuddy-accounts` 批量导出，一次导入多个账号：

```json
{ "format": "workbuddy-accounts", "version": 1, "accounts": [
  { "uid": "...", "nickname": "198****1251", "enterpriseId": "", "accessToken": "...", "refreshToken": "...", "expiresAt": 1796004686, "domain": "www.codebuddy.cn" },
  { "uid": "...", "nickname": "192****5004", "accessToken": "...", "refreshToken": "...", "expiresAt": 1796107327 }
] }
```

- 也接受裸数组 `[ {账号}, {账号}, ... ]`。
- 第一个账号作为本次登录结果，其余用 `client.auth.set` 存到 `workbuddy-import#<uid 哈希>`：**按 uid 去重，重复导入同一账号落同一槽位，不会产生重复条目**。
- `expiresAt` 兼容秒和毫秒（小于 1e12 视为秒，自动 ×1000）；`domain` 兼容 markdown 链接写法 `[www.codebuddy.cn](...)`，自动取文本。
- 缺 `accessToken` / `uid` 的、以及加密登录（`$wbEncrypted`）的账号自动跳过并在结果里列出原因。

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

magpie 窗口 → 插件页 → 添加插件 → 填本插件文件夹路径（例如把本包解压到）：

    D:\Ai\magpie-plugins\workbuddy-auth-jsonimport

然后在 Providers → WorkBuddy 那一行选「导入登录 JSON」，粘贴登录 JSON。

> Docker 部署 magpie 时，把本包目录放进 magpie 的插件卷（如 `magpie-config/_data/magpie/plugins/...`），在插件页填容器内对应路径，重启生效。

## 改动说明（相对官方 0.2.20）

官方 0.2.20 把包拆成了三个文件：`opencode.mjs`（OpenCode 1.x 入口，re-export）、`opencode2.mjs`（OpenCode 2.x 适配，原样照搬）、`index.mjs`（主体）。本 fork **只改 `index.mjs`**，共四处、未改原逻辑：

- `SITES` 里把 `workbuddy` / `workbuddy-ai` 两个站点的 **`id` 和 `name`** 改成 `-import` 版（见上表）。
- 新增 **`mapAccount()`**（把单个账号——`.info` / 扁平 token / `workbuddy-accounts` 条目——解析成存储用的 token 形状，含秒↔毫秒归一化 `msOf()` 与 markdown domain 清洗 `stripMd()`）、**`parseImport()`**（单个）、**`parseImportAll()`**（批量：数组或 `{accounts:[...]}`）和 **`importSignIn(site, client)`**（第四种登录方式，一次导入全部，其余经 `client.auth.set` 存入 `<id>#<uid 哈希>`），紧跟在 `success()` 之后。
- 在 `auth.methods` 数组里（桌面登录与 API key 之间）加了「导入登录 JSON」一项。
- `_internal` 导出里加了 `importSignIn`、`parseImport`、`parseImportAll`、`mapAccount`，方便测试。
- `auth.methods` 里「导入登录 JSON」一项的 `authorize` 改为 `() => importSignIn(site, client)`（传入 `client` 以便批量写入）。

`opencode.mjs` / `opencode2.mjs` / `package.json` 的 `main`、`exports` 结构与官方一致；`package.json` 只改了 `name`、`version`、`description`。

升级官方版时，把以上四处重新应用到新版本的 `index.mjs` 即可。

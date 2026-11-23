# sctl / scriptcat-userscript skills — upstream vs. localized

本仓库保存两套 ScriptCat skill 的两份版本，用来做「原版 → 本地化」的对照与回滚。

```
upstream/   官方 main 分支原样拷贝（未改动），附 SHA256SUMS.txt
local/      本地化版本：面向 Windows + PowerShell 7 + DSH，并补进实测结论
```

装到 DSH 里的是 `local/`：

```powershell
Copy-Item -Recurse -Force .\local\sctl                   "$env:DSH_HOME\skills\sctl"
Copy-Item -Recurse -Force .\local\scriptcat-userscript   "$env:DSH_HOME\skills\scriptcat-userscript"
```

`dsh-skill-filesystem` 扫描 `<root>/<name>/SKILL.md`，落盘即热加载，不需要重启。

## 改了什么

### sctl

| 文件 | 改动 |
|---|---|
| `SKILL.md` | 新增开头一节 **"This machine: Windows + PowerShell + DSH"**：`$sctl` 绝对路径（裸 `sctl` 在常驻 shell 里不在 PATH）、`ConvertFrom-Json` 取代 `jq`、引号规则（单引号包 JS、内部双引号、对象字面量加圆括号）、`--params` JSON、`$LASTEXITCODE`、`>` 重定向是 UTF-8 无 BOM。全篇 `jq .tabId` / `TAB=$(…)` 示例改为 PowerShell；`browsers -o json` 的实际形状修正为裸数组；补 `status -o json` 的形状 |
| `SKILL.md` | 新增结尾一节 **DSH specifics**：优先用 `mcp__scriptcat__*`（54 个工具，与 CLI 一一对应），何时仍用 CLI；审批策略在扩展卡片而非 sctl；两个通道之间可能出现的重连失败；本机布局表（二进制、任务计划、数据目录、地址、bundle） |
| `SKILL.md` | 错误表：`PAGE_UNRESPONSIVE` 补「关标签页重开而不是原地重试」；`PAGE_NOT_AUTOMATABLE` 补 `edge://` 与扩展页面（配对码、策略开关、插件开关不可自动化） |
| `SKILL.md` | 截图一节补两个坑：扩展名必须与 `--format` 一致；全页 PNG 会超 4 MiB，改 `--quality 60-70` + 视口 |
| `references/page.md` | 调试流程示例改 PowerShell + 引号提醒 |
| `references/cdp.md` | `cdp send` / `cdp endpoint` 改 PowerShell，补 `$env:SCTL_CDP_HTTP` 取用与清理，点明 JSON 是 `.endpoint.httpUrl` 嵌套 |
| `references/scriptcat.md` | 编辑流程改 PowerShell，补 `-f edits.json` 的用法与 `$` 插值提醒，补 `GM_log(message, level)` 顺序 |

官方原版的技术事实（权限模型表、破坏性命令的 `--yes` 规则、退出码表本身）**未改动**；措辞层面已本地化的部分：
`dev-loop.md` 的「每次写操作都等审批」标题改成「*可能*等审批」并补了 直接允许 的说明，`SKILL.md` 里
`PAGE_UNRESPONSIVE` / `PAGE_NOT_AUTOMATABLE` 两行重写，`references/page.md` 补了 `page wait --text` 大小写敏感。

### scriptcat-userscript

| 文件 | 改动 |
|---|---|
| `SKILL.md` | 新增 **第 0 节 "This machine: Windows + PowerShell + DSH"**：`$sctl`、`ConvertFrom-Json`、何时改用 MCP 工具、审批在扩展卡片、`$env:DSH_NODE_EXECUTABLE` 跑 `check.mjs` |
| `SKILL.md` | 日志规则补 **`GM_log(message, level)` 顺序**：写反会被静默丢弃，日志里只剩 "exec script complete"（1.5.2 实测） |
| `references/dev-loop.md` | 全文 PowerShell 化：`$sctl` 前缀、`ConvertFrom-Json).tabId`、`-f edits.json`、`file:///D:/…` 路径、`$env:DSH_NODE_EXECUTABLE`；「每次写操作都要审批」改为「先看策略，本机是直接允许」；新增末节 **本机实测坑**（6 条） |
| `references/background.md` | `GM_log` 顺序告警 |
| `README.md` | 补本机装法（`$DSH_HOME/skills` 拷贝）、本地化差异表、`PAGE_NOT_AUTOMATABLE` 对热重载的影响；提醒不要用 `npx skills update` 覆盖本地化 |

## 回滚与更新

```powershell
# 单文件回滚
git checkout upstream/local -- . ; # 或手工：
Copy-Item -Recurse -Force .\upstream\sctl "$env:DSH_HOME\skills\sctl"

# 全部回滚到上游
Remove-Item -Recurse -Force "$env:DSH_HOME\skills\sctl","$env:DSH_HOME\skills\scriptcat-userscript"
Copy-Item -Recurse -Force .\upstream\sctl,.\upstream\scriptcat-userscript "$env:DSH_HOME\skills\"
```

上游更新（`upstream/` 里是什么、什么时候拉的）：

- `sctl`：GitHub `scriptscat/sctl` `main`，`skills/sctl/`，2026-11-23 拉取，对应 sctl v0.4.1 发布周期。
- `scriptcat-userscript`：GitHub `scriptscat/scriptcat` `main`，`skills/scriptcat-userscript/`，同日拉取。

更新流程：把新的上游文件放进 `upstream/`，`git diff upstream local` 看差异，手工合并到 `local/`，
再拷进 `$DSH_HOME/skills`。`upstream/SHA256SUMS.txt` 是拉取时的基线，用来确认 `upstream/` 没被误改。

## 已实测验证过的改动

- `$sctl` 绝对路径下 CLI 子命令可用；`ConvertFrom-Json).tabId`、`.tabs | Where-Object`、`.endpoint.httpUrl` 均验证。
- `page eval` 圆括号规则：**注意真实失败模式**——单键 `{a:1}` 不报错，而是被当成标签语句、静默只返回值（exit 0）；
  多键才报 `SyntaxError: Unexpected token ':'`。
- `cdp send --params '{"…"}'` 单引号包 JSON：正常返回 Chrome 结果；JSON 数组会被拒。
- `-f edits.json`、`-f -`、`@path`、`@@` 的语义全部验证；用不匹配的 oldText 测试，两个脚本未被改动。
- 截图：`--format` **默认 png 且与文件名无关**，两个方向都会错配（`.png`+jpeg / `.jpeg`+无 format）。
- `PAGE_UNRESPONSIVE` 用「关标签页重开」恢复：实际恢复成功。
- `PAGE_NOT_AUTOMATABLE`：扩展页与 `edge://` 不可自动化，实测。
- **并发是硬约束**：4 个进程同时压 daemon 时扩展 4 分钟内重连 6 次，`get` 的退出码在 2/3 之间跳变。

## 已知未验证 / 已知缺陷

- `page upload` 在本机**完全不可用**：任何调用都返回 `INVALID_REQUEST: {"code":-32000,"message":"Not allowed"}`，
  即使元素可见、文件可读。CDP `DOM.setFileInputFiles` 的替代路径**未验证**。
- Playwright/Puppeteer 经 `cdp endpoint` 真实连接**未验证**（只验证了签发与关闭）。
- `@require file:///` 的百分号编码路径（含空格/中文）能否被 ScriptCat 正确解码**未验证**。
- 定时脚本（`@crontab`）真实触发**未验证**，只验证了 `run` 手动执行。
- `sctl Browser` 0.4.0 与 `sctl` 0.4.1 版本不一致，目前正常。

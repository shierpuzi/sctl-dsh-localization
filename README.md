# sctl / scriptcat-userscript skills — upstream vs. localized

本仓库保存两套 ScriptCat skill 的两份版本，用来做「原版 → 本地化」的对照与回滚。

```
upstream/   官方 main 分支原样拷贝（未改动），附 SHA256SUMS.txt
local/      本地化版本：面向 Windows + PowerShell 7 + DSH，并补进实测结论
```

## 结论先说

- **没有做本地化。** `local/` 目前与 `upstream/` 逐字节相同——我当初是直接从 GitHub 拉下来落盘的，
  一个字符都没改。你说得对：仓库的 skill 不针对 DSH 做优化。
- 逐文件校验（`SHA256SUMS.txt`）确认 `sctl/SKILL.md` 与 4 个 references 全部 identical。

## 原版对 DSH + 本机的四处硬伤

| # | 原版写法 | 本机实际 | 后果 |
|---|---|---|---|
| 1 | 通篇 `sctl <cmd>` | DSH 的 shell 是常驻进程，**看不到**后来写进注册表的新 PATH；`sctl` 不在 PATH 里 | 裸 `sctl` 直接「不是内部或外部命令」 |
| 2 | `TAB=$(sctl tabs open … -o json \| jq .tabId)` | **没有 jq**；PowerShell 里 `$( )` 是子表达式不是命令替换 | 取 tabId 的写法整段作废 |
| 3 | 假设 bash/zsh 与 Unix 路径 | 是 PowerShell 7.6.6；`>` 重定向默认 UTF-8 无 BOM（已实测，正常） | 引号、管道、`export` 一类写法全要换 |
| 4 | 面向 Claude Code / Codex 的挂载方式（`npx skills add -g`、`ln -s ~/.claude/skills`） | DSH 的技能根是 `$DSH_HOME/skills`，工具是 `plugin_manager` | 安装/更新路径完全不同，还牵扯审批与 HMR |

另外原版把 `GM_log` 的参数顺序讲对了（`GM_log(message, level)`），这点原版没错——是我第一次照 Tampermonkey
习惯写反了。这条会作为「实测结论」写进 local 版，避免下次再踩。

## local/ 要做的事（待办）

1. **命令别名**：所有 `sctl` 统一改成 `$sctl`（会话开始时 `$sctl = "$env:LOCALAPPDATA\sctl\bin\sctl.exe"`），
   或在正文开头给一次 `Set-Alias`；避免依赖 PATH。
2. **jq 替换**：`| jq .tabId` → `| ConvertFrom-Json).tabId`；`jq '.tabs[] | select(...)'` → `... | ConvertFrom-Json).tabs | Where-Object { $_.url -match '…' }`。
3. **引号规则**：`sctl page eval '({a: 1})'` 用单引号包住、内部用双引号；对象字面量必须带外层圆括号。
4. **DSH 专章**：MCP 走 `sctl.exe mcp --name dsh`（bundle 见 `D:\text\sctl-dsh-bundle`），
   工具名 `mcp__scriptcat__*`；写操作/读源码的审批策略在扩展卡片里，不在 sctl 里。
5. **实测补丁**：`GM_log(msg, level)` 参数顺序、`tabs open` 返回时页面尚未加载完、截图扩展名必须与 `--format` 一致、
   文档站重载后可能 `PAGE_UNRESPONSIVE`（关掉重开而不是重试）、`raw.githubusercontent.com` 在本机超时。
6. **安装/更新**：`$DSH_HOME/skills/<name>/` 落盘即被发现（无需重启）；更新用 `cp` 覆盖 local/ 即可。

改完用 `git diff --stat upstream local` 对照，必要时 `cp upstream/... local/...` 回滚。

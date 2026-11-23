# sctl / scriptcat-userscript skills — 本地化与实测缺陷集

把 ScriptCat 官方的两个 agent skill 搬到 **Windows + PowerShell 7 + DeepSeek Harness** 上，并把实跑踩到的坑写回去。

```
upstream/   官方 main 分支原样拷贝（未改动），附 SHA256SUMS.txt
local/      本地化版本：面向 Windows + PowerShell + DSH，并补进实测结论
```

## 为什么需要这个仓库

官方 skill（[scriptscat/sctl](https://github.com/scriptscat/sctl) 的 `skills/sctl`、
[scriptscat/scriptcat](https://github.com/scriptscat/scriptcat) 的 `skills/scriptcat-userscript`）是写给
bash + `jq` + Claude Code/Codex 的。照抄到 PowerShell 上会有一批**静默失败**：命令跑通、退出码 0、结果是错的。
本仓库保留上游原文以便 diff 与升级，并提供可直接部署的本地化版。

## 装到 DeepSeek Harness

```powershell
Copy-Item -Recurse -Force .\local\sctl                 "$env:DSH_HOME\skills\sctl"
Copy-Item -Recurse -Force .\local\scriptcat-userscript "$env:DSH_HOME\skills\scriptcat-userscript"
```

`dsh-skill-filesystem` 扫描 `<root>/<name>/SKILL.md`，`SKILL.md` 落盘即热加载，不需要重启
（`references/` 子树的改动不触发重载，但下次加载会读到新内容）。

## 最有价值的几条（都是实测，不是推测）

| 缺陷 | 真实后果 |
|---|---|
| `sctl get <uuid> -o source > file.user.js` | PowerShell 在原生命令**运行前**截断目标文件；读取失败（exit 1/2/3）后只剩 **0 字节**（实测 48→0）。必须先捕获变量、判 `$LASTEXITCODE` 再写盘 |
| `grep` 无匹配返回 **exit 0** | 「旧文本没了 = 编辑生效」是错的推论。编辑失败后复检 grep 反而像成功。必须 grep **新**文本且要求非空 |
| `--replace-all` 作用于**本次调用的所有** edit | 为一条不唯一锚点加这个 flag，会把其它精心锚定的改动一起放开 |
| `page eval '{a: 1}'`（单键，无括号） | **不报错**，被当成标签语句，静默只返回值（exit 0，形状错）；多键才报 `SyntaxError` |
| `--format` 默认 png 且**与文件名无关** | `-f x.png --format jpeg` 和 `-f x.jpeg`（不带 format）两个方向都会写出扩展名与内容不符的文件 |
| `cdp status` / `cdp endpoint` 不带 `-o json` | 会把**带密钥的地址**打到终端。回答「有没有客户端连着」只能用 `-o json` 读 `clientConnected` |
| 并发调用 | 扩展单线程、每 tab 一个调试器会话。4 个进程同时压 → 4 分钟重连 6 次，退出码在 2/3 之间跳变 |
| `$sctl` 跨 tool call 不存在 | 每次 shell 调用都是新进程；且 `sctl` 不在 DSH 子进程继承的 PATH 里（注册表里有，继承块里没有） |
| `browsers --browser x` | `unknown flag`；`--limit` 在 `recent list` 是 25，在 tabs/windows/groups/extensions list 根本不存在 |
| `page upload` | 在本机恒定返回 `INVALID_REQUEST: {"code":-32000,"message":"Not allowed"}` |
| `page detach --all` | 会伸进**用户自己的**标签页并取消他们的对话框；应按 id detach |

完整清单（18 项）与未验证面见 `local/`、`upstream/` 之外的本文件下方「审计记录」。

## 审计记录

四个独立审计（CLI 事实核对、PowerShell 逐条实跑、负向 bug 猎手、上游对照）共产出 18 个已修缺陷与一批
显式「未验证」项。未验证的包括：真实审批拒绝路径（exit 1）、`run -o json` 的字段、Playwright 真连 CDP、
`@require file:///` 百分号编码的解码、`@crontab` 真实触发、`page upload` 的 CDP 替代方案。

一处值得记的教训：审计者曾报告「`raw.githubusercontent.com` 稳定挂、dev 文档正常」，复测三次结论**相反**，
最终措辞改成「两个方向都抖，重试一次再下结论」。子智能体的结论同样需要验证。

## 回滚与更新

```powershell
# 全部回滚到上游
Copy-Item -Recurse -Force .\upstream\sctl,.\upstream\scriptcat-userscript "$env:DSH_HOME\skills\"

# 上游更新：把新文件放进 upstream/，对照 diff 合并到 local/，再拷进 $DSH_HOME/skills
git diff --no-index upstream/sctl local/sctl
```

上游基线（`upstream/`）拉取于 2026-11-23：
- `scriptscat/sctl` `main` 的 `skills/sctl/`，对应 sctl v0.4.1
- `scriptscat/scriptcat` `main` 的 `skills/scriptcat-userscript/`

`upstream/SHA256SUMS.txt` 是拉取时的哈希基线，用来确认 `upstream/` 未被误改。
注意 `git diff upstream local` 不工作（二者是目录不是 ref），用 `git diff --no-index`。

## 许可

`upstream/` 与 `local/` 均派生自 GPL-3.0 项目：

- [scriptscat/sctl](https://github.com/scriptscat/sctl) — GPL-3.0
- [scriptscat/scriptcat](https://github.com/scriptscat/scriptcat) — GPL-3.0

因此本仓库整体以 **GPL-3.0** 分发，全文见 [`LICENSE`](LICENSE)。相对上游的修改已在本 README 与
`git diff --no-index upstream local` 中标注。上游作者与原版权归各自项目所有。

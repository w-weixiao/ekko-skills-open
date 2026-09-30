---
name: github-open-source
description: "把本地技能/仓库开源到 GitHub：认证、干净副本、敏感清洗、纯中文 README、可发现性。不用于纯代码项目开源。"
version: 1.0.0
license: MIT
author: Ekko Agent
metadata:
  keywords:
    - "github"
    - "open-source"
    - "publish"
    - "repo"
    - "readme"
---

# 本地技能开源到 GitHub

## 流程

1. **干净副本**：在 workspace 建公开版目录，不直接改本机活技能目录。
   - 删：内部路径、内部技能互引、`author: <本机 agent 名>`、本机 IP/用户名。
   - 敏感清洗（**首个 commit 前必跑**）：对全部文件 grep 成人内容关键词、个人项目名/作品名、本机绝对路径。内部运行日志/复盘进公开仓库必须**蒸馏成概述**，不 dump 原始日志；sed 占位符改名不算清洗——整段成人向写作 SOP 仍会泄漏，整段删。
   - 例外"原样模式"（用户明说"保持和本机原样/不用删信息"）：照抄本机活技能源文件（SKILL.md+references/；.archive/ 归档默认不传，仓库已有目录如 test/ 一律不动），不洗不重置版本/作者；敏感清洗降级为"扫并报告"——按 references/privacy-scan.md 的 grep 表扫一遍，命中列成表给用户判断，不自动改；用户批了才改，且改本机源文件后再覆盖仓库副本，不在副本里改。

2. **认证**：`gh auth login --web`（后台跑，从日志取设备码+URL 给用户，浏览器输码，密码不流经对话）。用户报账号密码也拒收，走设备码。本机 gh 为官方二进制（通常装在 `~/.local/bin/gh`，可能不在默认 PATH，需确认安装位置）。

3. **建仓+推送**：先 `gh repo list` 确认目标仓库不存在再 `gh repo create <owner>/<repo> --public`（rate limit 时可能静默失败——报成功≠真建上，必须复查 `gh repo list`）。在 workspace 干净副本目录里 `git init` + 本地 commit（不用 `--source .` 直接挂活目录）。
   - 推送走 token：`git remote set-url origin "https://x-access-token:$(gh auth token)@github.com/<owner>/<repo>.git"` → `git push -u origin master` → 重置回干净 URL。
   - 验证：`gh api repos/<owner>/<repo>/contents` 确认文件真推上去了。

4. **README（本用户固定约定）**：纯中文+emoji 节标记（用户不读英文，用户面向内容不翻英文；第三方 references/ 保留原文语言不翻）。固定分节顺序：
   🎯 一句话概括能干什么 → 🔥 碰到的痛点 → 🌟 用了以后的效果（一般结论）→ 📊 我们实测的效果（**用自己项目真实运行数据做判断标准的表格**，不写营销话术）→ 🗂️ 仓库结构 → 🚀 安装 → 🔗 配套 → 📄 许可。

5. **可发现性**：`gh repo edit <owner>/<repo> --description "带关键词的中文定位句" --add-topic llm-agent,claude-code,codex,...`（描述+topics 都是仓库元数据，改完无需 commit）。

6. **全历史验证**：`git grep -lE "<敏感词表>" $(git rev-list --all)` 命中须为 0，再 clone 干净副本复验远端。

## 坑

- 内容进了 git 历史就永久在（任何人可 log 翻出）；发现泄漏后"新增删除 commit"不够，必须重写历史。小仓库无 fork 时最稳=**重建单 commit 干净历史 + `git push -f`**（已验证）；`git filter-repo --path <单文件>` 会把树里其余文件搞丢，小仓库别用。
- `terminal_exec` 里含 `$(...)`、管道、`&&` 的长 shell 命令易被参数拆分吞掉（`git rev-list --all` 在 `git grep` 里触发 `fatal: ambiguous argument '$(git'` 等）。安全做法：拆成多行/单独命令、先落盘临时文件再 grep，或改用 python 单行脚本执行敏感扫描；命令里避免裸 `2>/dev/null` 与嵌套 `$()` 同时出现。
- 用户说"放进项目文件夹"=把仓库目录 clone/拷进去，别自作主张打 tar/zip 包（用户取消过自动生成的包）；要"发给我下载"才打包。
- 强推前确认无 fork（fork 后强推重置别人）；本用户仓库 fork=0 时才安全。
- 推送时网络断（GitHub TLS 握手掐断、curl 返回 000）：先本地 commit（本机 git 无全局身份，需先 per-repo `git config user.name/user.email`），**立刻把 remote 复原成无 token 的干净 URL**——注入过 token 的 URL 不能跨回合留在 remote 里，否则 token 明文落盘；再挂后台每 ~2 分钟重试 push（带 notify，完成自动回报）。网络断时 `gh auth status` 会误报 token invalid，那是连不上不是真失效，恢复后直接推，不重走设备码登录。
- 清单型人设泄漏（某行点名一串 18+ 技能的具体名字）修法 = 把名单换成通用判据描述（如"登记为 18+ 内容技能/用户维护的清单"），保留原判断逻辑不删整行——删整行会丢掉该条判据的分档作用。

## references

- `references/privacy-scan.md`：敏感清洗 grep 词表与判定规则。

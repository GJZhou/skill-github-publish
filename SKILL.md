---
name: skill-github-publish
description: Publish a local Codex skill to GitHub. Use when the user asks to upload or push one skill directory to its own GitHub repository, with private visibility by default and explicit confirmation required before replacing an existing remote.
---

# Skill GitHub Publish

将一个本地 Codex skill 安全地发布到 GitHub。默认以本地 skill 目录名创建同名私有仓库；用户明确要求公开时才创建公开仓库。一次调用默认处理一个 skill，用户指定多个时逐个处理并分别报告。

## 适用范围

当用户要求“把某个本地 skill 上传/推送到 GitHub”“为 skill 新建仓库并发布”或“同步本地 skill 到远端仓库”时使用本技能。

普通业务代码仓库、安装第三方 skill、只查看远端仓库或只生成 Git 命令不触发本技能。

## 输入与默认值

从用户输入中取得以下信息；没有提供时使用默认值：

- **本地 skill**：支持绝对路径、skill 目录名，或 `$skill-name`。目录必须包含 `SKILL.md`；只在已知的全局 skill 根目录中查找，不递归扫描用户整个磁盘。
- **GitHub 地址**：用户可提供完整仓库 URL（`https://github.com/owner/repo` 或 SSH 地址）、owner/repo，或只提供 owner。缺少 owner 时，从已认证的 GitHub 账号读取。
- **仓库名**：默认使用本地 skill 目录的基础名称；若用户明确指定仓库名，使用用户指定值。
- **可见性**：默认 `private`；只有用户明确说公开/public 时才使用 `public`。
- **账密**：优先使用已经登录的 `gh` 凭据；没有登录且用户允许时，从全局 Agent 配置读取 GitHub 账密，或使用用户在当前会话提供的凭据。具体安全流程见 [references/authentication-and-safety.md](references/authentication-and-safety.md)。

## 执行流程

### 1. 解析与预检

1. 唯一定位本地 skill，读取其 `SKILL.md`、`agents/openai.yaml` 和实际资源清单。出现多个同名目录时先列出候选并请求用户选择。
2. 只复制 skill 自身内容；不复制父目录、全局 `AGENTS.md`、浏览器会话、认证文件、`.git`、缓存、构建产物或临时目录。
3. 执行可用的 skill 校验（优先使用 `.system/skill-creator/scripts/quick_validate.py`）。校验失败先报告，不以“上传成功”掩盖源文件问题。
4. 检查待上传文件是否含密码、令牌、私钥、Cookie、认证状态或其他明显敏感值。发现疑似敏感内容时暂停并列出文件路径和原因，不上传、不在消息中复述敏感值。
5. 检查源 skill 是否有适用的 `AGENTS.md` 或其他规则文件；遵循其中的文件、提交和验证规则，但不要把这些规则文件自动复制进远端仓库，除非它们本来就在 skill 目录内。

### 2. 认证与远端定位

1. 优先使用 `gh auth status`、`gh api user` 或已配置的 Git credential helper。命令输出不得包含 token 或密码。
2. 未认证时，优先使用 `gh auth login --web` 配合 agent-browser 完成设备登录；不能把普通账号密码拼入 URL、remote URL、命令参数或提交内容。
3. 用户允许读取全局 Agent 配置时，只读取已知的 Agent 文件和明确的 GitHub 配置段；凭据只在内存或安全凭据存储中使用，不打印、不写入 skill 仓库。
4. 解析最终的 owner、仓库名和可见性，并在创建或推送前向用户报告目标。用户没有提供远端地址时，默认使用已认证账号和本地 skill 名称。

### 3. 判断远端是否存在

- 远端不存在：按可见性创建同名仓库。默认私有；公开仓库必须有用户的明确表述。
- 远端已存在：先读取远端默认分支、是否为空、当前提交和文件概况，**在用户明确确认前不执行覆盖、强制推送或删除远端文件**。
- 远端存在但内容与本地完全一致：不需要覆盖，验证后报告“已同步，无需写入”；不要为了形式再次推送。
- 远端存在且包含其他内容：确认问题必须说明精确仓库、默认分支以及“远端默认分支中不属于本地 skill 的文件是否全部替换”。用户只说“可以”但未指明覆盖范围时继续询问。

### 4. 打包、提交与推送

1. 在临时目录建立 staging 副本，保持源 skill 原样不改。复制相对目录结构和文本/脚本/资源文件。
2. 新仓库使用 `ADD SKILL-UPLOAD Initial skill upload` 形式的本地提交；已有仓库按适用项目规则使用 `MOD` 提交。若用户或项目规则指定需求编号，替换 `SKILL-UPLOAD`。
3. 新仓库使用普通 `git push`。已确认覆盖的远端先获取并核对默认分支和远端最新提交，再使用 `--force-with-lease`，不得使用无保护的 `--force`。
4. 覆盖范围以“远端默认分支的仓库内容替换为当前 skill 内容”为准；如用户只希望更新远端某个子目录，改为该子目录同步，不删除其他内容。
5. 只提交当前 skill 的文件；不暂存当前任务之外的文件，不执行 `git push` 到其他远端，不修改源 skill 的 Git 历史。
6. GitHub 返回 secondary rate limit、认证失败、权限不足或远端发生变化时停止重试，报告已完成步骤和待用户处理事项。不要高频循环请求。

### 5. 交付验证

推送后至少完成以下只读检查：

- `git ls-remote` 或 GitHub API 能看到目标分支和新提交；
- 远端仓库可见性与用户选择一致；
- 远端文件清单与 staging 副本一致，未包含认证文件或临时文件；
- 报告本地源路径、远端 URL、仓库可见性、提交 ID、文件数量、是否覆盖及未上传原因。

不要在最终消息中输出密码、token、Cookie、完整认证 URL 或凭据文件内容。认证、远端写入和覆盖属于外部状态变更，只有在用户明确请求本技能并满足上述确认条件时执行。

详细认证处理和远端覆盖分支分别见：

- [references/authentication-and-safety.md](references/authentication-and-safety.md)：凭据来源、设备登录、敏感信息和限流处理。
- [references/publish-workflow.md](references/publish-workflow.md)：源目录解析、staging、创建仓库、已有仓库确认和验证清单。

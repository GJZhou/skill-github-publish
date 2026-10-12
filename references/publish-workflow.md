# Skill 发布工作流

## 源目录解析

1. 用户给绝对路径时直接验证该目录；给 skill 名称时只在已知全局 skill 根目录查找，例如 `C:\\Users\\<user>\\.codex\\skills`、`C:\\Users\\<user>\\.agents\\skills` 和当前项目明确登记的 skill 目录。
2. 目录必须有 `SKILL.md`。读取 frontmatter 的 `name`，确认与目录名或用户指定名称没有冲突。
3. 用受限清单收集文件。默认排除 `.git`、`__pycache__`、`.pytest_cache`、`node_modules`、`.venv`、构建输出、浏览器会话、临时压缩包和凭据文件；若用户明确要求资源目录，再逐项加入。

## 远端命名和创建

- 完整 URL 或 `owner/repo` 以用户输入为准；没有 owner 时从 `gh api user --jq .login` 获取。
- 没有仓库名时使用源目录基础名，并把不符合 GitHub 命名规则的字符转换为单个连字符后再次向用户报告。
- 默认执行 `gh repo create owner/repo --private`；只有用户明确要求公开才使用 `--public`。
- 创建请求逐个串行执行。成功后记录返回的 HTML URL 和默认分支；失败不要继续创建一批相似仓库。

## staging 和提交

在临时目录完成打包，不把 `.git` 从源目录复制过去：

```text
staging/<repo-name>/
├── SKILL.md
├── agents/
├── references/       # 如果源 skill 有
├── scripts/          # 如果源 skill 有
└── assets/           # 如果源 skill 有
```

新仓库可在 staging 中执行：

```text
git init -b main
git add -A
git commit -m "ADD SKILL-UPLOAD Initial skill upload"
git remote add origin <authenticated-remote>
git push -u origin main
```

提交消息必须遵循源项目或全局 Agent 规则；上面只是没有专用规则时的默认示例。不要把 token 放进 `<authenticated-remote>`，让 `gh auth setup-git`、SSH 或 credential helper 提供认证。

## 已有仓库分支

远端存在时，先执行只读检查：

1. 获取默认分支、最新提交和文件树。
2. 比较远端与本地 skill 的相对文件清单和内容。
3. 若无差异，报告已同步并结束。
4. 若有差异，向用户说明：仓库 URL、默认分支、远端现有文件范围、本地将上传的文件范围，以及是否删除远端多余文件。
5. 只有用户明确确认“覆盖该仓库默认分支”后，才同步 staging、提交 `MOD SKILL-UPLOAD Update skill content` 并推送。先 `git fetch` 获取最新远端提交，再使用 `git push --force-with-lease`；lease 不匹配就停止并重新确认。

如果用户只确认“更新 skill 子目录”，保留远端其他文件，只替换指定子目录，不能扩大为整个仓库覆盖。

## 完成检查

- 远端 URL、可见性和默认分支记录清楚。
- `git ls-remote --heads origin <branch>` 返回刚刚提交的 commit ID。
- 远端树中只有预期的 skill 文件（或用户明确保留的其他文件）。
- 最终报告列出源路径、仓库地址、提交 ID、文件数量、创建/覆盖状态和任何跳过文件；认证信息只写“已认证”，不写具体值。

# Codex PR Agent Suite

这是对当前 `gpf.zip` / `gpr.zip` 的 Codex 版改造，原压缩包保持不变。

目录包含三个可安装的 skill：

- `codex-github-pr-reviewer`：只读审查单个 PR，输出结构化问题清单。
- `codex-github-pr-fixer`：根据已验证问题在当前 checkout 修复并运行测试。
- `pr-loop`：协调 reviewer → fixer → reviewer 循环，直到收敛或明确阻塞。

## 安装

将三个目录复制到 Codex skill 目录（通常是 `~/.codex/skills`；如果设置了
`CODEX_HOME`，则使用 `${CODEX_HOME}/skills`）：

```bash
SKILL_ROOT="${CODEX_HOME:-$HOME/.codex}/skills"
cp -R codex-github-pr-reviewer codex-github-pr-fixer pr-loop "$SKILL_ROOT/"
```

安装后重启 Codex 或新建会话，让 skill 清单重新加载。

## 使用

独立使用：

```text
Use $codex-github-pr-reviewer to review owner/repo #123.
Use $codex-github-pr-fixer to repair the verified findings in owner/repo #123.
```

协调运行：

```text
Use $pr-loop to review and repair owner/repo #123 until it converges.
```

默认只修改本地工作区，不提交、不推送、不发 GitHub 评论。只有用户明确要求
发布时，才把 `publish=true` 作为协调器的权限条件；最终提交和推送仍由父级
协调器做一次性安全检查。

Codex 的持久复用单位是 skill；agent ID 和上下文是运行时对象。因此协调器
skill 会在当前任务中使用 Codex 的多 agent 原语创建/复用两个临时角色，并通过
JSON handoff 交接 `head_sha`、发现、变更和测试证据。


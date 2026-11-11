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

`pr-loop` 默认通过 GitHub PR threads 让 reviewer 和 fixer 两个独立 agent
交流：reviewer 发布 findings，fixer 在对应 thread 回复、修复、提交并推送，
协调器验证 head 后发布最终状态。需要纯本地演练时显式指定 `publish=false`；
此时不提交、不推送、不发评论。

Codex 的持久复用单位是 skill；agent ID 和上下文是运行时对象。因此协调器
skill 会在当前任务中使用 Codex 的多 agent 原语创建/复用两个临时角色，并以
PR threads 作为 durable handoff，同时用 JSON 交接 `head_sha`、发现、变更和
测试证据。


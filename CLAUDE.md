# CLAUDE.md

本项目的说明与约束拆成两份，两份都要读：

- @PROJECT_RULES.md — **踩过的坑和不能破坏的边界**，动手前先看
- @AGENTS.md — 项目结构、构建测试命令、目录职责地图

## 本项目额外注意

- 面板前端 `internal/app/admin_ui/index.html` 是 `//go:embed` 进二进制的，
  改完必须重新 `go build`，重启旧二进制看不到任何改动。
- 判断「cookie 还能不能用」只认 `/app` 页面里的 `SNlM0e`，不能拿请求返回 200 当判据 ——
  失效的 cookie 不会被上游拒绝，只会被静默当成匿名用户。
- 涉及模型能力边界的改动，先查 `PROJECT_RULES.md` 第 3 条那张表。

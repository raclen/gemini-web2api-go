# AGENTS

给 AI 编码助手看的项目说明。**动手前先读 [`PROJECT_RULES.md`](./PROJECT_RULES.md)** ——
那里是踩过的坑和不能破坏的边界，本文件只讲项目长什么样。

---

## 这是什么

把 Gemini 网页版（`gemini.google.com`）逆向成 OpenAI 兼容 API 的单体 Go 服务。
自带 SQLite、管理面板、cookie 账号池、代理池和限流器，一个二进制跑起来就完事。

- 对外接口：`/v1/chat/completions`、`/v1/responses`、`/v1/models`、`/mcp`
- 管理面板：`/admin`（同端口，单独的 session 鉴权）
- 存储：SQLite（`modernc.org/sqlite`，纯 Go，无 CGO）

---

## 构建与测试

```bash
go build ./...          # 编译
go test ./...           # 全部测试（internal/app 下 11 个 _test.go）
go run . --port 8083    # 本地跑，改了前端也会自动重新 embed
```

发布产物：`go build -trimpath -ldflags="-s -w" -o gemini-web2api .`（见
`.github/workflows/release.yml`）。Docker 走 distroless + `CGO_ENABLED=0` 交叉编译。

**Windows 注意**：`go` 未必在 PATH 里，编译前先确认 `go version` 能跑。

---

## 目录地图

```
main.go                     只有一行：app.Run()
internal/app/
  app.go                    启动流程、路由注册、优雅关闭
  config.go                 启动期配置（CLI flag > config.json > 内置默认）
  runtime.go                运行时配置（面板可改，存 kv 表，盖在启动配置之上）
  db.go                     SQLite schema、启动期迁移、请求记录落库

  ── 请求主链路 ──
  server.go                 OpenAI 兼容 handler；Version 常量也在这里
  messages.go               OpenAI messages → Gemini prompt，含长度预算
  gemini.go                 核心：挑号 → 挑出口 → XSRF → StreamGenerate → 解析响应帧
  sse.go                    流式输出
  xsrf.go                   SNlM0e token 获取与缓存
  bl.go                     上游前端版本号 bl，支持定期自动跟随

  ── 资源池 ──
  cookie_pool.go            Google 账号池：挑号、健康度、连续失败自动停用
  rotate.go                 cookie 保活，合并上游刷新的 *SIDCC
  proxy.go                  代理池：熔断、冷却、账号↔出口粘性绑定
  ratelimit.go              按 proxy_id 分桶的并发 / RPM / RPH slot

  ── 能力 ──
  vision.go                 图片输入
  upload.go                 文件上传到上游
  context_file.go           超长 prompt 转文本附件（绕开请求体大小墙）
  media.go                  生图（Nano Banana）/ 音乐（Lyria）及产物取回
  search.go                 搜索结果解析
  tokenizer.go              token 计数
  mcp.go                    MCP over HTTP，对外暴露 web_search

  ── 管理面板 ──
  admin.go                  统计、请求记录、代理 CRUD、运行时配置、连通性测试
  admin_cookies.go          cookie 池 CRUD、检测、保活、出口绑定
  apikey.go                 /v1/* 的 API key
  admin_ui.go               //go:embed 前端
  admin_ui/index.html       单文件面板（HTML + CSS + JS 全在里面）
  scheduler.go              后台任务：保活、统计聚合、过期数据清理
```

## 数据表

`proxies` / `accounts` / `requests` / `stats_hourly` / `stats_daily` / `sessions` / `kv`

`requests` 存明细（按 `retention_days` 过期删除），`stats_*` 是聚合结果（永久保留）。
`kv` 存运行时配置、API key、各种迁移标记。

建表和迁移都在 `db.go`，加列用 `ALTER TABLE ... ADD COLUMN`（SQLite 重复执行会报错但
被忽略，见现有写法）。

---

## 配置的三层优先级

```
面板改过的（kv 表 runtime_config）  >  config.json / CLI flag / 环境变量  >  内置默认
```

- 部署期配置（监听地址、DB 路径、admin token、API key、cookie 文件）**只在启动期读**，
  改了要重启，不进面板表单。
- 调优参数（超时、重试、限流、降级开关）放 `RuntimeConfig`，面板改完立刻生效。
- 给 `RuntimeConfig` 加字段时要同步四处：`config.go` 的 `Config` + `defaultConfig()`、
  `runtime.go` 的结构体 + `initRuntimeConfig()`、`admin_ui/index.html` 的 `RT_GROUPS`。
  后端 PUT handler 直接 Decode 整个结构体，不用改。

---

## 发版要动的地方

`server.go` 的 `Version` 常量、`CHANGELOG.md`。README 是中英双份
（`README.md` / `README_EN.md`），涉及用户可见行为的改动两份都要更新。

---

## 代码注释里的悬空引用

几处注释指向的文件**在本仓库里不存在**，别浪费时间找：

- `rotate.go` 引用的 `../../CLAUDE.md`（指的是作者旧的本地笔记，不是本仓库现在这份）
- `rotate.go` 引用的 `docs/local/capture-reading.md`

`docs/local/` 在 `.gitignore` 里被标注为「本地工程笔记，不入库不分发」。抓包结论
本身已经写进了对应的代码注释，读注释就够。

---

## 写代码时

- 注释用中文，代码标识符用英文。
- 现有注释密度较高且**解释的是「为什么」而非「做什么」**（多数带实测数据），
  修改附近代码时不要顺手删掉它们。
- 临时文件放项目根目录 `/temp`，从 GitHub 拉的临时仓库放 `/gittemp`。
- 优先最小修改，不做无意义重构，不随意升级依赖。

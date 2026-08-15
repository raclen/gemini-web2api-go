# PROJECT_RULES

本文件只记录**长期有效、可复用、跨任务**的约束与经验。一次性需求、临时方案、
能从代码或 `git log` 直接读出来的事实都不写在这里。

新增条目的门槛：同类问题重复出现、影响后续多人开发、用户明确要求，或涉及
安全 / 架构边界。

---

## 1. 改了 `internal/app/admin_ui/**` 必须重新编译

**触发信号**
改完面板 HTML/JS，重启进程，浏览器强刷之后新按钮 / 新列 / 新配置项依然不出现。

**根因 / 约束**
`internal/app/admin_ui.go` 用 `//go:embed admin_ui/*` 把整个目录打进二进制，运行时
从 `embed.FS` 读，不碰磁盘。旧二进制里装的还是旧 HTML，重启多少次都一样。

**正确做法**
前端改动跟后端改动同等对待：改完跑 `go build`，用新产物启动。开发期反复调 UI 时，
用 `go run .` 每次自动重新 embed。

**验证方式**
`curl -s localhost:8083/admin/ | grep <你新加的字符串>` —— 出不来就是没重新编译。

**适用范围**
任何涉及 `admin_ui/index.html`、`chart.umd.min.js` 的改动。

---

## 2. 不能拿「请求成功」当 cookie 有效性判据

**触发信号**
想确认某个账号还能不能用，于是发一个请求看返回码；或者在代码里用
「HTTP 200」推断登录态正常。

**根因 / 约束**
cookie 失效后上游**不会拒绝**，只是把你当匿名用户，纯文本请求照样返回 200。
后果是静默的：`gemini-3.1-pro` 被降级成 `3.5 Flash-Lite`、思考链变成 0 字符、
生图请求回一句 "Are you signed in?" 的纯文本。客户端完全看不出区别。

**正确做法**
唯一可靠的判据是 `/app` 页面里有没有 `SNlM0e`（即 XSRF token），见
`xsrf.go` 和 `cookie_pool.go` 的 `checkAccountCookie`。它只抓页面、不发对话、
不消耗生成配额。新增任何「检测账号是否可用」的路径都复用它，不要另起炉灶。

同理，`markCookieByStatus` 只把 401/403 算成 cookie 的错 —— 网络错误、代理失败、
302→sorry（IP 被拦）都是出口的问题，算进 `fail_count` 会让好号被误判成坏号。

**验证方式**
拿一个已失效的 cookie 发纯文本请求，观察返回 200 但 `upstream_model` 字段
掉档；再用面板「检测」按钮，应当明确报「cookie 已失效」。

**适用范围**
cookie 健康度、账号轮转、降级策略、面板检测相关的所有改动。

---

## 3. 登录态才生效的能力清单

**触发信号**
新增模型或新功能时，需要判断「这个能力匿名跑得动吗」。

**根因 / 约束**
下列能力**必须**有效登录态，匿名不是报错而是静默退化：

| 能力 | 匿名时的实际表现 |
|---|---|
| `gemini-3.1-pro` | 静默降级成 `3.5 Flash-Lite` |
| 扩展思考（`inner[80]=2`，所有 `-thinking` 变体） | 参数被服务端忽略，思考链 0 字符 |
| 生图 / 音乐（`inner[49]`，`gemini-image` / `gemini-music`） | 回一句 "Are you signed in?" 纯文本 |
| 图片输入 | 文件能传上去，但对话里一引用就被回 1100 |
| 超长对话转文本附件 | 同上，所以 `prepareContextFile` 匿名时直接返回 `used=false` |

反过来，`gemini-3.7-flash` / `gemini-3.6-flash` / `gemini-3.5-flash-lite` 三个
匿名跑和登录跑拿到的是同一个模型。

**正确做法**
判断入口统一用 `ModelConfig.needsLogin()`（`gemini.go`），不要在别处按模型名
字符串匹配 —— `gemini-3.7-flash-thinking` 名字里带 flash 但必须登录。

**注意能力不只由模型决定**：图片输入和超长转附件是**请求内容**触发的上传需求，
跟模型无关。任何「什么时候取 cookie」的判断都要把这两条算进去，否则 flash 的
长对话会从「转附件成功」退化成 400。

**验证方式**
构造匿名请求打 `3.1 Pro`，检查响应帧 `[42]` 里上游自报的模型名（记录在
`requests.upstream_model` 列，面板「实际模型」列可见）。

**适用范围**
新增模型、改降级策略、改挑号条件、改附件/上传逻辑。

---

## 4. 一个 Google 账号要固定从同一个出口 IP 走

**触发信号**
改挑号或挑代理的调度逻辑；或者想「让代理池均匀轮转以分散压力」。

**根因 / 约束**
cookie 池和代理池各自独立轮转的话，同一个账号会在几十个出口 IP 之间跳 ——
这在 Google 眼里正是**账号共享**的特征，会触发风控。

**正确做法**
`accounts.proxy_id` 记账号绑定的出口，`pickProxyPreferring(preferID)` 优先复用它。
两条不能破坏的规则：

- 绑定的出口 slot 满时可以**临时**借别的出口，但**不改写绑定**。
- 只在「没绑过」或「绑的出口已不可用」（`proxyUsableByID` 为假）时才写新绑定。
  拿本次实际用的出口无条件覆盖，会在换号时把好号的粘性打散。

粘性是**偏好**不是独占：出口被停用 / 删除 / 熔断后仍会自动改绑，可用性优先。

**验证方式**
面板「Cookie 池」的「出口」列，同一个号在连续多次请求后应当稳定显示同一个代理名。

**适用范围**
`cookie_pool.go`、`proxy.go`、`gemini.go` 里挑号与挑出口的逻辑。

---

## 5. 代理池读取失败时保留旧池子，绝不半截覆盖

**触发信号**
给 `loadProxies` 加逻辑，或在读 `proxies` 表的循环里写 `continue` 跳过出错的行。

**根因 / 约束**
`acquireSlot` 用 `len(proxyCache)==0` 判断「没配代理池」，于是**池子一空就退回直连**
—— 部署者的真实 IP 直接暴露给上游，日志上只看到偶发的直连请求，极难定位。

而这条路径每个请求都会走（`recordProxyResult` 结束就调），WAL 模式下并发 UPDATE
期间 `rows.Next()` 完全可能返回 SQLITE_BUSY。「偶发」就是这么来的。

**正确做法**
读一半失败时整个放弃本次刷新、保留上一次的池子。三个静默失效点都要堵：
`Query` 出错、`Scan` 出错、`rows.Err()` 非空 —— 一律 `return`，不要 `continue`。

同理 `recordProxyResult` 只改内存里的那一条，不整表重读：它自己刚发起过 UPDATE，
高并发下正是这对读写在 WAL 上撞出 SQLITE_BUSY。

**验证方式**
`fallback_direct` 保持默认关闭，观察日志里不应出现非预期的直连请求；
面板「请求记录」的出口列不应出现空值。

**适用范围**
`proxy.go` 的池子加载与状态回写，以及任何「读 DB 刷新内存缓存」的同类结构。

---

## 6. 限流 slot 按 `proxy_id` 分组，直连是 slot 0

**触发信号**
改并发 / RPM / RPH 限流，或新增一条会发上游请求的代码路径。

**根因 / 约束**
上游的封禁按**出口 IP** 算，所以限流额度必须按出口分桶，而不是全局一个计数器。
`ratelimit.go` 的 `slots` 是 `map[int64]*ipSlot`，key 就是 `proxy_id`，`0` 表示直连。

**正确做法**
任何发往 `gemini.google.com` 的请求都要先 `acquireSlot` 拿到 slot，并**配套**
`defer releaseSlot(picked.ID)`。拿不到就返回 429，不要绕过限流器直接发。

已知两条例外，都是有意的，别照着抄：

- `rotateAccount`（保活）打的是 `accounts.google.com`，不同域、不占 Gemini 出口的
  额度，所以只用 `proxyURLByID` 复用账号绑定的出口保持 IP 一致，不申请 slot。
- `handleAdminTest`（面板连通性测试）人工触发、一次性诊断，面板上已明示
  「不消耗限流配额、不写入请求记录」。

`checkAccountCookie`（cookie 检测）**是**走 slot 的 —— 它打的是 `gemini.google.com/app`。

**验证方式**
面板「距离封禁红线」里每个出口独立计数；压测时单个出口的并发不应超过
`per_ip_concurrent`。

**适用范围**
新增任何调用上游 HTTP 接口的代码路径。

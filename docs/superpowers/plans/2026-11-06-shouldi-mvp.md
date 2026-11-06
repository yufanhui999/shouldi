# ShouldI（帮我选）MVP 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现设计文档定义的三端 MVP：微信小程序（投票+意见+参考社区）+ Node API + 管理后台，达到可提审状态。

**Architecture:** 单仓三包（monorepo）。后端先行：Express + Sequelize + MySQL，接口即契约（DTO 先定）；小程序与管理后台消费同一套 API。内容安全、防刷、回访提醒全部在后端强制。

**Tech Stack:** Node.js 20 / Express / Sequelize / MySQL 8 / jest+supertest；uni-app(Vue3+TS)；Vue3 + Vite + Element Plus + Pinia。

**Spec:** `docs/superpowers/specs/2026-11-06-shouldi-design.md`（本计划从该文档出发，执行者需同时读 spec）

## 前置人工事项（M0，与开发并行，不阻塞编码）

- [ ] 注册微信小程序（个人主体）+ 类目「工具」；拿到 AppID/AppSecret
- [ ] 购买域名并**提交 ICP 备案**（关键路径 1~2 周）
- [ ] 申请订阅消息模板（回访提醒类目）
- [ ] 小程序后台配置「获取当前位置」权限声明 + 隐私保护指引
- [ ] 采购轻量云服务器 2C2G（备案通过后才能解析域名）

## Global Constraints（每个任务隐含遵守，值取自 spec）

- 词库禁用：UI 与接口文案不得出现「劝 / 立誓 / 宣判 / 服从」（spec D12/D13）
- 匿名铁律：任何列表/详情接口不得返回他人帖子的 user 归属（spec 5.5）
- 一人一票：`votes(post_id,user_id)` 唯一索引，并发冲突返回 409（spec 5.2）
- 频率限制：发帖 ≤3/天、意见 ≤20/天，按 user 计数（spec 5.2）
- 全部 UGC（标题/背景/选项文案/意见/回访）过 msgSecCheck；**微信接口调用失败时 fail-closed → status=pending**（spec D16，见 Review Focus #2）
- 审核策略：机审通过即 `published`；命中风险 → `pending` 人工（spec D16）
- 附近定义：5km 内 + 72h 内发布，距离升序；坐标为空的帖子不得进入附近结果（spec 4.1/5.2）
- 日志安全：openid 不得出现在任何日志/错误信息明文中（spec 5.5）
- 回访提醒：`published_at` 满 7 天且无 outcome（spec 5.2）
- 选项：2~10 个；卡片只送票数前 3 名 + `optionTotal`（spec D3）
- MVP 禁做清单见 spec 4.3，任何任务不得引入

## Review Focus（spec 隐含、任务测试须覆盖的五个高危输入）

1. **并发重复投票**：同一用户两个并发投票请求 → 恰有一个 201、另一个 409（唯一索引兜底，非应用层检查）。→ T8 测试
2. **微信审核接口故障/超时**：msgSecCheck 抛错或超时 → 帖子进 pending，不直接发布也不丢弃。→ T6 测试
3. **非公开帖越权访问**：`GET /api/posts/:id` 访问 pending/rejected/removed 帖 → 作者本人 200（含状态），非作者 404。→ T7 测试
4. **附近查询边界**：坐标为空的帖子、72h 之前的帖子不得出现在 near 结果。→ T7 测试
5. **匿名泄露**：断言列表/详情响应 JSON 中不存在他人 `userId` 字段。→ T7/T8 测试

---

## Part 0 · 仓库基建

### Task 1: Monorepo 脚手架

**Files:**
- Create: `server/`（package.json、`src/app.js`、`tests/`、.env.example）、`miniprogram/`（uni-app 默认模板）、`admin/`（Vite 脚手架）、根 `.gitignore`、根 `README.md`

**Interfaces:**
- Produces: 三包目录与根 README 里的启动说明；`server/src/app.js` 导出 `createApp()` 工厂

- [ ] **Step 1:** 建 `server`（npm init, 安装 express/sequelize/mysql2/dotenv/jsonwebtoken/bcryptjs/cors/node-cron/axios, dev: jest/supertest）、`admin`（`npm create vite@latest -- --template vue-ts`）、`miniprogram`（HBuilderX/CLI uni-app Vue3 模板；本地无编译环境则先建目录结构占位）
- [ ] **Step 2:** 根 README 写清三包职责与启动命令
- [ ] **Step 3:** 验证：`cd server && npm test`（空测试套件跑通）、`cd admin && npm run build` 成功
- [ ] **Step 4:** Commit `chore: monorepo scaffold (server/miniprogram/admin)`

---

## Part A · 后端核心（M1）

### Task 2: Express 应用工厂 + /health + 测试基建

**Files:**
- Create: `server/src/app.js`、`server/src/config/index.js`、`server/tests/setup.js`、`server/tests/health.test.js`

**Interfaces:**
- Produces: `createApp({ sequelize, wechat }) -> Express app`（依赖注入，供测试替换）；`GET /health` → `{ok:true}`；jest 配置读取 `.env.test`

- [ ] **Step 1:** 写失败测试：`GET /health` 期望 200 且 body `{ok:true}`
- [ ] **Step 2:** 跑 `npm test` 确认失败（路由不存在）
- [ ] **Step 3:** 实现 `createApp`（cors、json 解析、路由挂载点）、config 模块（env 读取，缺省值给测试）
- [ ] **Step 4:** 跑测试通过
- [ ] **Step 5:** Commit `feat(server): app factory with /health`

### Task 3: 数据模型与迁移

**Files:**
- Create: `server/src/models/`（index.js + 8 个模型文件：User, Post, PostOption, Vote, VoteLike, Outcome, Report, AdminUser, Event）
- Test: `server/tests/models.test.js`

**Interfaces:**
- Produces: `models` 导出全部模型与关联；字段与索引**逐字对齐 spec 5.3**（含 `UNIQUE(post_id,user_id)`、`UNIQUE(post_id)` on outcomes、`UNIQUE(vote_id,user_id)` on vote_likes、posts.status 枚举 pending/published/rejected/removed）
- Consumes: Task 2 的 sequelize 实例（测试用 sqlite 内存库跑唯一性，生产用 MySQL 8）

- [ ] **Step 1:** 写失败测试：sync 后建两条同 `(post_id,user_id)` 的 Vote 抛唯一冲突；outcomes 同 post 两条冲突
- [ ] **Step 2:** 跑测试确认失败
- [ ] **Step 3:** 实现模型与关联（Post hasMany PostOption/Vote/Outcome 等）
- [ ] **Step 4:** 跑测试通过
- [ ] **Step 5:** Commit `feat(server): data models per spec 5.3`

### Task 4: WeChatService（微信接口封装）

**Files:**
- Create: `server/src/services/wechat.js`
- Test: `server/tests/wechat.test.js`

**Interfaces:**
- Produces:
  - `code2Session(code) -> {openid}`（失败抛 `WechatError`）
  - `msgSecCheck(text) -> {pass:boolean, risky:boolean}`；**网络/HTTP 错误 → 返回 `{pass:false, risky:false, error:true}`**（fail-closed 语义，见 Review Focus #2）
  - `sendSubscribeMessage(openid, templateId, data) -> boolean`
  - 构造器注入 http 客户端，测试全 mock

- [ ] **Step 1:** 写失败测试：code2Session 成功/失败、msgSecCheck 正常通过/命中风险/接口抛错三种返回
- [ ] **Step 2:** 确认失败 → 实现（axios、access_token 内存缓存）
- [ ] **Step 3:** 测试通过 → Commit `feat(server): wechat service (code2session/msgSecCheck/subscribe)`

### Task 5: 登录（openid → JWT）+ 鉴权中间件

**Files:**
- Create: `server/src/routes/auth.js`、`server/src/middleware/auth.js`、`server/src/services/user.js`
- Test: `server/tests/auth.test.js`

**Interfaces:**
- Produces: `POST /api/auth/login {code}` → `{token, anonNo}`（新用户自动建号，`anon_no` 格式 `#` + 4 位随机数字，冲突重试）；`requireUser` 中间件（Bearer token → `req.user={id,openid}`，无效 401）
- Consumes: T4 `code2Session`

- [ ] **Step 1:** 失败测试：新 code 登录建用户返回 token+anonNo；同 openid 二次登录不新建（用户数+0）；无效 code → 502；无 token 访问受保护路由 → 401
- [ ] **Step 2:** 确认失败 → 实现（jsonwebtoken 签发，secret 进 config）
- [ ] **Step 3:** 测试通过 → Commit `feat(server): silent login + jwt auth`

### Task 6: 发帖 + 两级审核 + 限额

**Files:**
- Create: `server/src/routes/posts.js`（POST 部分）、`server/src/services/post.js`、`server/src/services/moderation.js`
- Test: `server/tests/posts-create.test.js`

**Interfaces:**
- Produces: `POST /api/posts {title, background, options:[{label,desc?}], lng?, lat?, city?}` → 201 `{post:PostDetailDTO}`；status=published 或 pending（pending 时响应含 `reviewing:true`）
  - 校验：title 5~40 字；background ≤500 字；options 2~10 个、label 非空去重 ≤20 字；desc ≤10 字
  - 审核：title/background/各 label 逐个 msgSecCheck；全部 pass → published；任一 risky 或任一调用 error → pending（Review Focus #2）
  - 限额：该 user 当日 published+pending 帖数 ≥3 → 429
- Produces: `PostDetailDTO = {id, title, background, status, options:[{index,label,desc,votes}], participantCount, reasonCount, hasOutcome, outcome?, createdAt}`（**不含 userId**）

- [ ] **Step 1:** 失败测试清单（每个一条断言）：正常发布→published；机审 risky→pending + reviewing:true；微信接口 mock 抛错→pending；标题 4 字→400；第 11 个选项→400；当日第 4 帖→429；响应无 userId 字段
- [ ] **Step 2:** 确认失败 → 实现路由/服务/审核服务
- [ ] **Step 3:** 测试通过 → Commit `feat(server): post creation with two-tier moderation`

### Task 7: 列表与详情（最新/热门/附近）

**Files:**
- Create: `server/src/routes/posts.js`（GET 部分）、`server/src/services/feed.js`
- Test: `server/tests/feed.test.js`

**Interfaces:**
- Produces:
  - `GET /api/posts?tab=new|hot|near&cursor=&pageSize=20&lng=&lat=` → `{list:[PostCardDTO], nextCursor}`
  - `PostCardDTO = {id, title, participantCount, reasonCount, hasOutcome, options:[{index,label,pct}]}`——options 取票数前 3（<3 个全给），每项 `pct` 按总票数四舍五入；**不含 userId**
  - tab 语义：new=published 按 published_at 倒序、游标分页；hot=published 按 vote_count 倒序再按 id；near=坐标非空 + published_at ≥ 72h 前 + 经纬度矩形（±0.05°）过滤 + Haversine ≤5km + 距离升序
  - `GET /api/posts/:id` → 详情 PostDetailDTO + `myVoteIndex`（本人已投时）；**非作者访问非 published → 404**（Review Focus #3）
- Consumes: T3 模型

- [ ] **Step 1:** 失败测试：new 分页顺序与游标；hot 排序；near 含坐标空帖排除、超 72h 排除、5km 外排除、距离升序（Review Focus #4）；卡片 options 只含前 3 名；越权访问 pending 帖非作者 404、作者 200；列表响应无 userId（Review Focus #5）
- [ ] **Step 2:** 确认失败 → 实现 feed 服务
- [ ] **Step 3:** 测试通过 → Commit `feat(server): feed apis (new/hot/near) + detail`

### Task 8: 投票 + 意见 + 点赞

**Files:**
- Create: `server/src/routes/votes.js`、`server/src/services/vote.js`
- Test: `server/tests/votes.test.js`

**Interfaces:**
- Produces:
  - `POST /api/posts/:postId/votes {optionIndex, reason?}` → 201 返回最新 PostDetailDTO（比例已解锁）；重复投票（含**并发双请求**，Review Focus #1）→ 409；reason ≤200 字且过 msgSecCheck（risky→pending 意见不展示？**否**：意见机审 risky 直接 400 提示修改，无人工队列——意见量大人审不现实）；post 非 published → 404；当日第 21 条带 reason 请求 → 429
  - 投票/意见成功时在同一事务内更新 `posts.vote_count` / `posts.reason_count` 冗余计数（T7 热门排序依赖）
  - `PATCH /api/votes/:id/reason {reason}`（仅本人、原 reason 为空时可用）→ 200
  - `POST /api/votes/:id/like` → 幂等切换 `{liked, likeCount}`
- Consumes: T4 msgSecCheck、T5 requireUser、T7 DTO 组装

- [ ] **Step 1:** 失败测试：投票成功返回比例；同一用户二次投票 409；`Promise.all` 并发两票恰一 201 一 409；意见超长 400；意见 risky 400；未投票直接 PATCH reason（伪造 vote id 他人）→ 403/404；点赞幂等切换；20/日限额
- [ ] **Step 2:** 确认失败 → 实现（事务内 `findOrCreate`+捕获唯一索引冲突）
- [ ] **Step 3:** 测试通过 → Commit `feat(server): vote + reason + like with anti-abuse`

### Task 9: 回访 + 定时提醒

**Files:**
- Create: `server/src/routes/outcomes.js`、`server/src/services/followup.js`（含 cron）、`server/src/jobs/followup-cron.js`
- Test: `server/tests/followup.test.js`

**Interfaces:**
- Produces:
  - `POST /api/posts/:id/outcome {chosenIndex, content}`（仅作者；content 必填 ≤300 字过 msgSecCheck；同帖重复 → 409）→ 201 PostDetailDTO（hasOutcome=true）
  - `runFollowupSweep(now) -> {sent:number}`：查 published_at ∈ [now-8d, now-7d] 且无 outcome 的帖 → `sendSubscribeMessage`；cron 每日 10:00 调它
- Consumes: T4 sendSubscribeMessage（测试 mock）

- [ ] **Step 1:** 失败测试：作者提交回访成功、非作者 403、重复 409、content 过审核；sweep 选中第 7~8 天窗口内的帖且跳过已有 outcome 与未发布帖，逐个发订阅消息
- [ ] **Step 2:** 确认失败 → 实现（jest fake timers 测 cron 注册）
- [ ] **Step 3:** 测试通过 → Commit `feat(server): outcome + 7-day followup sweep`

---

## Part B · 小程序（M2~M3）

> UI 任务无单测：验证方式 = 微信开发者工具运行 + 真机预览走查清单 + 提交。所有页面文案过一遍 Global Constraints 禁用词表。API 基址与 token 走 `miniprogram/src/utils/request.ts`。

### Task 10: uni-app 骨架 + 静默登录 + 请求封装

**Files:**
- Create: `miniprogram/src/utils/request.ts`（含 token 注入、401 重登）、`miniprogram/src/api/*.ts`（对齐 T5~T9 全部接口的类型定义）、tabBar 三项、`pages.json`

**Interfaces:**
- Consumes: T5~T9 的全部 HTTP 接口（类型从此处对齐，后端 DTO 为准）
- Produces: `api.auth.login()`、`api.posts.list(params)`、`api.posts.create(payload)`、`api.votes.cast(postId, payload)` 等函数供页面调用

- [ ] **Step 1:** 实现请求封装（onLaunch 静默 wx.login → `POST /api/auth/login` → 存 token）
- [ ] **Step 2:** 验证：开发者工具 Network 面板看到登录请求成功、token 持久化
- [ ] **Step 3:** Commit `feat(mp): skeleton + silent login + typed api layer`

### Task 11: 广场页

**Files:**
- Create: `miniprogram/src/pages/feed/index.vue`、`miniprogram/src/components/PostCard.vue`

- [ ] **Step 1:** 实现三 tab（附近/最新/热门）、PostCard（标题 + 前 3 选项比例条 + 「N 人参与 · M 条意见」+ 已回访徽章 + >3 选项时「共 N 个选项」）、触底分页、下拉刷新、附近首次点击触发位置授权弹层（拒接→提示并回到最新）
- [ ] **Step 2:** 验证走查：三个 tab 数据正确；比例条与后端 pct 一致；分页与刷新；空状态文案
- [ ] **Step 3:** Commit `feat(mp): feed page with 3 tabs`

### Task 12: 详情页

**Files:**
- Create: `miniprogram/src/pages/detail/index.vue`、`miniprogram/src/components/VoteOptions.vue`、`ReasonList.vue`

- [ ] **Step 1:** 实现未投票态（选项卡「投它」）、投票后解锁比例 + 「你的选择」标记、意见流（立场标签「投给了X」+ 点赞）、底部意见输入条（未投票点击 → toast「先投一票，才能发表意见」）、回访块置顶、转发按钮、转盘入口、右上角「举报」入口（调 T18 的举报接口，reason ≤100 字）
- [ ] **Step 2:** 验证走查：投票→比例动画出现→意见可发；他人帖无任何作者信息展示
- [ ] **Step 3:** Commit `feat(mp): detail page with vote-gated reasons`

### Task 13: 发布页

**Files:**
- Create: `miniprogram/src/pages/publish/index.vue`

- [ ] **Step 1:** 实现表单（标题/背景/选项 2~10 动态增删，前端校验与后端一致）、匿名说明卡、提交三态（published → 成功页请求订阅授权 `wx.requestSubscribeMessage`；pending → 显示「审核中」）、「发布，让路过的人拦住我」按钮
- [ ] **Step 2:** 验证走查：校验边界（4 字标题、11 选项被拦）；成功后广场置顶可见
- [ ] **Step 3:** Commit `feat(mp): publish page`

### Task 14: 我的页 + 回访

**Files:**
- Create: `miniprogram/src/pages/mine/index.vue`、`miniprogram/src/components/OutcomeSheet.vue`

- [ ] **Step 1:** 实现：我的求助（状态 chip：审核中/投票中/待回访/已回访 + 「记录选择」按钮）、回访 Sheet（选最终选项必填 + 感受必填）、我投过的列表
- [ ] **Step 2:** 验证走查：提交回访后详情页顶部出现回访块、广场卡片出现徽章
- [ ] **Step 3:** Commit `feat(mp): mine page + outcome flow`

### Task 15: 命运转盘 + 分享卡片

**Files:**
- Create: `miniprogram/src/components/FateWheel.vue`

- [ ] **Step 1:** 转盘组件（扇区=选项、旋转动画、结果提示、可重转，视觉对照原型 index.html）；`onShareAppMessage` 返回 `{title: 帖子标题, path: 详情页路径}`（MVP 用微信默认卡片样式，标题后缀实时票数文案如「· 412 人参与」）
- [ ] **Step 2:** 验证走查：真机转发到群，点开直达详情页
- [ ] **Step 3:** Commit `feat(mp): fate wheel + share card`

### Task 16: 附近与位置

**Files:**
- Modify: `miniprogram/src/pages/publish/index.vue`（可选「携带位置」开关）、`pages.json`（permission 声明）
- Create: `miniprogram/src/utils/geo.ts`

- [ ] **Step 1:** 发布页可选位置（wx.getLocation，失败→toast 可跳过）；附近 tab 传 lng/lat 调 near；授权拒绝的降级文案
- [ ] **Step 2:** 验证走查：带位置的帖出现在附近；不带的绝不出现
- [ ] **Step 3:** Commit `feat(mp): optional location + nearby integration`

---

## Part C · 管理后台（M4）

### Task 17: admin 后端 + 登录 + 守卫

**Files:**
- Create: `server/src/routes/admin-auth.js`、`server/src/middleware/admin-auth.js`、seed 脚本（建首位管理员）
- Create: `admin/`（Element Plus 布局、登录页、路由守卫、axios 拦截器）
- Test: `server/tests/admin-auth.test.js`

**Interfaces:**
- Produces: `POST /api/admin/login {username,password}` → `{token}`（bcrypt 校验）；`requireAdmin` 中间件；admin JWT 与用户 JWT 签发 audience 不同、互不通用（**测试断言用户 token 访问 admin 接口 401**）

- [ ] **Step 1:** 失败测试：正确密码 200、错误 401、用户 token 访问 401
- [ ] **Step 2:** 实现 + admin 前端登录与守卫 → 测试通过
- [ ] **Step 3:** Commit `feat(admin): admin auth (backend + console login)`

### Task 18: 审核队列 + 帖子管理

**Files:**
- Create: `server/src/routes/admin-posts.js`（列表/详情/通过/驳回/下架/搜索）、`admin/src/views/Review.vue`、`Posts.vue`
- Test: `server/tests/admin-posts.test.js`

**Interfaces:**
- Produces: `GET /api/admin/posts?status=pending|published|rejected|removed|reported`（reported = 有未处理举报的已发布帖）、`POST /api/posts/:id/report {reason}`（任何登录用户，≤100 字，幂等：同一用户同一帖重复举报忽略）、`POST /api/admin/posts/:id/approve|reject{reason}|remove{reason}`；驳回后作者端详情显示「未通过审核：原因」（PostDetailDTO 增加 `rejectedReason`，仅作者可见）；举报处理 `POST /api/admin/reports/:id/dismiss`
- 验证走查：审核队列通过→帖子上广场；驳回→作者可见原因；下架→用户端 404；举报帖出现在 reported 队列

- [ ] **Step 1:** 失败测试：pending 列表只含待审；report 建立记录且重复举报不新增；approve 后 status=published；reject 必填 reason；removed 帖详情接口对非作者 404
- [ ] **Step 2:** 实现后端 + 两张管理页面 → 测试通过
- [ ] **Step 3:** Commit `feat(admin): review queue + post management`

### Task 19: 用户管理 + 数据看板

**Files:**
- Create: `server/src/routes/admin-users.js`、`admin-stats.js`、`admin/src/views/Users.vue`、`Dashboard.vue`
- Test: `server/tests/admin-stats.test.js`

**Interfaces:**
- Produces: `GET /api/admin/users?banned=`、`POST /api/admin/users/:id/ban|unban`（封禁者发帖/投票 → 403，**测试覆盖**）；`POST /api/events {type}`（登录用户记录 pv/dau/share_click，T10 启动时上报一次）；`GET /api/admin/stats` → `{posts, votes, reasons, outcomes, dau7:[{date,count}], participationAvg}`（成功指标表的可视化数据源，dau7 取 events 按日去重 user 计数）

- [ ] **Step 1:** 失败测试：封禁用户投票 403；stats 数值与种子数据一致
- [ ] **Step 2:** 实现接口 + Dashboard 五指标卡 + 7 日趋势折线 + 用户页 → 测试通过
- [ ] **Step 3:** Commit `feat(admin): user management + metrics dashboard`

---

## Part D · 上线（M5）

### Task 20: 部署与提审（一键脚本方案）

**Files:**
- Create: `deploy/nginx.conf`、`deploy/ecosystem.config.js`（PM2 cluster ×2）、`deploy/backup.sh`（每日 mysqldump）、`deploy/deploy.sh`、`server/scripts/migrate.js`、`server/.env.production.example`、`docs/release-checklist.md`

**Interfaces:**
- Produces: `deploy.sh` = 服务器一键部署，依次执行：`git pull` → `cd server && npm ci --omit=dev` → `node scripts/migrate.js`（sequelize.sync 增量建表/加列，不做破坏性变更）→ `pm2 reload ecosystem.config.js`（零停机）→ `cd admin && npm ci && npm run build`（产物复制到 Nginx 目录）
- Produces: `docs/release-checklist.md` 记录部署策略：后端改动秒级热更、免打包；小程序仅在 UI/交互变更时提审（攒批次，勿逐 commit 提审）；开发期用体验版（上传即生效免审核）
- Consumes: 全部前序任务的产物

- [ ] **Step 1:** 服务器初始化：Node 20 / MySQL 8 / Nginx / PM2；Nginx 反代 api 域名 + 托管 admin 静态；certbot HTTPS；crontab 挂 `backup.sh`（每日 mysqldump 保留 7 份）
- [ ] **Step 2:** 编写 `deploy.sh` 与 `migrate.js`；**零停机验证**：后台循环 `curl /health` 期间执行 `./deploy.sh`，断言全程无失败响应；再验证真实改动：改一处接口文案 → `git push` → `./deploy.sh` → `curl` 确认新文案生效
- [ ] **Step 3:** 小程序后台配置（request 合法域名、隐私指引、订阅消息模板、类目核对）；上传体验版，扫码可用
- [ ] **Step 4:** 全链路真机回归（登录→发帖→投票→意见→回访→附近→转发）+ 首次提审
- [ ] **Step 5:** Commit `chore: one-key deploy script + release checklist`

---

## 依赖与顺序

T1 → T2 → T3 → T4 → T5 → T6 → T7 → T8 → T9（后端串行，接口即契约）
T10 依赖 T5；T11~T16 依赖 T10 与对应后端任务；T17~T19 依赖 T5/T3；T20 最后。
Part B 与 Part C 可并行（不同包，互不冲突）。

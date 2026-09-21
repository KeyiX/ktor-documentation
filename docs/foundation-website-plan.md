# Foundation Website — Implementation Plan (v1, for discussion)

> 状态：草案，待讨论确认。确认后新建私有仓库开始实施。

## 0. 目标回顾

| # | 需求 | 方案要点 |
|---|------|---------|
| 1 | 美观大气、专业 | 参考 Gates / Ford / Rockefeller / Wellcome / charity: water 的版式：大图 hero + 衬线标题、使命宣言、影响力数字、项目卡片、最新动态、捐赠 CTA、信息型页脚 |
| 2 | Home / News / Contact + 标准分页 | 建议加 About、Programs(What We Do)、Donate(可先外链)、Privacy；共 6 个一级导航 |
| 3 | News 增删查改，仅 admin | Cloudflare D1 存储 + `/admin` 后台，Cloudflare Access 做登录（邮箱 OTP / Google），Worker 内再校验 JWT |
| 4 | GitHub 私有仓库，dev / release 分支 | GitHub Actions：push dev → staging 环境；push release → production 环境；release 分支受保护 |
| 5 | 最便宜且稳定，TTFB < 200ms | Cloudflare Workers(静态资源 + 边缘 SSR) + D1 + R2，实际成本 $0/月 + 域名费 |
| 6 | 域名后配 | 域名只在 `wrangler.toml` production env 的 `routes` 和 GitHub Variable `SITE_URL` 两处出现，上线前用 `*.workers.dev` |

## 1. 技术选型

**框架：Astro 5 (+ React islands)**
- 内容型站点首选，公共页面默认 0 JS，Lighthouse 容易到 95+。
- 官方 `@astrojs/cloudflare` 适配器，直接绑定 D1 / R2 / KV。
- 仅在需要交互的地方用 React island：导航、联系表单、后台富文本编辑器。
- 备选：Next.js + Vercel。开发体验好，但 Vercel Hobby 禁止商业用途（非营利+无薪志愿者可能符合但存在灰色地带），Pro $20/月；Next.js 在 Cloudflare 上需要 OpenNext，复杂度更高。

**托管：Cloudflare Workers（含 Static Assets）**
- 静态资源请求免费且不计量；动态请求（news 页、admin、API）免费额度 10 万/天。
- D1（SQLite）：免费 5GB、500 万行读/天、10 万行写/天，2026-09-01 起超限报错而非计费，对小站足够。
- R2（图片存储）：免费 10GB，出口流量免费。
- Cloudflare Access：免费 ≤ 50 用户，可一键保护 `workers.dev` 与自定义域名。
- Turnstile（表单防机器人）：免费。
- 全球 300+ 节点，静态 HTML 边缘命中 TTFB 通常 20–80ms；边缘 SSR + D1 查询约 30–120ms。
- 风险：若主要受众在中国大陆，Cloudflare 访问质量差，需要改用国内云 + ICP 备案，方案需重做（见 §9 待确认问题）。

**成本估算**

| 项目 | 免费额度 | 预计月费 |
|------|---------|---------|
| Workers + Static Assets | 10 万动态请求/天 | $0（超出后 Paid $5/月） |
| D1 | 5GB / 500 万读/天 | $0 |
| R2 | 10GB | $0 |
| Access | 50 用户 | $0 |
| Turnstile | 无限 | $0 |
| Resend（邮件通知） | 3000 封/月 | $0 |
| 域名 | — | 约 $10–15/年（Cloudflare Registrar 成本价） |

## 2. 站点结构与设计

**页面**
- `/` Home：hero（大图 + 一句使命 + 2 个 CTA）、我们是谁、影响力数字、重点项目 3 卡、最新动态 3 卡、捐赠/参与横幅
- `/about` About：使命愿景、历史、团队/理事会、合作伙伴
- `/programs` What We Do：项目列表；`/programs/[slug]` 可选详情
- `/news` 新闻列表（分页，10/页）；`/news/[slug]` 详情
- `/contact` 联系方式 + 表单 + 地图（静态图或 OpenStreetMap 嵌入，避免 Google Maps 拖慢）
- `/donate` 先外链到捐赠平台（Stripe Payment Link / PayPal Giving Fund 等，按国家定）
- `/privacy`、`/404`
- `/admin/*` 后台（不进 sitemap，`noindex`）

**视觉系统**
- 字体：展示用衬线（Fraunces 或 Newsreader）+ 正文无衬线（Inter）；通过 `@fontsource` 自托管，`font-display: swap`，预加载 2 个字重。
- 配色：一个深主色（如 forest green `#1F3D2B` 或 navy `#14213D`）+ 暖色强调（琥珀/赤陶）+ 米白背景，大量留白。
- 组件：Tailwind CSS v4 + 少量自定义组件（Button、Card、SectionHeading、Prose）。
- 图片：Astro `<Image>` 输出 AVIF/WebP，响应式 `srcset`，hero 图 `fetchpriority=high` 预加载。
- 无障碍：WCAG AA 对比度、键盘导航、`prefers-reduced-motion`。
- 初期用 Unsplash 占位图，替换为基金会真实照片后效果才会真正"大气"。

## 3. News 后台（CRUD）

**数据模型（D1 / SQLite）**
```sql
posts(
  id INTEGER PK, slug TEXT UNIQUE, title TEXT, excerpt TEXT,
  body_html TEXT, cover_key TEXT, status TEXT CHECK(status IN ('draft','published')),
  published_at TEXT, created_at TEXT, updated_at TEXT, author_email TEXT
)
media(id, key UNIQUE, filename, content_type, size, width, height, created_at)
contact_messages(id, name, email, subject, message, created_at, read_at)
```
迁移文件放在 `migrations/`，CI 用 `wrangler d1 migrations apply` 自动执行。

**后台页面**
- `/admin` 文章列表（状态筛选、搜索、发布/取消发布、删除二次确认）
- `/admin/posts/new`、`/admin/posts/[id]` 编辑器：标题、slug（自动生成可改）、摘要、封面、正文、发布时间、状态
- `/admin/media` 图片库（上传到 R2，浏览端先压缩到 ≤1600px WebP，避免付费图片变换）
- `/admin/messages` 联系表单收件箱

**编辑器**：Tiptap（React island），输出 HTML，服务端用 `sanitize-html` 白名单清洗后入库。备选 Markdown + 预览，实现更简单但对非技术人员不友好。

**鉴权**
1. Cloudflare Access 应用保护 `/admin/*` 与 `/api/admin/*`，策略 = 允许的邮箱列表，登录方式用 One-time PIN（邮箱验证码）或 Google。
2. Worker 中间件校验 `Cf-Access-Jwt-Assertion` JWT（Access 的 JWKS 公钥），并核对邮箱在 `ADMIN_EMAILS` 环境变量中，防止绕过。
3. 上线前用 `workers.dev` 域名 + 一键 Access；接域名后同一 Access 应用加一个域名即可。
4. 若不想依赖 Access，可换成应用内 Google OAuth（`arctic` + KV session），约 150 行代码。

**缓存策略**：公共 news 页边缘 SSR，`Cache-Control: public, s-maxage=60, stale-while-revalidate=600`；admin 写入后接受最多 60s 延迟。静态页全部预渲染。

## 4. 联系表单
- 前端 React island，Turnstile 验证。
- 提交到 `/api/contact`：写入 `contact_messages`，并通过 Resend 发邮件到基金会邮箱（未接域名前用 Resend 的默认发件域名）。
- 速率限制：Workers 内置 Rate Limiting binding，每 IP 5 次/10 分钟。

## 5. 仓库与分支
- 新建私有仓库（例：`keyix/<foundation>-website`）；本仓库是 ktor-documentation 的 fork，不适合复用。
- 分支：`dev`（默认分支、集成）、`release`（生产）。功能分支 → PR → dev；dev → PR → release。
- 保护规则：release 仅允许 PR 合入，要求 CI 通过；dev 要求 CI 通过。
- 目录结构：
```
src/pages, src/components, src/layouts, src/lib (db, auth, email),
src/content (静态文案), migrations/, public/, wrangler.toml,
.github/workflows/{ci,deploy-staging,deploy-production}.yml
```

## 6. CI/CD（GitHub Actions）
- `ci.yml`（所有 PR）：pnpm install → typecheck → lint → vitest → `astro build` → Playwright e2e（本地 `wrangler dev` + 临时 D1）→ Lighthouse CI 断言 performance ≥ 95。
- `deploy-staging.yml`（push dev）：`wrangler d1 migrations apply --env staging` → `wrangler deploy --env staging`。
- `deploy-production.yml`（push release）：同上 `--env production`，需要 GitHub Environment `production` 审批（可选）。
- Secrets：`CLOUDFLARE_API_TOKEN`、`CLOUDFLARE_ACCOUNT_ID`、`RESEND_API_KEY`、`TURNSTILE_SECRET`；Variables：`SITE_URL`、`ADMIN_EMAILS`。
- staging 与 production 使用独立的 D1 数据库和 R2 bucket。

## 7. 域名接入（后置，约 30 分钟）
1. 域名 DNS 迁到 Cloudflare（或直接在 Cloudflare Registrar 注册）。
2. `wrangler.toml` `[env.production]` 加 `routes = [{ pattern = "example.org", custom_domain = true }]`，`www` 用 Redirect Rule 301 到主域。
3. GitHub Variable `SITE_URL` 改为正式域名（影响 canonical、sitemap、OG 链接）。
4. Access 应用增加正式域名；Resend 验证发件域名；Turnstile 增加域名。
5. 合并 release 触发部署，完成。

## 8. 性能预算
- 公共页 HTML ≤ 40KB（gzip）、JS ≤ 30KB（仅 island）、hero 图 ≤ 150KB AVIF。
- TTFB：静态页 < 100ms，news 页 < 150ms（全球中位数）。LCP < 1.5s，CLS = 0。
- 用 Lighthouse CI + Cloudflare Web Analytics（免费）持续监控。

## 9. 实施阶段（预计 6–8 个工作日）
1. 仓库 + 脚手架 + Cloudflare 资源 + CI/CD，staging 可访问 hello world
2. 设计系统 + 全部公共页面（占位内容）
3. D1 schema + Admin CRUD + 媒体上传 + Access 鉴权
4. 联系表单 + 邮件 + Turnstile
5. SEO（sitemap、OG、JSON-LD Organization）、性能与无障碍调优、Lighthouse CI
6. 域名接入手册 + 后台使用手册（交接文档）

## 10. 待确认问题
1. 基金会名称、领域、所在国家；网站语言（仅英文，还是中英双语需要 i18n）。
2. 主要访问者所在地区（若为中国大陆，托管方案需换）。
3. 管理员人数与登录方式偏好（邮箱验证码 / Google 账号）。
4. 联系表单邮件收件地址。
5. 是否需要 Donate 页面，用哪个捐赠平台。
6. 是否有品牌素材（logo、配色、照片）。
7. 框架选 Astro（推荐）还是 Next.js。
8. 新私有仓库名称，由谁创建。

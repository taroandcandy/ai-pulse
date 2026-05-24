# Vercel 部署环境变量配置

本文记录 `ai-pulse` 部署到 Vercel 时需要配置的环境变量，以及部署后常见检查项。

## 必须配置

这些变量是线上页面、Supabase 数据读取、定时任务接口正常运行所必需的。

```env
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
CRON_SECRET=
```

### NEXT_PUBLIC_SUPABASE_URL

Supabase 项目的根地址。

正确格式：

```text
https://xxxx.supabase.co
```

不要填写成：

```text
https://xxxx.supabase.co/rest/v1
https://xxxx.supabase.co/auth/v1
https://xxxx.supabase.co/storage/v1
```

如果填成带路径的地址，线上可能会出现 Supabase/PostgREST 报错，例如：

```text
PGRST125 Invalid path specified in request URL
```

### NEXT_PUBLIC_SUPABASE_ANON_KEY

Supabase 的 `anon public` key。

这个 key 通常用于浏览器端公开访问，但当前项目主要还是通过服务端读取数据。

### SUPABASE_SERVICE_ROLE_KEY

Supabase 的 `service_role` key。

这个变量只能放在服务端环境变量中，不能提交到 Git，也不能暴露到浏览器端。当前项目的抓取、入库、邮件日志等服务端逻辑会用它读写 Supabase 数据库。

### CRON_SECRET

定时任务接口的访问密码，用来保护这些接口：

```text
/api/cron/fetch
/api/cron/digest
```

`CRON_SECRET` 不是从 Vercel 或 Supabase 获取的，而是自己生成的一串随机字符串。只要 Vercel 环境变量和请求 URL 中的 `secret` 参数保持一致即可。

示例：

```env
CRON_SECRET=your-random-secret
```

手动触发抓取：

```text
https://your-domain.vercel.app/api/cron/fetch?secret=your-random-secret
```

手动触发邮件摘要：

```text
https://your-domain.vercel.app/api/cron/digest?secret=your-random-secret
```

如果返回：

```json
{ "error": "Unauthorized" }
```

通常说明 `CRON_SECRET` 不一致，或者 Vercel 修改环境变量后还没有重新部署。

## 邮件功能需要配置

如果要启用邮件日报，还需要配置：

```env
RESEND_API_KEY=
EMAIL_FROM=
DIGEST_TO_EMAIL=
```

### RESEND_API_KEY

Resend 邮件服务的 API key。

### EMAIL_FROM

发件人地址。

示例：

```text
AI Pulse <onboarding@resend.dev>
```

如果使用自己的域名邮箱，需要先在 Resend 中完成域名验证。

### DIGEST_TO_EMAIL

日报接收邮箱。

## 可选配置

```env
NEXT_PUBLIC_APP_URL=
```

建议填写线上主页地址：

```text
https://aipulse-nine.vercel.app
```

当前代码中它不是核心必需项，但后续如果邮件内容或页面链接需要生成站点 URL，可以使用这个变量。

## 当前不需要配置

当前代码没有读取这些变量：

```env
APP_BASE_URL=
ADMIN_API_KEY=
```

除非后续新增后台管理接口或其他功能，否则不用配置。

## 配置位置

在 Vercel 后台进入：

```text
Project -> Settings -> Environment Variables
```

环境选择：

```text
Production
```

如果需要预览部署也能访问数据库，可以同时配置到 `Preview`。

## 重要注意事项

Vercel 环境变量修改后，不会自动影响已经部署好的版本。

每次新增或修改环境变量后，都需要重新部署：

```text
Deployments -> 最新部署右侧 ... -> Redeploy
```

否则线上应用读到的仍然是旧环境变量。

## Supabase 数据库初始化

环境变量配置完成后，还需要确保 Supabase 数据表已经创建。

项目中的迁移文件：

```text
supabase/migrations/20260524084500_create_rss_pipeline.sql
supabase/migrations/20260524092000_create_email_digest_pipeline.sql
```

可以在 Supabase Dashboard 的 SQL Editor 中按顺序执行。

如果没有执行迁移，页面或接口可能会报表不存在，例如：

```text
relation "sources" does not exist
```

## 部署成功的判断

Vercel 部署详情页中看到以下状态，说明构建和生产部署成功：

```text
Status: Ready Latest
Environment: Production Current
Build Logs: 成功
```

如果部署详情页缩略图已经能看到页面内容和数据，例如 RSS 数量、文章数量、文章列表，说明线上应用已经可以被 Vercel 正常访问。

如果本地浏览器打开 `vercel.app` 域名出现：

```text
ERR_CONNECTION_TIMED_OUT
```

这通常是本地网络访问 Vercel 域名不稳定，不是项目代码或 Vercel 部署失败。可以尝试切换网络、使用手机热点、使用代理，或绑定自定义域名。

## 常用访问地址

主页：

```text
https://aipulse-nine.vercel.app
```

手动抓取：

```text
https://aipulse-nine.vercel.app/api/cron/fetch?secret=your-random-secret
```

手动邮件摘要：

```text
https://aipulse-nine.vercel.app/api/cron/digest?secret=your-random-secret
```

## 手动抓取接口返回值

访问 `/api/cron/fetch` 后，理想返回类似：

```json
{
  "startedAt": "2026-05-24T12:00:00.000Z",
  "finishedAt": "2026-05-24T12:00:10.000Z",
  "sourceCount": 3,
  "failedCount": 0,
  "insertedCount": 12,
  "results": []
}
```

字段含义：

```text
sourceCount   启用的 RSS 源数量
failedCount   抓取失败的 RSS 源数量
insertedCount 本次新增入库文章数量
results       每个 RSS 源的抓取结果
```

如果 `failedCount > 0`，不一定代表系统坏了。它可能只是某些 RSS 源在 Vercel 节点访问慢、超时或被目标站点拒绝。

当前 RSS 抓取超时时间是 60 秒，因此 `/api/cron/fetch` 可能比普通页面慢很多。

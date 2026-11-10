# SubsTracker（订阅管理与到期提醒系统）

> 本文档为中文项目说明。介绍当前仓库的功能、运行结构、部署方式、认证方式、配置管理和维护方法。

## 1. 项目简介

SubsTracker 是运行在 Cloudflare Workers 上的轻量级订阅管理系统。它用于记录软件、会员、云服务等订阅的周期和到期时间，并按计划发送提醒。

主要能力：
- 管理订阅：新增、编辑、删除、启用或停用。
- 到期提醒：按提前天数或小时数计算提醒时间。
- 周期管理：支持常规周期、手动续订和续订记录。
- 支付追踪：记录金额、币种、支付历史，并汇总支出。
- 日期显示：支持时区设置和农历显示。
- 多渠道通知：Telegram、NotifyX、企业微信、Webhook、Resend 邮件、Bark 等（以当前代码和配置页实际支持项为准）。

## 2. 技术架构

- 运行平台：Cloudflare Workers。
- 数据存储：Cloudflare KV，绑定名称为 `SUBSCRIPTIONS_KV`。
- 服务端：主要逻辑集中在根目录 `index.js`。
- 前端：登录页、管理页、配置页和仪表盘由 Worker 返回 HTML，页面通过 Fetch 调用 `/api/*`。
- 定时任务：Worker 的 `scheduled()` 入口负责检查即将到期的订阅并发送通知。

### 主要请求路径

| 路径 | 用途 |
|---|---|
| `/` | 登录页面 |
| `/api/login` | 管理员登录 |
| `/api/logout` | 退出登录 |
| `/admin` | 订阅管理页面 |
| `/admin/config` | 系统配置页面 |
| `/admin/dashboard` | 支出统计仪表盘 |
| `/api/config` | 读取或保存系统配置 |
| `/api/notify/*` | 第三方触发通知的接口 |

## 3. 登录认证与凭证说明

当前代码的认证结构：
1. 登录页面将用户名和密码通过 HTTPS POST 到 `/api/login`。
2. Worker 从 KV 的 `config` 项读取管理员配置。
3. 登录成功后，服务端签发 HS256 JWT。
4. JWT 放入名为 `token` 的 HttpOnly Cookie。
5. 管理页面和受保护 API 从 Cookie 读取 Token，并调用 JWT 验证函数。
6. 退出登录时，服务端通过过期 Cookie 让浏览器删除本地 Token。

重要说明：JWT 是签名令牌，不是加密后的用户资料。不要把密码或 API Secret 放入 JWT Payload。

### 当前版本需要注意的安全事项

请在公网部署前检查并完成以下事项：
- 修改旧版本默认管理员密码，不能继续使用 `admin/password`。
- 管理员密码应使用带随机盐的慢速哈希保存，不能明文存储。
- JWT 必须包含并验证 `exp`，不能只依赖浏览器 Cookie 的 Max-Age。
- JWT 签名密钥必须由密码学安全随机数生成，并妥善保存。
- 登录 Cookie 应设置 `HttpOnly; Secure; SameSite=Strict`。
- 配置 API 不得把 Bot Token、API Key、Webhook URL、第三方访问 Token 返回给浏览器。
- 第三方 API 优先通过 `Authorization: Bearer <token>` 传递凭证；避免把 Token 放在 URL 路径或查询参数中。
- 对登录接口增加失败次数限制或速率限制。

以上是部署安全要求，不代表当前主分支已经全部实现。合并安全整改前，请先在独立分支验证兼容性。

## 4. Cloudflare 部署

### 准备工作
- 一个 Cloudflare 账号。
- 一个 Worker。
- 一个 KV Namespace。
- 需要使用的通知渠道账号及对应凭证。

### 部署步骤
1. Fork 本仓库，或将代码导入自己的 GitHub 仓库。
2. 在 Cloudflare Dashboard 创建 Worker。
3. 将仓库中的 Worker 代码部署到 Worker。
4. 创建 KV Namespace。
5. 在 Worker Settings 的 Bindings 中添加 KV Namespace，变量名必须是 `SUBSCRIPTIONS_KV`。
6. 确认 Worker 绑定了正确的 KV Namespace。
7. 访问 Worker 域名，使用初始管理员配置登录。
8. 首次登录后立即设置强管理员密码。
9. 配置通知渠道并发送测试通知。
10. 检查 Cron Trigger，确认定时任务按预期执行。

### 定时任务和时区
Cloudflare Cron 使用 UTC 时间。系统配置中的时区用于日期计算和显示，两者不是同一个设置。修改提醒时间时，必须同时检查 Cron 表达式和系统时区。

## 5. 配置项说明

| 配置项 | 用途 | 是否敏感 |
|---|---|---|
| `ADMIN_USERNAME` | 管理员用户名 | 否 |
| `ADMIN_PASSWORD` | 管理员密码或密码哈希 | 是 |
| `JWT_SECRET` | JWT 签名密钥 | 是 |
| `THIRD_PARTY_API_TOKEN` | 外部通知 API 访问令牌 | 是 |
| `TG_BOT_TOKEN` | Telegram Bot 凭证 | 是 |
| `NOTIFYX_API_KEY` | NotifyX API 凭证 | 是 |
| `RESEND_API_KEY` | 邮件发送 API 凭证 | 是 |
| `BARK_DEVICE_KEY` | Bark 设备凭证 | 是 |
| `WEBHOOK_URL` / `WEBHOOK_HEADERS` | Webhook 地址及请求头 | 通常是 |
| `TIMEZONE` | 日期计算和显示时区 | 否 |
| `NOTIFICATION_HOURS` | 允许发送提醒的小时 | 否 |

不要把真实密钥提交到 GitHub、截图、问题反馈或公开日志中。

## 6. 数据存储与备份

主要业务数据保存在 KV 中：
- `config`：系统设置和通知渠道配置。
- `subscriptions`：订阅及其支付历史。

建议定期导出并备份 KV 数据。修改、删除或迁移订阅记录前，先完成备份。不要把真实用户数据放进公开示例文件。

## 7. 常见问题

**登录后仍然跳回登录页**
- 检查 Worker 是否使用 HTTPS。
- 检查浏览器是否保存 Cookie。
- 检查 `SUBSCRIPTIONS_KV` 是否绑定正确。
- 检查 JWT 密钥是否在不同 Worker 环境间发生变化。

**订阅没有自动提醒**
- 检查 Cron Trigger 是否启用。
- 检查订阅是否启用、提醒时间是否正确。
- 检查系统时区与 Cron 的 UTC 时间换算。
- 查看 Worker Logs 中的任务执行和通知渠道错误。

**通知发送失败**
- 检查对应渠道是否启用。
- 检查 Token、Webhook、收件人等配置。
- 使用配置页的测试通知功能。
- 不要把完整密钥复制到日志或公开 Issue。

## 8. 维护建议

- 修改认证逻辑后，必须测试登录、错误密码、Token 过期、退出登录和配置保存。
- 修改 KV 数据结构前，先设计迁移和回滚方案。
- 修改通知渠道后，至少完成一次真实测试发送。
- 生产环境不要依赖默认密码或调试页面。
- 建议在发布前通过 PR 审查和备份流程。

## 9. 许可证

请以仓库中的 LICENSE 文件为准。

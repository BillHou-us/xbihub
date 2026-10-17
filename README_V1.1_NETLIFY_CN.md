# XBIHUB 官网 V1.1｜Netlify 升级与部署指南

本版本以用户提供的官网源代码为基础，保留公开展示页面，新增服务端合作伙伴入口 `secure.html`、Netlify Functions 后端接口及 Supabase 数据库结构。

## 一、已经完成的安全改造

- 旧登录页中的 `admin@xbihub.com / admin` 硬编码登录通道已移除。
- 旧 `login.html`、`apply.html`、`portal.html`、`admin.html` 统一跳转到新的 `secure.html`，不再运行旧版浏览器模拟登录代码。
- 禁用旧版 JavaScript 自动生成演示申请和协议数据。
- 新入口使用 Supabase Auth，服务端通过 Secure、HttpOnly、SameSite=Lax Cookie 识别会话。
- 注册用户默认只有 `applicant` 权限；不自动创建管理员。
- Supabase Service Role Key 只允许放在 Netlify Functions 的服务端环境变量中。
- 正式电子签名尚未接入，相关接口默认拒绝签署请求。
- 节点激活必须有供应商验证过的已签协议，并满足付款确认与培训完成条件。

**注意：** 代码包不能远程停用线上 Netlify 的旧账号。必须部署新版本才会替换线上页面；此前公开过的密码也应轮换。

## 二、架构

- 前端：原有静态 HTML/CSS/JS + `secure.html`
- 托管与后端：Netlify + Netlify Functions
- 认证和数据库：Supabase Auth + Postgres
- 可选状态邮件：Resend（需要验证发件域名）
- 正式电子签名：尚未接入，需要选择 DocuSign 或 Adobe Acrobat Sign 等供应商并开发签署与回调验证

## 三、部署步骤

### 第 1 步：备份并使用预览环境
1. 先下载当前 Netlify 部署版本备份。
2. 先创建 staging / deploy preview，不要直接覆盖生产站。
3. 不要删除现有网站或数据库。

### 第 2 步：创建 Supabase 项目
1. 使用 XBIO 控制的组织账号创建 Supabase 项目。
2. 在 SQL Editor 执行 `supabase/schema.sql`。
3. 复制 Project URL 和 anon key。
4. 将 Service Role Key 仅放入 Netlify 的服务端环境变量，绝不可提交到 Git 或放入网页 JS。
5. 配置允许的登录回调 URL、邮箱验证和正式 SMTP。
6. 生产环境不要依赖 Supabase 默认测试邮件额度。

### 第 3 步：配置 Netlify 环境变量
在 Netlify → Site configuration → Environment variables 中配置：

- `SUPABASE_URL`
- `SUPABASE_ANON_KEY`
- `SUPABASE_SERVICE_ROLE_KEY`
- `ALLOWED_ORIGINS=https://xbihub.com,https://www.xbihub.com`
- `URL=https://xbihub.com`

可选邮件通知：
- `RESEND_API_KEY`
- `MAIL_FROM=XBIHUB <verified-sender@your-domain.com>`

不要将密钥写入 HTML、浏览器 JS 或任何公开仓库文件。`netlify.toml` 已指定发布目录和 Functions 目录。

### 第 4 步：部署到测试站
1. 将 `xbihub/` 目录内容提交到 XBIO 控制的代码仓库。
2. 在 Netlify 连接正确仓库；Publish directory 为 `.`，Functions directory 为 `netlify/functions`。
3. 先部署 staging / 分支预览。
4. 打开 `secure.html`，检查是否识别后端配置。不要先录入真实合作伙伴数据。

### 第 5 步：安全创建管理员
先通过注册并完成邮箱验证，创建由 XBIO 控制的管理员账号。然后在 Supabase SQL Editor 中执行，将邮箱替换为真实受控邮箱：

```sql
update public.profiles
set role = 'admin'
where email = 'your-controlled-admin@example.com';
```

确认仅更新一行。不要把管理员角色赋予尚未验证或不受 XBIO 控制的账号。

### 第 6 步：测试完整业务流程
- 合作伙伴注册、验证邮箱、登录。
- 提交节点申请，计划说明至少 50 个字符。
- 管理员审核、通过、要求补充信息、拒绝。
- 配置 Resend 后验证状态通知。
- 发布测试资料，确认不同访问级别。
- 验证没有已签协议时无法激活节点。
- 验证未确认付款或未完成培训时无法激活节点。
- 用第二个账号验证不能读取他人的申请。
- 验证普通合作伙伴无法调用管理员操作。
- 检查 Netlify Functions 日志与 Supabase 审计记录。

### 第 7 步：正式电子签名（仍需完成）
当前接口会拒绝正式签署请求，直到真正的供应商集成完成。正式上线前必须实现：
1. 根据批准的协议模板创建签署任务；
2. 验证签署人身份和签署授权；
3. 按供应商规则验证 Webhook 签名；
4. 通过供应商 API 复核签署完成状态，不能只相信浏览器回调；
5. 将签署 PDF、证书和审计轨迹归档到私有存储；
6. 仅在验证成功后设置 `provider_verified=true`；
7. 处理拒签、撤销、过期、重复回调和协议版本更新。

### 第 8 步：邮件
可选 Resend 集成仅发送申请状态通知。发件域名需先在 Resend 验证；注册验证邮件由 Supabase Auth SMTP 配置负责。

## 四、功能状态

| 功能 | 当前状态 |
|---|---|
| 原官网展示页面 | 保留 |
| 公开演示管理员登录 | 已从登录入口移除 |
| 演示数据自动注入 | 已禁用 |
| 服务端注册、登录、会话识别 | 已实现代码，需配置 Supabase |
| 申请提交、个人申请列表 | 已实现代码，需配置数据库 |
| 管理员审核 | 已实现代码，需手动设置管理员角色 |
| 申请状态邮件 | 可选 Resend 集成，需配置 |
| 资料列表和权限级别 | 已实现代码；机密文件仍需私有存储和短期签名链接 |
| 节点激活门槛 | 已实现；需真实已签协议记录及付款、培训确认 |
| 正式电子签名 API 和可信回调 | 尚未接入 |
| 私密文件上传和签署件归档 | 尚未实现 |
| 线上部署和生产端到端验收 | 尚未执行 |

## 五、生产环境剩余安全工作
- 为登录和申请接口增加限流及机器人防护。
- 为管理员启用 MFA，补齐账户恢复、会话续期和轮换。
- 机密资料改用私有存储和短期签名下载 URL。
- 完成跨境数据处理、隐私和保留期限审查。
- 增加邮件重试队列和投递日志。
- 接受真实合作协议或敏感资料前，进行独立安全审查。

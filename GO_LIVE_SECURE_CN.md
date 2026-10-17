# XBIHUB 真正登录 / 提交申请 — 上线清单

本流程启用 `secure.html` 的真实注册、登录与节点申请（Netlify Functions + Supabase）。

---

## 一、创建 Supabase 项目

1. 打开 https://supabase.com → New project  
2. 记录：
   - **Project URL**（形如 `https://xxxxx.supabase.co`）
   - **anon public** key  
   - **service_role** key（仅服务端，勿写入前端）

3. 进入 **SQL Editor** → 粘贴并执行本包内 `supabase/schema.sql`  
4. 确认表已创建：`profiles`、`applications`、`agreements`、`resources`、`audit_logs`

5. **Authentication → Providers**  
   - 启用 Email  
   - 开发阶段可暂时关闭 “Confirm email”（方便立即登录测试）  
   - 生产环境建议开启邮箱验证，并配置自定义 SMTP

6. **Authentication → URL Configuration**  
   - Site URL：`https://你的域名` 或 `https://xxx.netlify.app`  
   - Redirect URLs：同上 + `https://你的域名/secure.html`

---

## 二、部署到 Netlify

1. 上传 / 连接本站点根目录（含 `netlify.toml`、`netlify/functions/api.mjs`）  
2. **Site configuration → Environment variables** 添加：

| Key | Value |
|-----|--------|
| `SUPABASE_URL` | `https://xxxxx.supabase.co` |
| `SUPABASE_ANON_KEY` | anon public key |
| `SUPABASE_SERVICE_ROLE_KEY` | service_role key |
| `URL` | 正式站点 URL（可选） |

3. **Trigger deploy → Clear cache and deploy site**  
4. 打开 `https://你的站/secure.html`

---

## 三、验证真实流程

### 1. 健康检查
在浏览器控制台或使用：

```js
fetch('/.netlify/functions/api', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ action: 'health' })
}).then(r => r.json()).then(console.log)
```

期望：`{ ok: true, configured: true, ... }`

### 2. 注册
- 打开 `/secure.html` → Create partner account  
- 姓名 + 邮箱 + 密码（**至少 12 位**）  
- 若开启邮箱验证：先点邮件链接再登录  

### 3. 登录
- Sign in → 成功后看到 “My account” 与申请表单  

### 4. 提交申请
- 填写实体类型、公司名、申请人、国家/地区、节点类型  
- **Node plan 至少 50 字**  
- Submit → 获得编号如 `XBI-2026-XXXXXXXX`  
- 右侧 “My applications” 应出现记录  

### 5. 管理员（可选）
在 Supabase SQL 执行（换成你的邮箱）：

```sql
update public.profiles
set role = 'admin'
where email = 'your-admin@example.com';
```

重新登录后可见 Admin console，可审核 / 批准申请。

---

## 四、安全说明

- Service Role Key **只能**放在 Netlify 环境变量，不能进 Git 或前端  
- 会话使用 **HttpOnly + Secure + SameSite=Lax** Cookie  
- 数据库对 `anon` / `authenticated` 无直接表权限，仅 Functions 用 service role 访问  
- 电子签与付款激活需另行对接供应商；当前会返回明确 503/409 提示  

---

## 五、故障排查

| 现象 | 原因与处理 |
|------|------------|
| `Server is not configured` | 环境变量未设或未重新部署 |
| `Sign-in failed` | 密码错误，或需先完成邮箱验证 |
| 注册成功但无法登录 | Supabase 开启了 Confirm email |
| 提交申请 401 | Cookie 未写入：确认站点是 HTTPS；勿用错误域名打开 |
| 提交 500 | Functions 日志查看；常见为 schema 未执行 |
| 无 Admin | 未执行 `update profiles set role='admin'` |

查看日志：Netlify → **Functions** → `api` → 实时日志。

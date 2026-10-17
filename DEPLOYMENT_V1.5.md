# XBIHUB Deployment Guide — V1.5 Production

## Status
This is the elevated production candidate after content enrichment, schema fix, mobile navigation, and visual consistency pass.

## Pre-flight (must be done by operator)

### 1. Supabase
- Create project under XBIO-controlled organization.
- SQL Editor → paste and run the full content of `supabase/schema.sql` (already fixed).
- Authentication → URL Configuration: add production and staging domains.
- Enable email confirmation.
- Configure custom SMTP for production volume.
- After first verified admin registers, run:
```sql
update public.profiles set role = 'admin' where email = 'your-controlled-admin@xbio...';
```

### 2. Netlify Environment Variables
Site configuration → Environment variables:
- `SUPABASE_URL`
- `SUPABASE_ANON_KEY`
- `SUPABASE_SERVICE_ROLE_KEY` (server only)
- `ALLOWED_ORIGINS=https://xbihub.com,https://www.xbihub.com` (adjust to real domain)
- `URL=https://xbihub.com`

Optional:
- `RESEND_API_KEY`
- `MAIL_FROM=XBIHUB <verified@domain>`

### 3. Deploy
- Connect repository or drag-drop the `xbihub/` folder.
- Publish directory: `.`
- Functions directory: `netlify/functions`
- Deploy first to a staging site / deploy preview.
- Test full flow: register → email verify → login → submit application → admin review → permission isolation.
- Confirm e-signature and activation remain fail-closed until provider integration is complete.

### 4. Go-live
Only after staging checklist in `PRODUCTION_GO_LIVE_CHECKLIST_CN.md` is fully checked.

## Version
V1.3 — content, mobile nav, schema fix, consistency.

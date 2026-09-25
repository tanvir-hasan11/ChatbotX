# 🚀 ORIN — FREE Deployment Guide (Render.com)

## কেন Render (Free tier)?

| রিসোর্স | Render Free | কোথায় পাবেন |
|---------|-------------|------------|
| Web Service (Next.js builder) | ✅ 750 hrs/মাস free | Render Web Service |
| PostgreSQL | ✅ Free (14 days, তারপর $6) **বা** Supabase টা Permanent Free | Supabase / Neon |
| Redis | ✅ Upstash টা Permanent Free | Upstash |
| WebSocket (Realtime) | ✅ PartyKit free tier | PartyKit Cloud |
| Worker | ⚠️ Free tier এ একটাই service — worker + builder একসাথে চালাতে হবে | Render |
| S3 Storage | ✅ Cloudflare R2 free (10GB) বা Backblaze B2 (10GB free) | Cloudflare R2 |

**Total Monthly Cost: 0 টাকা** (যতক্ষণ না DB টা 14 দিন পরে expire হয় — তাই আমরা Supabase/Neon ব্যবহার করবো)

## ধাপ ১: Free Services Account খুলুন

### 1a. Database → **Neon.tech** (Recommended, Permanent Free)
1. https://neon.tech এ signup করুন (GitHub দিয়ে)
2. Project বানান → ORIN নামে
3. Connection string কপি করুন → `DATABASE_URL`
   - ফরম্যাট: `postgresql://user:pass@ep-xxx.aws.neon.tech/orin?sslmode=require`

### 1b. Redis → **Upstash** (Permanent Free)
1. https://upstash.com signup
2. Database বানান → ORIN-Redis
3. Region: Singapore (বাংলাদেশের কাছে)
4. `REDIS_URL` কপি করুন → `rediss://default:xxx@xxx.upstash.io:6379`

### 1c. Storage → **Cloudflare R2** (10GB Free)
1. https://dash.cloudflare.com → R2 → Create Bucket
2. Bucket name: `orin`
3. API token বানান → Access Key ID + Secret পাবেন
4. S3 config এ বসাবেন (S3_ENDPOINT = `https://<account_id>.r2.cloudflarestorage.com`)

### 1d. Email → cPanel SMTP ফ্রি
আপনার cPanel এ email account বানান: `no-reply@stratmarkbd.com`
- SMTP Server: `mail.stratmarkbd.com`
- Port: `587` (TLS) বা `465` (SSL)
- SMTP_SERVER=`smtp://no-reply@stratmarkbd.com:PASSWORD@mail.stratmarkbd.com:587`

## ধাপ ২: Render.com এ ORIN Deploy

1. https://render.com → GitHub দিয়ে login
2. **New → Web Service** → `tanvir-hasan11/ChatbotX` repo সিলেক্ট করুন
3. **Branch:** `rebrand/orin`
4. **Root Directory:** `apps/builder`
5. **Build Command:**
   ```
   cd ../.. && corepack enable && pnpm install && pnpm build --filter=builder
   ```
6. **Start Command:**
   ```
   pnpm --filter builder start
   ```
7. **Instance Type:** Free
8. **Environment Variables** যোগ করুন (নিচে list আছে)
9. Deploy চাপুন 🚀

## ধাপ ৩: Environment Variables (Render Dashboard)

```
NEXT_PUBLIC_EDITION=community
PLATFORM_ADMIN_EMAIL=your@email.com
BETTER_AUTH_SECRET=<openssl rand -base64 32 দিয়ে generate করুন>
BETTER_AUTH_URL=https://orin.stratmarkbd.com
ENCRYPTION_KEY=<openssl rand -hex 32>
DATABASE_URL=<Neon থেকে>
REDIS_URL=<Upstash থেকে>
S3_ACCESS_KEY_ID=<R2 Access Key>
S3_SECRET_ACCESS_KEY=<R2 Secret>
S3_BUCKET=orin
S3_REGION=auto
S3_ENDPOINT=https://<account_id>.r2.cloudflarestorage.com
SMTP_SERVER=smtp://no-reply@stratmarkbd.com:PASS@mail.stratmarkbd.com:587
SMTP_FROM=no-reply@stratmarkbd.com
REALTIME_BROADCAST_SECRET=<random string>
JAVASCRIPT_EXECUTOR_URL=http://localhost:3210
JAVASCRIPT_EXECUTOR_TOKEN=<random 32+ chars>
NEXT_PUBLIC_BUILDER_URL=https://orin.stratmarkbd.com
FORCE_PUBLIC_HTTPS=true
LOG_LEVEL=info
```

## ধাপ ৪: cPanel DNS (Zone Editor)

Render আপনাকে একটা URL দেবে যেমন `orin-xxxx.onrender.com`

cPanel → **Zone Editor** → stratmarkbd.com →

| Type | Name | Value |
|------|------|-------|
| CNAME | orin | orin-xxxx.onrender.com |

Render → Settings → Custom Domain → `orin.stratmarkbd.com` যোগ করুন → Free SSL auto হবে ✅

## ধাপ ৫: Worker + Realtime

Worker গুলো background এ চলবে। Render এ আলাদা **Background Worker** service বানাতে হবে (Paid $7/mo) — অথবা Free এ থাকতে হলে:

**Free Option:** প্রথমে শুধু builder deploy করুন। Worker ছাড়া chatbot কাজ করবে, কিন্তু automation/sequences চলবে না।

পরে দরকার হলে ২টা অপশন:
- **Render Background Worker** ($7/mo) — worker + realtime একসাথে
- **Railway** ($5 credit free/mo) — worker আলাদা চালাতে পারবেন

## ⚠️ জরুরি নোট

1. **Free tier এ Render এর service 15 মিনিট inactive হলে sleep করে।** কেউ ওয়েবসাইটে ঢুকলে ৫০ সেকেন্ড cold start হয়।
   - Fix: https://cron-job.org (free) দিয়ে প্রতি ৫ মিনিটে `https://orin.stratmarkbd.com/api/health` ping করুন — sleep হবে না
2. **Neon free tier** 5GB storage, 190 compute hours — ছোট business এর জন্য যথেষ্ট
3. **Upstash free** 10,000 commands/day — যথেষ্ট
4. **R2 free** 10GB storage + free egress — যথেষ্ট

## 📊 Cost Summary

| Service | Free Limit | পরে দরকার হলে |
|--------|-----------|---------------|
| Render | 750 hrs | $7/mo (no sleep) |
| Neon DB | 5GB / 190 hrs | $19/mo |
| Upstash | 10K cmd/day | $10/mo |
| R2 | 10GB | pay-per-GB |
| cron-job.org | Unlimited | Free |
| **Total** | **$0/mo** | |

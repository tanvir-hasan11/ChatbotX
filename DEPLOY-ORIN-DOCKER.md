# 🐳 ORIN — Deploy on Render free tier using a pre-built Docker image

Render free tier এ এই monorepo build করা যায় না (Next.js build এ 8GB+ RAM লাগে, free tier এ 512MB)।
সমাধান: **GitHub Actions** এ image build করে **GHCR** এ push করা (GitHub Actions runner এ 16GB RAM free),
তারপর Render শুধু ওই ready-made image চালায় — build RAM এর প্রয়োজনই হয় না।

## Image

```
ghcr.io/tanvir-hasan11/orin-builder:main
```

`.github/workflows/orin-docker.yml` প্রতিবার `main` এ push হলে এই image টা automatically rebuild করে।

## Render Setup (একবারই)

1. Render Dashboard → **New +** → **Web Service**
2. **Deploy an existing image** সিলেক্ট করুন (repo না!)
3. **Image URL:** `ghcr.io/tanvir-hasan11/orin-builder:main`
4. **Name:** `orin` | **Region:** Singapore | **Instance Type:** Free
5. **Note:** image টা public হলে credentials লাগবে না; private হলে GHCR read token লাগবে

## Environment Variables (Render → Environment)

আগের মতোই একই env vars:

```
NEXT_PUBLIC_EDITION=community
PLATFORM_ADMIN_EMAIL=<your email>
BETTER_AUTH_SECRET=<generated>
BETTER_AUTH_URL=https://orin.onrender.com
ENCRYPTION_KEY=<64-char hex>
DATABASE_URL=<Neon production branch>
REDIS_URL=<Upstash>
S3_ACCESS_KEY_ID=<R2>
S3_SECRET_ACCESS_KEY=<R2>
S3_BUCKET=orin
S3_REGION=auto
S3_ENDPOINT=https://<account_id>.r2.cloudflarestorage.com
SMTP_SERVER=smtp://user:pass@mail.stratmarkbd.com:587
SMTP_FROM=no-reply@stratmarkbd.com
REALTIME_BROADCAST_SECRET=<generated>
JAVASCRIPT_EXECUTOR_URL=http://localhost:3210
JAVASCRIPT_EXECUTOR_TOKEN=<generated>
NEXT_PUBLIC_BUILDER_URL=https://orin.onrender.com
FORCE_PUBLIC_HTTPS=true
RUN_DB_MIGRATE=true
RUN_DB_SEED=true
LOG_LEVEL=info
```

⚠️ `RUN_DB_MIGRATE=true` + `RUN_DB_SEED=true` — প্রথম deploy এ একবার দিন, database তৈরি হয়ে গেলে
`RUN_DB_SEED` বাদ দিতে পারেন (migrate রাখতে পারেন)।

## Code বদলালে নতুন version deploy করা

1. `main` এ push করুন → GitHub Actions এ `orin-docker.yml` run হবে (~15–20 min)
2. Actions: https://github.com/tanvir-hasan11/ChatbotX/actions
3. Image নতুন হলে Render → **Manual Deploy** → **Deploy latest reference**

## GHCR Image Public করা (প্রথমবার)

Image টা default ভাবে private থাকতে পারে। Public করতে:

1. https://github.com/users/tanvir-hasan11/packages/container/orin-builder/settings
2. **Danger Zone** → **Change visibility** → **Public**

Public করলে Render কোনো token ছাড়াই image pull করতে পারবে।

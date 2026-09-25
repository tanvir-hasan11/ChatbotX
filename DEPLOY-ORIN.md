# ORIN — Production Deployment Guide (cPanel Shared Hosting)

## ⚠️ Important: cPanel shared hosting এ Docker চলবে না

ORIN (ChatbotX fork) হলো একটা বড় Next.js + NestJS style monorepo যার দরকার:
- Node.js >= 24
- PostgreSQL (TimescaleDB)
- Redis
- RustFS (S3-compatible storage)
- Worker processes, realtime websocket server, JavaScript executor

cPanel shared hosting এ **সাধারণত** এসব চালানো যায় না — কারণ:
- Root/SSH access নেই, Docker নেই
- সাধারণত শুধু PHP + MySQL support করে
- Long-running Node.js process, port binding, websocket সাপোর্ট করে না

## 🎯 আপনার আসল অপশন

### অপশন ১: শুধু domain point করা (সবচেয়ে সহজ)
ORIN কে একটা VPS (DigitalOcean/Hetzner/Contabo - $5-10/mo) অথবা Render/Railway/Fly.io তে deploy করুন, তারপর cPanel-এ DNS থেকে:
- `orin.stratmarkbd.com` → A record দিয়ে VPS IP তে
- অথবা CNAME দিয়ে Render/Railway URL এ

cPanel → Zone Editor এ গিয়ে শুধু DNS record বসালেই হবে। Hosting ওখানে লাগবে না।

### অপশন ২: cPanel-এ শুধু static frontend, backend আলাদা — NOT RECOMMENDED
ORIN এর কোনো pure static build নেই, সব একসাথে চলে।

## 🚀 কীভাবে Deploy (VPS ধরে নিচ্ছি)

```bash
# 1. Clone
git clone -b rebrand/orin https://github.com/tanvir-hasan11/ChatbotX.git orin
cd orin

# 2. Env setup
cp .env.orin.production.example .env
nano .env   # secrets গুলো fill করুন

# 3. Docker compose (infra + app)
docker compose up -d postgres redis filesystem

# 4. Build & migrate
corepack enable
pnpm install
pnpm build

# 5. Start apps
docker compose -f apps/builder/docker/docker-compose.yml up -d
# অথবা pm2 দিয়ে
pnpm --filter builder start
```

## 🌐 DNS (cPanel Zone Editor)

```
Type: A
Name: orin
Value: <VPS-এর-IP>
TTL: 300
```

## 🔐 প্রথম যে জিনিসগুলো করবেন
- BETTER_AUTH_SECRET, ENCRYPTION_KEY generate করুন
- PLATFORM_ADMIN_EMAIL আপনার নিজের ইমেইল দিন
- HTTPS এর জন্য reverse proxy (Nginx/Caddy) + Let's Encrypt

## ✅ যা যা rebrand হয়েছে
- `site.webmanifest` → ORIN
- `packages/business/src/platform/settings.ts` → default name "ORIN", policy/terms URL → orin.stratmarkbd.com

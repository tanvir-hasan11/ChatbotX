# ORIN on Oracle Cloud Always Free — the single-VM path

This guide puts **the entire ORIN stack** (Postgres + pgvector, Redis, RustFS object
storage, JavaScript sandbox, the Next.js builder, all BullMQ workers, and the realtime
WebSocket server) onto **one free Oracle Cloud VM**, behind Caddy with automatic HTTPS.

No Neon. No Upstash. No Cloudflare R2. No GitHub Container Registry. No Render.
No build-RAM ceiling. No 15-minute sleep.

---

## 1. Why Oracle

| | Oracle Always Free | Render free |
|---|---|---|
| RAM | **24 GB** (ARM Ampere A1) | 512 MB |
| vCPU | 4 | shared |
| Disk | 200 GB | ephemeral |
| Sleeps when idle | **never** | after 15 min |
| Docker / root | **yes** | no |
| Cost | **₹0 forever** | ₹0 (with limits) |

That 24 GB is what makes the whole difference — the builder compiles locally, workers
run in the same box, and nothing has to be pre-built in CI.

## 2. Create the account + VM

1. Sign up at <https://cloud.oracle.com> → **Start for free**. A card is required for
   identity verification; the Always Free resources below are not charged. (If your
   first attempt is rejected, try again in 24 h with the same details — this is common.)
2. Region: pick the closest to Bangladesh that still shows **Always Free eligible** —
   Singapore or Mumbai are the usual winners.
3. **Compute → Instances → Create instance**
   - Image: **Ubuntu 24.04**
   - Shape: **VM.Standard.A1.Flex** → 4 OCPU / 24 GB (this is the free allowance)
   - SSH key: paste your public key (or let Oracle generate and download the private key)
   - Boot volume: 200 GB (default for A1 Flex)
4. Note the **public IP** once it's running.

## 3. Open the ports — two places, both are needed

Oracle blocks traffic by default in **two independent layers**. If you skip either one,
Caddy's certificate challenge will fail and you'll be stuck wondering why.

**a) VCN Security List**
Networking → Virtual Cloud Networks → your VCN → Security Lists → Default →
**Add Ingress Rules**:

| Source CIDR | Protocol | Dest port |
|---|---|---|
| `0.0.0.0/0` | TCP | `80` |
| `0.0.0.0/0` | TCP | `443` |

**b) Inside the VM (Oracle's Ubuntu images ship with an iptables `INPUT` policy of DROP)**

```bash
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 443 -j ACCEPT
sudo netfilter-persistent save
```

## 4. Install Docker

```bash
ssh ubuntu@<PUBLIC_IP>

sudo apt-get update
sudo apt-get install -y ca-certificates curl git
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo tee /etc/apt/keyrings/docker.asc >/dev/null
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list >/dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
docker version   # sanity check
```

> Building the Next.js app inside the VM needs swap headroom on the smallest shapes.
> With 24 GB you don't need this, but if you ever shrink the VM:
> `sudo fallocate -l 4G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile`

## 5. Get the code

```bash
cd ~
git clone https://github.com/tanvir-hasan11/ChatbotX.git orin
cd orin
git checkout main
```

## 6. Configure

```bash
cp .env.orin.oracle.example .env
nano .env
```

Fill every `CHANGE_ME`. Generate the secrets right there:

```bash
openssl rand -base64 32   # → BETTER_AUTH_SECRET
openssl rand -hex 32      # → ENCRYPTION_KEY   (must be exactly 64 chars)
openssl rand -hex 24      # → JAVASCRIPT_EXECUTOR_TOKEN
openssl rand -base64 24   # → REALTIME_BROADCAST_SECRET
openssl rand -hex 16      # → POSTGRES_PASSWORD and S3_SECRET_ACCESS_KEY
```

**Critical:** `POSTGRES_PASSWORD` must be the exact same value as the password inside
`DATABASE_URL` (they're two separate lines in the template — keep them in sync).

Also set `ORIN_DOMAIN` to your real hostname and update the three URLs
(`BETTER_AUTH_URL`, `NEXT_PUBLIC_BUILDER_URL`, `NEXT_PUBLIC_STORAGE_URL`) to match.

## 7. DNS before you start

Caddy needs the domain to already point here, otherwise the ACME challenge has nothing
to validate against. In cPanel → **Zone Editor** for `stratmarkbd.com`:

```
Type: A     Name: orin     Value: <PUBLIC_IP>     TTL: 300
```

Verify from the VM:

```bash
getent hosts orin.stratmarkbd.com   # must print the VM's public IP
```

## 8. Launch

First build compiles the whole monorepo locally — expect **20–35 minutes** on 4 vCPU.
This is the part that Render could never do; here it just runs.

```bash
docker compose -f docker-compose.orin.yml up -d --build
```

Watch it come up:

```bash
docker compose -f docker-compose.orin.yml ps
docker compose -f docker-compose.orin.yml logs -f builder
```

Wait for `builder` to report healthy and Caddy to log `certificate obtained successfully`.
Then open **https://orin.stratmarkbd.com** and sign up with the email you put in
`PLATFORM_ADMIN_EMAIL` — that account gets access to `/manage`.

Once you're in and everything looks right, you can drop the seeder so restarts don't
re-run it:

```bash
sed -i 's/^RUN_DB_SEED=.*/RUN_DB_SEED="false"/' .env   # or edit the compose env block
docker compose -f docker-compose.orin.yml up -d builder
```

## 9. Day-to-day

```bash
cd ~/orin

# update to the latest fork state
git pull
docker compose -f docker-compose.orin.yml up -d --build

# status / logs
docker compose -f docker-compose.orin.yml ps
docker compose -f docker-compose.orin.yml logs -f worker

# restart one service
docker compose -f docker-compose.orin.yml restart builder

# back up the database (runs pg_dump inside the container)
docker compose -f docker-compose.orin.yml exec postgres \
  pg_dump -U orin orin | gzip > ~/orin-$(date +%F).sql.gz
```

## 10. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Caddy logs `no such host` / certificate never arrives | DNS not propagated, or port 80 still blocked | `getent hosts $ORIN_DOMAIN` and re-check both port layers from step 3 |
| `builder` exits with exit 137 | OOM (only on the 1 GB AMD free shape) | Switch to the A1.Flex shape, or add swap |
| Images/attachments 404 | Storage URL mismatch | Confirm `NEXT_PUBLIC_STORAGE_URL=https://<domain>/storage/orin/` matches `S3_BUCKET` |
| Login loops back to `/auth` | `BETTER_AUTH_URL` doesn't match the address in the browser bar | Fix the env var and `docker compose ... up -d builder` |
| `javascript-executor` unhealthy | `JAVASCRIPT_EXECUTOR_TOKEN` shorter than 32 chars | Regenerate and restart |

---

**What this replaces:** the earlier Render + Neon + Upstash + R2 + GHCR plan. None of
those are needed anymore — everything is on the one VM. `DEPLOY-ORIN-FREE.md` is kept
only as a historical reference.
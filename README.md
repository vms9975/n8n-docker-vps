# self host n8n: a production-ready Docker setup with Postgres and Caddy, plus the right VPS plan from $10.99 a year

To self host n8n you need three things: a small Linux VPS, Docker, and a domain name. The software itself costs nothing — n8n's community edition can run on your own infrastructure for free under its Sustainable Use License, and on a budget provider like RackNerd the server can run you as little as $10.99 for an entire year.

The part most tutorials skip is picking the server. n8n stores your workflows and credentials on disk, generates an encryption key on first launch, and quietly falls back to a single SQLite file if you don't give it a real database. A careless `docker run` works great until the first restart takes your setup with it. This guide covers both halves: a Docker Compose stack (n8n + PostgreSQL + Caddy) that survives reboots, and a mapping of n8n's actual memory needs onto RackNerd's plan lineup so you neither overpay nor undersize.

> **In a hurry?** The **2 GB KVM plan at $18.88/year** is the sweet spot: enough RAM for n8n and PostgreSQL on the same box, 40 GB SSD, 4.5 TB of monthly transfer. Light setups with a few workflows run on the 1 GB plan at $10.99/year, but expect swapping under load. 👉 [Check RackNerd's current promo plans and prices](https://bit.ly/my-rn)

## What self-hosting n8n actually costs

The community edition is free to self-host for internal business use, and for most individuals and small teams it's complete enough to run in production. What sits behind paid licenses are the enterprise features (SSO, advanced permissions, environments, log streaming) and n8n Cloud, the hosted subscription where n8n runs and maintains the instance for you.

Cloud hosting is convenient, and it bills you for the privilege on a per-execution model:

| Setup | Price | Execution limit |
| --- | --- | --- |
| n8n Cloud Starter | €20/month | 2,500 executions/month |
| n8n Cloud Pro | €50/month | 10,000 executions/month |
| Community edition on a RackNerd 2 GB VPS | $18.88/year (~$1.57/month) | None — bounded by your server |

One month of Starter costs more than a full year of the 2 GB VPS. If your automations fire regularly (webhooks, scheduled syncs, monitoring), executions pile up faster than you'd expect, and the per-execution model caps you monthly no matter what.

The usual alternatives — Hetzner is a common shortlist pick in Europe, DigitalOcean the friendlier option if you're new to servers — all run n8n fine too. You just pay noticeably more per month for the same RAM. RackNerd competes on one axis: annual price. Whether that trade makes sense depends on how much you value a polished dashboard over roughly $1.57 a month. If the answer is "not much," 👉 [browse RackNerd's full VPS lineup](https://bit.ly/my-rn) and pick a tier from the table below.

## Which RackNerd plan actually fits n8n

For most readers, the 2 GB plan — and the reasoning comes straight from n8n's documented requirements. Production guides converge on the same numbers: 2 GB of RAM runs n8n and PostgreSQL comfortably on one machine, 1 GB works for light use but starts swapping once executions queue up, and n8n's own Docker Compose documentation treats 2 vCPUs and 4 GB as the sensible floor for a serious deployment. Map that onto RackNerd's promo lineup and the tiers mostly pick themselves.

RackNerd's recurring promotional lineup (KVM virtualization, 1 Gbps port, annual billing):

| Plan | RAM / vCPU | SSD | Monthly transfer | Price | Order |
| --- | --- | --- | --- | --- | --- |
| 1 GB KVM | 1 GB / 1 vCPU | 20 GB | 1.5 TB | $10.99/yr | [ Order the 1 GB plan](https://bit.ly/my-rn) |
| 2 GB KVM | 2 GB / 2 vCPU | 40 GB | 4.5 TB | $18.88/yr | [ Order the 2 GB plan](https://bit.ly/my-rn) |
| 2.5 GB KVM | 2.5 GB / 3 vCPU | 55 GB | 6 TB | $25.88/yr | [ Order the 2.5 GB plan](https://bit.ly/my-rn) |
| 3.5 GB KVM | 3.5 GB / 3 vCPU | 75 GB | 8 TB | $38.99/yr | [ Order the 3.5 GB plan](https://bit.ly/my-rn) |
| 5 GB KVM | 5 GB / 4 vCPU | 100 GB | 10 TB | $58.88/yr | [ Order the 5 GB plan](https://bit.ly/my-rn) |
| 6.5 GB KVM | 6.5 GB / 5 vCPU | 130 GB | 13 TB | $68.88/yr | [ Order the 6.5 GB plan](https://bit.ly/my-rn) |

One honest caveat: the exact SSD and transfer numbers rotate slightly between promo batches. The RAM-and-price pairs above are the recurring headline tiers, and the order form shows the precise allocation for whichever batch is live when you order. These prices are also the renewal prices, so year two doesn't sneak up on you — a rarity among VPS deals.

Decision rules, by situation:

- **Just trying n8n out** — a few personal workflows, a handful of runs per day → the 1 GB plan at $10.99/year. Add a swap file and accept some sluggishness under load.
- **The default choice** — n8n plus PostgreSQL on the same box, credentials you actually care about → the 2 GB plan at $18.88/year. The rest of this guide assumes this tier. 👉 [Order the 2 GB plan for $18.88/year](https://bit.ly/my-rn)
- **Heavy use** — long execution history, large data transformations, or you want headroom toward n8n's 2 vCPU / 4 GB guidance → 3.5 GB ($38.99/year) or 5 GB ($58.88/year).
- **Scaling out later** — if you split into queue mode with separate workers, plan for the 5 GB tier or a second small server.

## The deployment: n8n + PostgreSQL + Caddy in Docker Compose

Three services in one Compose file, about 30–40 minutes on a fresh server. Why this shape: PostgreSQL replaces the default SQLite (a single file that handles concurrent writers poorly and makes reliable backups painful — Postgres gives you proper concurrent access and point-in-time recovery), and Caddy sits in front on ports 80/443, fetching and renewing a Let's Encrypt certificate automatically with zero manual steps.

### 1. Prep the server

Ubuntu 24.04 is a safe pick in the OS selector at checkout. SSH in and install Docker:

bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker
docker --version


The Compose plugin ships with modern Docker, so `docker compose version` should just work.

### 2. Create the project folder and secrets

bash
mkdir -p ~/n8n && cd ~/n8n
{
  echo "N8N_ENCRYPTION_KEY=$(openssl rand -hex 32)"
  echo "POSTGRES_PASSWORD=$(openssl rand -hex 24)"
  echo "POSTGRES_USER=n8n"
  echo "POSTGRES_DB=n8n"
  echo "N8N_HOST=n8n.yourdomain.com"
  echo "GENERIC_TIMEZONE=UTC"
} > .env
chmod 600 .env


The encryption key deserves its own warning. **n8n encrypts every credential you save with `N8N_ENCRYPTION_KEY`.** If the key changes between redeploys, all stored credentials become unreadable and you re-enter everything by hand. That's exactly why it lives in `.env` instead of being auto-generated inside the container — back this file up somewhere off the server before you do anything else.

A note on outdated tutorials: older guides tell you to set `N8N_BASIC_AUTH_ACTIVE`, `N8N_BASIC_AUTH_USER`, and `N8N_BASIC_AUTH_PASSWORD`. Those variables disappeared when n8n 1.0 replaced basic auth with built-in user management. Setting them today does nothing; you create your owner account in the browser on first launch.

### 3. The Compose file

yaml
services:
  postgres:
    image: postgres:16
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  n8n:
    image: n8nio/n8n:latest
    restart: unless-stopped
    environment:
      - N8N_HOST=${N8N_HOST}
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://${N8N_HOST}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - TZ=${GENERIC_TIMEZONE}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_PROXY_HOPS=1
      - N8N_RUNNERS_ENABLED=true
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      postgres:
        condition: service_healthy

  caddy:
    image: caddy:2
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - n8n

volumes:
  postgres_data:
  n8n_data:
  caddy_data:
  caddy_config:


Notice what's missing: n8n's port 5678 is never published to the host. Caddy reaches it over Docker's internal network, so the editor never faces the internet directly. This is the detail naive `docker run -p 5678:5678` setups get wrong, and it's the fastest way to hand your automation panel to a port scanner.

### 4. The Caddyfile


n8n.yourdomain.com {
    reverse_proxy n8n:5678
}


Three lines, and that's the entire HTTPS setup. Once your domain resolves and ports 80/443 are reachable, Caddy requests the certificate on first visit and renews it on its own — no certbot, no cron jobs. Use a subdomain you actually own (an A record pointing at your VPS IP); reserved documentation domains like `example.com` will never issue a certificate. Because `N8N_HOST` and `WEBHOOK_URL` in `.env` match this domain, webhook triggers register with the correct public URL from day one.

### 5. Launch, claim the owner account, lock the firewall

bash
docker compose up -d
docker compose logs -f n8n


Postgres starts first, passes its healthcheck, then n8n connects and runs its database migrations — watching the logs confirm it's actually talking to Postgres rather than silently falling back to SQLite. If you want to verify directly, temporarily bind `127.0.0.1:5678:5678` on the n8n service and hit `http://localhost:5678/healthz`; it should return `{"status":"ok"}`. Remove the binding once Caddy is serving.

Open `https://n8n.yourdomain.com` and create the owner account immediately, before anyone else finds the URL. Then close everything except SSH and web traffic:

bash
sudo ufw allow OpenSSH
sudo ufw allow 80,443/tcp
sudo ufw enable


Total exposure: 80/443 for Caddy, SSH for you. Postgres and 5678 stay internal to Docker.

## Backups and updates

Two artifacts have to survive a dead server: the Postgres dump and the encryption key. Everything else is reproducible.

Save this as `backup.sh`:

bash
#!/usr/bin/env bash
set -euo pipefail
cd ~/n8n
docker compose exec -T postgres pg_dump -U n8n n8n | gzip > "n8n-db-$(date +%F).sql.gz"


Run it on a cron, and copy both the dump and your `.env` off the server. Do one test restore before you trust the routine — an untested backup is just a hope.

For updates, pin a specific image tag in the Compose file instead of tracking `:latest` blindly (n8n's GitHub releases page shows the current version number, and tested guides pin things like `n8nio/n8n:2.36.8`). When you deliberately upgrade, it's `docker compose pull && docker compose up -d`. Your data lives in named volumes, so removing containers never touches workflows or credentials — though running a backup first costs you ten seconds and is worth the habit.

## RackNerd specifics worth knowing before you order

None of the tutorial articles cover the buying side, so here's what actually affects the decision:

- **The headline prices are annual.** You pay for twelve months up front. Shorter billing exists in their catalog at a much higher effective rate; the promo lineup is where the value lives.
- **Renewals don't jump.** These promo plans renew at the same rate you paid, so the $18.88 is a recurring figure rather than an intro gimmick.
- **You choose the datacenter at checkout.** Los Angeles, San Jose, Seattle, Dallas, Chicago, New York, Atlanta, and Amsterdam are the usual options. For n8n, latency to the APIs your workflows call matters more than latency to your own screen.
- **It's real KVM**, so Docker, swap files, and custom kernels all behave natively. No OpenVZ workarounds.
- **New orders carry a 7-day money-back guarantee.** If the box turns out undersized for your workflows, the experiment costs you nothing but time.
- **Checkout takes PayPal, major cards, or crypto.**

## FAQ

**Can I run n8n on the $10.99 1 GB plan?**
Yes, for light use — a few personal workflows that run a handful of times per day. It will swap under load, and Postgres makes that worse. For $8 more per year, the 2 GB plan removes that ceiling entirely.

**Is the free community edition enough, or do I need to pay n8n?**
For internal automations, the community edition is complete. The paid layer covers SSO, advanced permissions, environments, log streaming, and the hosted Cloud option — things most solo users and small teams never touch.

**What happens if I lose the encryption key?**
Every saved credential becomes unreadable and you re-enter all of them by hand. Keep `.env` backed up off the server, next to your database dumps.

**Do I need a domain name?**
Yes. The HTTPS certificate and your webhook URLs both need a real hostname. Any registrar works; add an A record like `n8n.yourdomain.com` pointing at your VPS IP before launching the stack.

## Pick your setup and go

Testing n8n for the first time: the 1 GB plan at $10.99/year gets you there, with a swap file and modest expectations. Real workflows with credentials attached: the 2 GB plan at $18.88/year with the Compose stack above — it fits most readers and pays for itself within roughly a month compared to n8n Cloud Starter. Heavy volume or queue-mode ambitions: 3.5 GB or 5 GB, depending on how much execution history you keep.

Before you click order, two last checks: confirm the current batch's exact SSD and transfer allocation on the order form, and have your subdomain's A record ready so Caddy can issue the certificate the moment the stack comes up.

👉 [Set up your n8n VPS on RackNerd](https://bit.ly/my-rn)

# Clean VoIP Billing

Clean FastAPI + SQLite prepaid billing dashboard for a new VoIP switch instance.

This repository is a code template only. It intentionally does not include:

- production database files
- clients, balances, routes, CDR, SIP hits, or pcap records
- `.env` files
- API keys, admin passwords, Telegram tokens, or OpenAI keys
- FreeSWITCH runtime logs or backups

## What is included

- billing dashboard and REST API
- client/originator management
- terminator groups and routing
- prepaid balance checks and reservations
- credit limits
- CDR finalization
- SIP hit diagnostics
- optional passive SIP packet collector
- Telegram admin bot and client alert bot support

## Required environment

Create your own `.env` from `.env.example`.

Minimum billing variables:

```env
ADMIN_USER=admin
ADMIN_PASSWORD=change-this-long-password
API_SECRET_KEY=change-this-long-random-secret
BILLING_DB_PATH=/opt/voip-billing/data/billing.db
PUBLIC_SIP_HOST=YOUR_SERVER_PUBLIC_IP
```

Optional Telegram bot variables:

```env
TELEGRAM_BOT_TOKEN=
TELEGRAM_ALLOWED_CHAT_IDS=
TELEGRAM_WEBHOOK_SECRET=
BILLING_API_BASE_URL=http://127.0.0.1:8080
BILLING_API_SECRET_KEY=${API_SECRET_KEY}
```

Optional AI diagnostics:

```env
OPENAI_API_KEY=
OPENAI_MODEL=gpt-4.1-mini
```

The app fails closed when credentials are missing:

- `/` and dashboard write endpoints require `ADMIN_USER` / `ADMIN_PASSWORD`
- `/api/reserve` and `/api/finalize` require `API_SECRET_KEY`
- `/docs`, `/redoc`, and `/openapi.json` are disabled

## Run locally

```bash
python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
python railway_start.py
```

Open:

```text
http://127.0.0.1:8080
```

## Ubuntu VPS

Use `DEPLOY_UBUNTU.md` for a clean server install plan.

Recommended test VPS:

- Ubuntu 22.04 or 24.04
- 2 vCPU
- 4 GB RAM
- public IPv4

Recommended live VPS:

- 4 vCPU
- 8 GB RAM
- public IPv4
- opened SIP/RTP ports

## FreeSWITCH

The FreeSWITCH examples live in `freeswitch/`.

Before using them on a real server:

1. set `BILLING_API` to your billing API URL
2. put the same `API_SECRET_KEY` into `/etc/freeswitch/billing_api_key`
3. add your real client/provider IPs in the dashboard
4. update FreeSWITCH ACLs for your network policy

The clean template uses placeholders and localhost defaults. It does not contain old provider IPs.

## Service API authentication

FreeSWITCH or another trusted integration must send one of these headers:

```http
Authorization: Bearer <API_SECRET_KEY>
```

or:

```http
X-API-Key: <API_SECRET_KEY>
```

`/healthz` is intentionally public so a process manager or load balancer can check the app.

# Ubuntu VPS Deploy Notes

This is a clean deploy checklist for a new company/server.

## 1. Server

Use Ubuntu 22.04 or 24.04 with a public IPv4.

Open at minimum:

```text
22/tcp      SSH
8080/tcp    billing web/API, or proxy it behind nginx
5060/udp    SIP
5080/udp    SIP, if you use an external profile
10000-20000/udp RTP, adjust to your FreeSWITCH config
```

## 2. Install packages

```bash
apt update
apt install -y python3 python3-venv python3-pip curl git sqlite3 nginx
```

Install FreeSWITCH separately using the package source you trust for your Ubuntu version.

## 3. App install

```bash
mkdir -p /opt/voip-billing
cd /opt/voip-billing
git clone https://github.com/bergmandenis753-oss/clean-voip-billing.git app
cd app
python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
```

Edit `.env`:

```bash
nano .env
```

Set real values for:

```text
ADMIN_USER
ADMIN_PASSWORD
API_SECRET_KEY
BILLING_DB_PATH
PUBLIC_SIP_HOST
PUBLIC_BASE_URL
```

Use a new random secret for `API_SECRET_KEY`.

## 4. Billing systemd service

Create `/etc/systemd/system/voip-billing.service`:

```ini
[Unit]
Description=VoIP billing dashboard and API
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/opt/voip-billing/app
EnvironmentFile=/opt/voip-billing/app/.env
ExecStart=/opt/voip-billing/app/.venv/bin/python railway_start.py
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Enable it:

```bash
systemctl daemon-reload
systemctl enable --now voip-billing
systemctl status voip-billing --no-pager
```

Check:

```bash
curl http://127.0.0.1:8080/healthz
```

## 5. FreeSWITCH billing API key

Put the same `API_SECRET_KEY` into FreeSWITCH key file:

```bash
install -m 0600 -o freeswitch -g freeswitch /dev/null /etc/freeswitch/billing_api_key
nano /etc/freeswitch/billing_api_key
```

Copy scripts:

```bash
cp freeswitch/billing.lua /etc/freeswitch/scripts/billing.lua
cp freeswitch/billing_mark_answer.lua /etc/freeswitch/scripts/billing_mark_answer.lua
chown freeswitch:freeswitch /etc/freeswitch/scripts/billing*.lua
```

If billing runs on the same VPS, `freeswitch/billing.lua` defaults to:

```text
http://127.0.0.1:8080
```

For a separate billing API URL, set `BILLING_API` in the Lua script or service environment.

## 6. Dialplan

Copy:

```bash
cp freeswitch/dialplan/00_billing_clients.xml /etc/freeswitch/dialplan/00_billing_clients.xml
cp freeswitch/dialplan/default/00_billing_clients.xml /etc/freeswitch/dialplan/default/00_billing_clients.xml
```

If you use the FreeSWITCH default context for authenticated SIP users, run:

```bash
bash freeswitch/install_default_context_billing_include.sh
```

Reload FreeSWITCH:

```bash
fs_cli -x 'reloadxml'
```

## 7. First setup in dashboard

Open:

```text
http://YOUR_SERVER_PUBLIC_IP:8080
```

Create fresh data manually:

1. terminator group/provider
2. terminators/routes
3. originators/clients
4. client rates
5. balances or credit limits

No old clients, routes, balances, or CDR are included in this repository.

## 8. Telegram bot

For Telegram webhook you normally need HTTPS. For IP-only testing, use long polling or add HTTPS later with a domain/reverse proxy.

The billing and Telegram app code is included, but tokens/chat IDs are not.

## 9. Optional pcap collector

The collector is only for diagnostics. It watches SIP packets and posts compact SIP metadata to billing.

```bash
cp freeswitch/pcap_collector.py /usr/local/sbin/billing-pcap-collector.py
cp freeswitch/systemd/billing-pcap-collector.service /etc/systemd/system/billing-pcap-collector.service
nano /etc/systemd/system/billing-pcap-collector.service
systemctl daemon-reload
systemctl enable --now billing-pcap-collector
```

Set `VOIP_LOCAL_IPS` in the service to the public/private IPs of the FreeSWITCH server.

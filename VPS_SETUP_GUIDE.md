# OpenRung — Self-hosting on your own VPS (Complete Setup Guide)

This guide explains how to run the **OpenRung broker + relay (+ optional relay hub)** on your own VPS, end-to-end. It is based on the repository's `deploy/` manifests.

## Architecture recap

- **Broker** = control plane (relay directory, registrations, heartbeats, telemetry). Tiny traffic → any VPS.
- **Relay** = data plane (Xray VLESS+REALITY+Vision) on a public IP. Needs TCP/443 open.
- **Relay Hub** (optional) = tunnel for CGNAT volunteer relays. Run only where bandwidth is cheap.

```
client ──VLESS+REALITY──► relay (your VPS) ──► open internet
client ──relay discovery──► broker (your VPS)
relay  ──register+heartbeat──► broker
```

## Prerequisites

- A Linux VPS with a **public IP** (Ubuntu 22.04+ recommended).
- Docker + Compose v2 installed.
- Ports open in firewall/security group:
  - Broker: TCP 8080 (or fronted by Cloudflare/HTTPS)
  - Relay: TCP 443 (mandatory); optionally UDP for QUIC
  - Hub (optional): TCP 9443 control, TCP 9444 HTTP API, UDP reflector ports, public port range.
- A long random token for registration.

## 1. Install Docker

```sh
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
docker compose version
```

## 2. Clone repository

```sh
git clone https://github.com/uzinlay85/zin-openrung.git
cd zin-openrung
```

## 3. Deploy the Broker

```sh
cd deploy/broker
install -m 0600 .env.example .env
```

Edit `.env`:

```sh
OPENRUNG_VOLUNTEER_TOKEN=<long-random-secret>   # shared with relays
# OR explicitly allow anonymous:
# OPENRUNG_ALLOW_ANONYMOUS_REGISTRATION=true

OPENRUNG_RELAY_STORE=memory          # or postgres
OPENRUNG_TELEMETRY_STORE=jsonl
OPENRUNG_DASHBOARD_TOKEN=<optional>
OPENRUNG_API_TOKEN=<optional, differs from others>
```

Start:

```sh
docker compose up -d --build
docker compose logs -f
curl http://localhost:8080/healthz
curl http://localhost:8080/api/v1/relays
```

Broker listens on `:8080` (host networking). Point relays/hubs at `OPENRUNG_BROKER_URL=http://<vps-ip>:8080`.

### Optional: TLS via Cloudflare/Caddy

Front the origin with Cloudflare or a reverse proxy for HTTPS. Set client apps' HTTPS broker URL to the Cloudflare hostname. See `deploy/broker/Caddyfile` and `origin-tls.md`.

## 4. Deploy a Relay

```sh
cd ../relay
install -m 0600 .env.example .env
```

Minimum `.env`:

```sh
OPENRUNG_BROKER_URL=http://<broker-ip>:8080
OPENRUNG_PUBLIC_HOST=<your-public-ip-or-dns>
OPENRUNG_VOLUNTEER_TOKEN=<same-as-broker>   # or foundation token
```

Foundation-operated (optional): set only `OPENRUNG_FOUNDATION_TOKEN` (forces foundation class + direct mode). Do not set `OPENRUNG_NODE_CLASS`/`OPENRUNG_MODE` with it.

Start:

```sh
docker compose up -d --build
docker compose logs -f   # expect: registered relay ... / heartbeat ok
```

Firewall: open TCP 443 to world. The relay bundles pinned Xray-core.

### Operations

```sh
docker compose logs -f
docker compose restart
docker compose down
```

Shutdown is graceful (SIGTERM).

### Update Xray-core

Change version in Dockerfile and docker-compose.yml, then:

```sh
docker compose build --build-arg XRAY_VERSION=v26.3.27
```

## 5. (Optional) Relay Hub for CGNAT relays

Run on cheap/unmetered bandwidth provider.

```sh
cd ../relayhub
install -m 0600 .env.example .env
mkdir -p certs   # put hub.crt + hub.key
```

Minimum `.env`:

```sh
OPENRUNG_HUB_PUBLIC_HOST=hub.example.com
OPENRUNG_BROKER_URL=http://<broker-ip>:8080
OPENRUNG_HUB_PORT_RANGE=...   # size to concurrent tunnels
```

Start:

```sh
docker compose up -d --build
```

Then point CGNAT relays at it:

```sh
OPENRUNG_TUNNEL=true
OPENRUNG_HUB_ADDR=hub.example.com:9443
OPENRUNG_VOLUNTEER_TOKEN=<same token>
```

Open firewall: control port, public port range, reflector UDP ports, punch coordinator TCP.

## 6. Client apps

Point client apps (mobile/desktop/terminal) to your broker HTTPS URL. They discover relays via broker. No other config needed.

## 7. Persistence & backups

- Broker telemetry: JSONL at `/var/lib/openrung/telemetry.jsonl` or Postgres.
- Relay identity: keep `.env` (registration creds + stable identity) safe.
- Use `OPENRUNG_RELAY_STORE=postgres` for durable relay state across restarts.

## 8. Security notes

- Never commit `.env`.
- Broker fails closed without token or anonymous opt-in.
- Foundation token is self-contained bearer; protect it.
- Trusted proxies: set `OPENRUNG_TRUSTED_PROXY_CIDRS` when behind proxy.

## 9. Troubleshooting

- `healthz` fails → check `docker compose logs`, port 8080.
- Relay not registering → verify broker URL reachable, token matches, public host set.
- Clients can't connect → TCP 443 open, public host correct.
- Hub high egress → expected; move to unmetered host.

## References

- `deploy/broker/README.md`
- `deploy/relay/README.md`
- `deploy/relayhub/README.md`
- `docs/architecture.md`, `docs/wss-fallback.md`, `docs/api.md`
---

# OpenRung — ကိုယ်ပိုင် VPS မှာ Host လုပ်ခြင်း (အပြည့်အစုံ လမ်းညွှန်)

ဒီလမ်းညွှန်သည် OpenRung ၏ broker + relay (နှင့် ရွေးချယ်နိုင်သော relay hub) ကို ကိုယ်ပိုင် VPS ပေါ်မှာ end-to-end တည်ဆောက်နည်းကို ရှင်းပြထားသည်။ repository ၏ `deploy/` manifest များအပေါ် အခြေခံထားသည်။

## အကျဉ်းချုပ်

- **Broker** = control plane (relay directory, registration, heartbeat, telemetry)။ traffic နည်း → ဘယ် VPS မှာမဆို run နိုင်။
- **Relay** = data plane (Xray VLESS+REALITY+Vision) public IP ပေါ်တွင် run။ TCP/443 ဖွင့်ရန်လို။
- **Relay Hub** (ရွေးချယ်နိုင်) = CGNAT volunteer relay များအတွက် tunnel။ bandwidth စရိတ်သက်သာသောနေရာမှာသာ run ပါ။

## လိုအပ်ချက်များ

- public IP ရှိသော Linux VPS (Ubuntu 22.04+ အကြံပြု)။
- Docker + Compose v2 တပ်ဆင်ထားရန်။
- firewall/security group တွင် port များဖွင့်ထားရန်:
  - Broker: TCP 8080 (သို့မဟုတ် Cloudflare/HTTPS ဖြင့် front)
  - Relay: TCP 443 (မဖြစ်မနေ); QUIC အတွက် UDP ရွေးချယ်နိုင်
  - Hub (ရွေးချယ်နိုင်): TCP 9443, TCP 9444, UDP reflector, public port range
- registration အတွက် ရှည်လျားသော random token။

## 1. Docker တပ်ဆင်ခြင်း

```sh
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
docker compose version
```

## 2. Repository clone လုပ်ခြင်း

```sh
git clone https://github.com/uzinlay85/zin-openrung.git
cd zin-openrung
```

## 3. Broker deploy လုပ်ခြင်း

```sh
cd deploy/broker
install -m 0600 .env.example .env
```

`.env` ကို တည်းဖြတ်ပါ:

```sh
OPENRUNG_VOLUNTEER_TOKEN=<ရှည်လျားသော-random-secret>
# သို့မဟုတ် anonymous ခွင့်ပြု:
# OPENRUNG_ALLOW_ANONYMOUS_REGISTRATION=true

OPENRUNG_RELAY_STORE=memory
OPENRUNG_TELEMETRY_STORE=jsonl
OPENRUNG_DASHBOARD_TOKEN=<ရွေးချယ်နိုင်>
OPENRUNG_API_TOKEN=<ရွေးချယ်နိုင်>
```

စတင်ရန်:

```sh
docker compose up -d --build
docker compose logs -f
curl http://localhost:8080/healthz
curl http://localhost:8080/api/v1/relays
```

Broker သည် `:8080` တွင် နားထောင်သည်။ relay/hub များကို `OPENRUNG_BROKER_URL=http://<vps-ip>:8080` ဖြင့် ညွှန်ပါ။

### TLS (ရွေးချယ်နိုင်)

Cloudflare သို့မဟုတ် reverse proxy ဖြင့် HTTPS front လုပ်ပါ။ client app များ၏ broker URL ကို Cloudflare hostname သို့ သတ်မှတ်ပါ။ `deploy/broker/Caddyfile` နှင့် `origin-tls.md` ကို ကြည့်ပါ။

## 4. Relay deploy လုပ်ခြင်း

```sh
cd ../relay
install -m 0600 .env.example .env
```

အနည်းဆုံး `.env`:

```sh
OPENRUNG_BROKER_URL=http://<broker-ip>:8080
OPENRUNG_PUBLIC_HOST=<သင့်-public-ip-သို့-dns>
OPENRUNG_VOLUNTEER_TOKEN=<broker-နှင့်တူ>
```

Foundation-operated (ရွေးချယ်နိုင်): `OPENRUNG_FOUNDATION_TOKEN` တစ်ခုသာ သတ်မှတ်ပါ (foundation class + direct mode ဖြစ်စေသည်)။ ၎င်းနှင့်အတူ `OPENRUNG_NODE_CLASS`/`OPENRUNG_MODE` မထည့်ပါနှင့်။

စတင်ရန်:

```sh
docker compose up -d --build
docker compose logs -f   # registered relay ... / heartbeat ok မျှော်လင့်ရ
```

Firewall: TCP 443 ကို world သို့ ဖွင့်ပါ။ relay သည် pinned Xray-core ပါဝင်သည်။

### လည်ပတ်ခြင်း

```sh
docker compose logs -f
docker compose restart
docker compose down
```

Shutdown သည် graceful ဖြစ်သည် (SIGTERM)။

### Xray-core အဆင့်မြှင့်ခြင်း

Dockerfile နှင့် docker-compose.yml တွင် version ပြောင်းပြီး:

```sh
docker compose build --build-arg XRAY_VERSION=v26.3.27
```

## 5. (ရွေးချယ်နိုင်) CGNAT relay များအတွက် Relay Hub

bandwidth စရိတ်သက်သာသော provider (Hetzner, OVH စသည်) တွင် run ပါ။

```sh
cd ../relayhub
install -m 0600 .env.example .env
mkdir -p certs   # hub.crt + hub.key ထည့်ပါ
```

အနည်းဆုံး `.env`:

```sh
OPENRUNG_HUB_PUBLIC_HOST=hub.example.com
OPENRUNG_BROKER_URL=http://<broker-ip>:8080
OPENRUNG_HUB_PORT_RANGE=...   # concurrent tunnel အရေအတွက်
```

စတင်ရန်:

```sh
docker compose up -d --build
```

CGNAT relay များကို ညွှန်ရန်:

```sh
OPENRUNG_TUNNEL=true
OPENRUNG_HUB_ADDR=hub.example.com:9443
OPENRUNG_VOLUNTEER_TOKEN=<တူညီသော token>
```

Firewall: control port, public port range, reflector UDP, punch coordinator TCP ဖွင့်ပါ။

## 6. Client app များ

mobile/desktop/terminal client app များကို သင့် broker HTTPS URL သို့ ညွှန်ပါ။ relay များကို broker မှ ရှာဖွေသည်။ အခြား config မလို။

## 7. Persistence နှင့် အရန်ကူးယူခြင်း

- Broker telemetry: `/var/lib/openrung/telemetry.jsonl` သို့မဟုတ် Postgres။
- Relay identity: `.env` (registration creds + stable identity) ကို လုံခြုံစွာ သိမ်းပါ။
- restart များတွင် တည်တံ့ရန် `OPENRUNG_RELAY_STORE=postgres` သုံးပါ။

## 8. Security ဆိုင်ရာ သတိပေးချက်များ

- `.env` ကို commit မလုပ်ပါနှင့်။
- token သို့မဟုတ် anonymous opt-in မရှိပါက broker စတင်မည်မဟုတ်။
- Foundation token သည် self-contained bearer ဖြစ်သည်; ကာကွယ်ပါ။
- proxy နောက်တွင်ရှိပါက `OPENRUNG_TRUSTED_PROXY_CIDRS` သတ်မှတ်ပါ။

## 9. ပြဿနာဖြေရှင်းခြင်း

- `healthz` မအောင်မြင် → `docker compose logs` နှင့် port 8080 စစ်ပါ။
- relay မ register ဖြစ် → broker URL ရောက်နိုင်မှု, token တူညီမှု, public host သတ်မှတ်ထားမှု စစ်ပါ။
- client ချိတ်ဆက်မရ → TCP 443 ဖွင့်ထားမှု, public host မှန်ကန်မှု စစ်ပါ။
- hub egress များ → ပုံမှန်ဖြစ်သည်; unmetered host သို့ ပြောင်းပါ။

## ဆက်စပ် ကိုးကားချက်များ

- `deploy/broker/README.md`
- `deploy/relay/README.md`
- `deploy/relayhub/README.md`
- `docs/architecture.md`, `docs/wss-fallback.md`, `docs/api.md`


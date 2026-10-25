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

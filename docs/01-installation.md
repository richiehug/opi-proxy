# Installation

## Prerequisites

Install:

- Docker Engine
- Docker Compose

This public distribution is Docker-only. Node.js usage stays in the private source repo for development work.

## Quick Installation

1. Copy `.env.example` to `.env`.
2. Copy `config/terminal-profiles.json.example` to `config/terminal-profiles.json`.
3. Copy `config/terminals.json.example` to `config/terminals.json`.
4. Set each terminal's IP address, model, profile, and workstation ID in `config/terminals.json`.
5. Pull the release image from GHCR or load the bundled tar archive.
6. Run `docker compose run --rm auth-init`.
7. Start the proxy with `docker compose up -d`.
8. Check the startup logs and call `/health`.

Continue with [Configuration](02-configuration.md) before processing a real payment.

## Required Network Paths

For terminal transactions, the host must expose the OPI device callback port as well as the HTTP API. The public Linux bundle uses Docker host networking so the proxy uses the host network stack directly. This avoids Docker bridge routing surprises on hosts with terminals reachable through interfaces such as `wlan1` or `eth0`.

| Direction | Purpose | Default port |
| --- | --- | --- |
| POS/ECR to proxy | HTTP or HTTPS API | `PORT=3001` |
| Proxy to terminal | OPI pay channel | `payChannel0=4100` |
| Terminal to proxy | OPI device callbacks | `deviceChannel1=4102` |

Both OPI directions matter. A payment can be sent successfully on `4100` and still lose its final outcome if the terminal cannot reach the proxy on `4102` for receipt, display, journal, or input callbacks. If you change `deviceChannel1`, change it in the terminal JSON and on the terminal itself; the legacy `DEVICE_CHANNEL1_PUBLIC_PORT` variable is not used with host networking.

## Image Source

Online from GHCR:

```bash
docker compose pull
```

That pulls `ghcr.io/richiehug/opi-proxy:<version>`.

Offline from a public release asset, for example on Raspberry Pi 5:

```bash
docker load -i image/opi-proxy-<version>-linux-arm64.tar
```

After `docker load`, you can keep the same image reference in `.env` if the archive was built from the matching release tag.

The public repo itself stays GHCR-first.

## First-Time Setup

Generate `config/auth.json` and the bootstrap admin token:

```bash
docker compose run --rm auth-init
```

The bootstrap admin bearer is shown once in the terminal. Store it securely; `config/auth.json` contains only its hash.

## Start And Verify

```bash
docker compose up -d
```

Check logs:

```bash
docker compose logs -f proxy
```

For LAN-connected payment terminals, make sure:

- `payChannel0` and `deviceChannel1` match the terminal configuration
- the host firewall allows inbound TCP on `deviceChannel1`
- the host can connect to the terminal on `payChannel0`
- the terminal can reach the Docker host by IP
- `docker-compose.yml` keeps `network_mode: host` on Linux hosts such as Raspberry Pi

Default localhost health check:

```bash
curl http://127.0.0.1:3001/health
```

The public `.env.example` uses HTTP-first settings (`TLS_MODE=disabled` and `ALLOW_INSECURE_REMOTE_HTTP=true`) so the initial LAN setup can start without a certificate.

Before exposing the API beyond a trusted setup network, switch to `TLS_MODE=required`, provide a trusted certificate and key, set `ALLOW_INSECURE_REMOTE_HTTP=false`, and use `https://`.

Finally, test live OPI reachability:

- `GET /terminals/:terminalAlias/ping` checks network reachability only.
- `GET /terminals/:terminalAlias/status` opens an OPI flow and confirms the terminal application can respond. This route should only be used for initial status checks, not for continuous availability checks.

## Optional Development Certificate

Generate a self-signed certificate for quick HTTPS testing:

```bash
docker compose run --rm tls-init-dev
```

That certificate is for testing only. Shared environments should use a proper certificate and key mounted into `certs/`.

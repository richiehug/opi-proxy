# Security And TLS

The initial public configuration is designed for straightforward testing on a trusted network. Read this page before exposing the proxy more widely, enabling HTTPS, replacing certificates, or tightening firewall and token access.

## Transport Policy

The proxy follows this runtime policy:

- localhost may use HTTP
- non-local traffic should use TLS
- insecure remote HTTP is only allowed when `ALLOW_INSECURE_REMOTE_HTTP=true`

The public `.env.example` deliberately permits HTTP for an easy first LAN setup. That is not the recommended final state for a shared or remotely reachable environment.

## Terminal OPI Peers

The proxy also checks the peer IP address on OPI terminal communication:

- pay-channel responses must come from the configured terminal `ip`
- device-channel callbacks are accepted only from the configured terminal `ip` for the active terminal context
- unexpected device-channel peers are rejected without an OPI response

This is a network-level guardrail in addition to bearer-token authentication on the public HTTP API. Keep `config/terminals.json` terminal IPs accurate, and avoid exposing the device callback port outside the trusted terminal network.

For a production-style Docker deployment, use:

- `HOST=0.0.0.0`
- `network_mode: host` in the public Docker bundle on Linux hosts
- `TLS_MODE=required`
- `ALLOW_INSECURE_REMOTE_HTTP=false`
- a trusted certificate and private key

`PUBLISHED_HOST` only applies if you intentionally switch the compose file back to Docker bridge networking with `ports:`.

## TLS Modes

| Mode | Local bind | Non-local bind | Use it for |
| --- | --- | --- | --- |
| `auto` | HTTP is allowed | TLS certificate and key are required | Development that may move between loopback and LAN binds |
| `required` | HTTPS | HTTPS | Shared, LAN, and remotely reachable deployments |
| `disabled` | HTTP | HTTP only when explicitly allowed | Initial testing on a trusted network |

Remote HTTP with `TLS_MODE=disabled` is blocked unless `ALLOW_INSECURE_REMOTE_HTTP=true` is set intentionally.

## Certificate Files

The bundle expects file-based certificates so Docker can mount them directly:

- `TLS_CERT_FILE`
- `TLS_KEY_FILE`

These paths are resolved from the bundle root and mounted into `/app/certs` inside the container. Configure both or neither; startup validation rejects a certificate without its matching key.

Example:

```env
TLS_MODE=required
TLS_CERT_FILE=certs/proxy.crt
TLS_KEY_FILE=certs/proxy.key
ALLOW_INSECURE_REMOTE_HTTP=false
```

## Development TLS

For a quick self-signed certificate during testing:

```bash
docker compose run --rm tls-init-dev
```

That creates:

- `certs/dev.crt`
- `certs/dev.key`

This is convenient for development, but clients will warn until the certificate is trusted locally.

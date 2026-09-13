# Release Notes

## 1.0.0

# 🎉 OPI Proxy is out!

OPI Proxy 1.0.0 gives POS systems, local applications, middleware, and kiosks a simple HTTP and JSON interface for OPI payment operations. Run one Docker service while the proxy handles the underlying TCP connections, XML messages, terminal callbacks, and transaction follow-up.

## Highlights in 1.0.0

- Docker-first distribution for `linux/amd64` and `linux/arm64`, with a versioned GHCR image and optional offline image archive.
- Scoped bearer-token authentication and a TLS-ready runtime.
- Payments, pre-authorisations, capture, cancellation, refunds, reversal, cashback, and Force Acceptance.
- Structured DCC results when supplied by the terminal.
- Synchronous responses or asynchronous operations with transaction polling.
- Idempotency keys, durable terminal locks, and persisted transaction history.
- Customer and merchant receipt handling with transaction-linked receipt retrieval.
- JSON or SQLite storage, with migration, export, and verification commands.
- Normal and extended diagnostics with sensitive-field redaction.

## API at a glance

| Flow | Routes | Purpose |
| --- | --- | --- |
| Payments | `/payment`, `/refund`, `/reversal` | Start terminal financial operations. |
| Stored transactions | `/transactions/{id}`, `/capture`, `/cancel`, `/refund` | Read results and operate on stored transactions. |
| Receipts | `/receipts`, `/receipts/latest`, `/reprint` | Retrieve stored receipts or request the terminal's last ticket. |
| Recovery | `/repeat`, `PATCH /transactions/{id}` | Repeat the last card result or manually reconcile verified transaction facts. |
| Terminal control | `/info`, `/status`, `/ping`, `/activate`, `/deactivate`, `/abort` | Inspect and control the terminal. |
| Maintenance | `/submit`, `/close`, `/config`, `/init`, `/reset` | Submit transactions and maintain terminal configuration and sessions. |

Terminal routes use `/terminals/{terminalAlias}` as their prefix. Transaction operations use `/transactions/{id}`. See the OpenAPI description for full paths and schemas.

`/submit` uses OPI `TransmitTrx`; `/transmit` remains a compatibility alias with the same handler and behavior. `/repeat` retrieves the last card result using `RepeatLastMessage` and checks its original request ID. `/reprint` requests the last ticket and is a separate operation.

`202 Accepted` means an operation is pending. Poll its transaction resource for the final result. An `unknown` outcome requires reconciliation before treating the payment as successful or failed.

## Downloads and documentation

- Docker image: `ghcr.io/richiehug/opi-proxy:1.0.0`.
- Optional offline ARM64 archive: `opi-proxy-1.0.0-linux-arm64.tar`.
- [Installation and integration](https://github.com/richiehug/opi-proxy/blob/1.0.0/README.md)
- [OpenAPI description](https://github.com/richiehug/opi-proxy/blob/1.0.0/openapi.yaml)

Runtime configuration, tokens, and transaction data remain outside the container. New tokens use the `opi_proxy_` prefix; existing tokens remain valid against their stored hashes.

Created and maintained by [Richard Hug](https://richiehug.com/).

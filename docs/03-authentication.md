# Authentication

## Authentication Model

The proxy protects all terminal, transaction, and receipt routes with bearer-token authentication.

`GET /health` is the only unauthenticated route.

All other routes require an `Authorization: Bearer ...` header.

## Initialization

Generate the auth file and bootstrap admin token:

```bash
docker compose run --rm auth-init
```

This creates `config/auth.json`.

The bootstrap admin bearer is printed once when the file is created. Store it in a secret manager; the raw value cannot be recovered from `auth.json`.

## What `auth.json` Contains

`config/auth.json` stores:

- bearer token hashes, not raw token secrets
- enabled scopes for each token
- terminal access groups and explicit terminal access

The bootstrap admin token is intended for setup and recovery. Create narrower tokens for day-to-day integrations.

## Scopes

| Scope | Allows |
| --- | --- |
| `terminals:read` | terminal info, live status, and ping |
| `terminals:write` | activation, deactivation, submission, abort, close, config, init, and reset |
| `transactions:read` | transaction lookup |
| `transactions:write` | payment, refund, capture, cancellation, and repeat operations |
| `receipts:read` | receipt lookup and ticket reprint |
| `admin:manage` | manual transaction reconciliation with `PATCH` |
| `*` | every scope; reserve for bootstrap administration |

Scopes and terminal access are both enforced. A token must have the required route scope and access to the selected terminal.

## Create A Restricted Token

For example, create a POS token that can take payments and read their result for one terminal:

```bash
docker compose run --rm proxy node src/index.js auth add-bearer \
  --id pos-lane-1 \
  --scopes transactions:read,transactions:write,receipts:read \
  --terminals lane-1
```

Use `--groups <group-id>` for a configured terminal group or `--all` for every terminal. The command prints the raw bearer once.

Useful management commands:

```bash
docker compose run --rm proxy node src/index.js auth list
docker compose run --rm proxy node src/index.js auth rotate --id pos-lane-1
docker compose run --rm proxy node src/index.js auth revoke --id pos-lane-1
```

Rotation issues a new raw secret for the same token ID. Update the client immediately and remove the old secret from its configuration.

## Example Header

```http
Authorization: Bearer opi_proxy_xxxxxxxxxxxxxxxxxxxxxxxx
```

Never place bearer tokens in URLs, request bodies, terminal JSON, or support logs. Send them only through the `Authorization` header over a trusted connection.

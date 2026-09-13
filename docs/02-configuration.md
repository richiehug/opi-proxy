# Configuration

The public bundle is configured through `.env` for deployment-wide behavior and JSON files under `config/` for terminals, profiles, and authentication.

## Before You Start

Create the environment-owned files once, then edit the copies:

```bash
cp .env.example .env
cp config/terminal-profiles.json.example config/terminal-profiles.json
cp config/terminals.json.example config/terminals.json
```

Run `docker compose run --rm auth-init` to create `config/auth.json`, then validate the configuration by starting the proxy and checking its startup logs.

## Configuration Files

| File | Owner | Purpose | Upgrade action |
| --- | --- | --- | --- |
| `.env` | Environment | Runtime, storage, networking, TLS, logging, and global defaults | Preserve and compare with the new `.env.example` |
| `config/terminal-models.json` | OPI Proxy release | Supported model capabilities and defaults | Review and normally refresh from the release |
| `config/terminal-profiles.json.example` | OPI Proxy release | Profile examples | Review only |
| `config/terminals.json.example` | OPI Proxy release | Terminal examples | Review only |
| `config/terminal-profiles.json` | Merchant/integrator | Shared settings for groups of terminals | Preserve |
| `config/terminals.json` | Merchant/integrator | Terminal IPs, models, profiles, and overrides | Preserve |
| `config/auth.json` | Environment | Hashed bearer tokens, scopes, and terminal access | Preserve |

## How Defaults Are Resolved

For settings that can be supplied at more than one level, the first defined value wins:

1. HTTP request
2. terminal entry in `terminals.json`
3. terminal profile in `terminal-profiles.json`
4. terminal model in `terminal-models.json`
5. global `.env` default

This distinction matters when troubleshooting: a value shown in an effective OPI request may come from configuration even when the merchant did not submit it in the HTTP body. Set `LOG_LEVEL=extended` to see the resolved value sources and the effective OPI request.

## Environment Variables

The tables below cover the options supported by the public Docker bundle. Durations ending in `_MS` use milliseconds; durations ending in `_SECONDS` use seconds. Use whole numbers for ports and durations, and `true` or `false` for boolean settings. Zero is supported only where the description explicitly says so.

### Image And Network

| Variable | Values / public default | What it controls |
| --- | --- | --- |
| `OPI_PROXY_IMAGE` | Versioned GHCR image | Docker image used by Compose. Change this to a matching published tag or a locally loaded test image. |
| `HOST` | IP address; `0.0.0.0` | API bind address inside the proxy process. |
| `PORT` | `3001` | HTTP or HTTPS API port. |
| `POS_LISTENER_HOST` | IP address; `0.0.0.0` | Bind address for terminal device-channel callbacks. It rarely needs changing. |
| `PUBLISHED_HOST` | Empty in the public bundle | Legacy bridge-network API bind. It is ignored while the supplied Compose file uses `network_mode: host`. |
| `DEVICE_CHANNEL1_PUBLISHED_HOST` | Empty in the public bundle | Legacy bridge-network callback bind. It is ignored with host networking. |
| `DEVICE_CHANNEL1_PUBLIC_PORT` | `4102` | Legacy bridge-network callback mapping. With host networking, the actual listener port comes from the terminal's `deviceChannel1` configuration. |

The public Linux bundle uses host networking. Normally, the proxy sends OPI requests to the terminal on `payChannel0=4100` and listens for terminal callbacks on `deviceChannel1=4102`. If you change either port, change the terminal JSON configuration and terminal setup together.

### Request Defaults

| Variable | Values / public default | What it controls |
| --- | --- | --- |
| `DEFAULT_LANGUAGE` | Language code; `en` | Fallback OPI language when no request, terminal, profile, or model value exists. |
| `DEFAULT_CURRENCY` | ISO currency code; `CHF` | Fallback transaction currency. |

These are fallbacks, not mandatory merchant inputs. Terminal, profile, and model configuration can override them.

### Timeouts And Recovery

| Variable | Values / public default | What it controls |
| --- | --- | --- |
| `OPI_CONNECT_TIMEOUT_MS` | `3000` | Maximum time to establish the TCP connection to a terminal. |
| `REQUEST_TIMEOUT_MS` | `90000` | Idle timeout while waiting for pay-channel data or device callbacks. Activity refreshes this timer. |
| `REQUEST_MAX_DURATION_MS` | `300000` | Hard maximum for an OPI operation, even while callbacks continue. Must be at least `REQUEST_TIMEOUT_MS`. |
| `ACTION_TIMEOUT_MS` | `50000` | General timeout used for an individual terminal-side action and terminal-lock sizing. |
| `ASYNC_ACCEPT_GRACE_MS` | `1000` | In async mode, how long to wait for an immediate terminal failure before returning `202 Accepted`. Set `0` to skip the grace wait. Higher values catch slower immediate failures but delay successful async responses. |
| `RETRY_AFTER_DEVICE_UNAVAILABLE_MS` | `5000` | Short cooldown after a fresh `DeviceUnavailable` or `PrintLastTicket`; requests during it receive `503` and `Retry-After`. Set `0` to disable the cooldown. |
| `TRANSACTION_PENDING_TIMEOUT_SECONDS` | `300` | Time before an unresolved pending transaction becomes eligible for recovery handling. |
| `TRANSACTION_RECOVERY_GRACE_SECONDS` | `30` | Startup grace before older pending transactions are marked for reconciliation. Set `0` to apply no startup grace. |

`REQUEST_TIMEOUT_MS` and `REQUEST_MAX_DURATION_MS` govern the live OPI socket operation. The transaction pending and recovery settings govern persisted state after the live operation can no longer be trusted; they are intentionally separate.

### Storage And Response Behavior

| Variable | Options / public default | What it controls |
| --- | --- | --- |
| `STORAGE` | `db` or `json`; `db` | Transaction and receipt backend. `db` uses SQLite; `json` uses files under `data/`. |
| `DB_URL` | SQLite file URL; `file:/app/data/opi-proxy.sqlite` | SQLite database location when `STORAGE=db`. |
| `RESPONSE_MODE` | `async` or `sync`; `async` | Default response style for supported terminal operations. Async avoids making the caller's HTTP timeout decide the terminal outcome. |

The runtime fallback, when these variables are absent, remains `STORAGE=json` and `RESPONSE_MODE=sync` for backward compatibility. The shipped public `.env.example` uses `db` and `async` for new installations.

In async mode, a `202 Accepted` means the OPI request was handed to the terminal and remained pending after the acceptance grace window. It does not mean the payment succeeded. Use the `Location` header or returned transaction ID to poll `GET /transactions/:transactionId` until the status is final.

A request body `responseMode` or `Prefer: respond-async` header can override the environment default on async-enabled routes. Mutating routes also support `Idempotency-Key`; reuse the same key and identical request content for safe retries.

### Logging

| Variable | Options / public default | What it controls |
| --- | --- | --- |
| `LOG_LEVEL` | `normal` or `extended`; `normal` | `normal` writes concise operational logs. `extended` adds DEBUG conversion details and separate OPI trace files. |
| `TRACE_LOG_MAX_SIZE_MB` | `200` | Maximum OPI trace-file size before rotation. It applies only when extended tracing is active. |

Normal payment logs include the sanitized JSON body submitted by the merchant. Extended logging additionally shows resolved configuration and the effective OPI request. Sensitive card and authorization data is redacted, but extended logs should still be protected as operational data.

### TLS

| Variable | Options / public default | What it controls |
| --- | --- | --- |
| `TLS_MODE` | `disabled`, `auto`, or `required`; `disabled` | `disabled` serves HTTP, `required` always uses TLS, and `auto` permits HTTP only on a loopback bind. |
| `TLS_CERT_FILE` | File path; `certs/dev.crt` | PEM certificate path. Relative paths resolve from the bundle root. |
| `TLS_KEY_FILE` | File path; `certs/dev.key` | Matching PEM private-key path. Certificate and key must be configured together. |
| `ALLOW_INSECURE_REMOTE_HTTP` | `true` or `false`; `true` | Required when `TLS_MODE=disabled` and `HOST` is non-loopback. Set it to `false` when TLS is enabled. |

The public template favors an easy HTTP first run. For a shared or remotely reachable environment, use `TLS_MODE=required`, provide a trusted certificate and key, and set `ALLOW_INSECURE_REMOTE_HTTP=false`.

### Advanced OPI Defaults

These settings are fallback behavior for terminal integrations. Prefer request, terminal, or profile configuration when the behavior differs by terminal. Change them only when the terminal/acquirer setup requires it.

| Variable | Options / public default | What it controls |
| --- | --- | --- |
| `DEFAULT_REQUEST_TRANSACTION_INFORMATION` | `true` or `false`; `true` | Requests terminal transaction information when the applicable OPI attribute is serialized. |
| `DEFAULT_REQUEST_TOKEN` | `true` or `false`; `true` | Requests a reusable terminal token when supported. |
| `DEFAULT_REQUEST_RECEIPT_HEADER` | `true` or `false`; `true` | Requests the terminal receipt header when supported. |
| `DEFAULT_ALLOW_MANUAL_PAN` | `true` or `false`; `false` | Global fallback for manual PAN-entry capability. |
| `DEFAULT_ALLOW_FORCE_ACCEPTANCE` | `true` or `false`; `false` | Global fallback for Force Acceptance capability. This does not itself request Force Acceptance on a payment. |
| `POS_EJOURNAL_STATUS` | `Available`, `MemoryLow`, or `Unavailable`; `Available` | Fallback OPI E-journal status when no terminal/model default exists. |
| `DEFAULT_LOGOFF_MODE` | OPI logoff-mode value; `3` | Fallback mode for applicable logoff requests. Keep `3` unless instructed otherwise for the terminal setup. |

False request flags are omitted from OPI when the protocol defines absence as false. Model defaults may take precedence over these global fallbacks.

## Terminal Models, Profiles, And Terminals


Use the layers for different purposes:

| Layer | Use it for |
| --- | --- |
| Model | Model capabilities and safe model defaults |
| Profile | Settings shared by a group of terminals in one environment |
| Terminal | IP address, workstation identity, enabled state, and exceptions for one device |

`workstationId` is the POS/ECR identity sent as OPI `WorkstationID`. Give every checkout or terminal context a stable, unique value such as `lane-1` or `mobile-dx`. If omitted, the proxy uses the terminal alias. Terminals can share a callback port when their workstation IDs are distinct.

If `ports` are omitted, the proxy uses the model defaults, normally `payChannel0=4100` and `deviceChannel1=4102`. The device listener accepts callbacks only from the configured terminal IP for the active context.

## Receipt Handling

**Recommended:** explicitly set both copies to `external`, which maps to OPI `Available`. This supports printerless terminals and lets the ECR retrieve and store receipts before printing. Use `local` (OPI `PrintLocal`) only when the terminal has a working printer and should print the copy itself. An unsupported local-printer request can return `DeviceUnavailable` and leave a pending receipt.

Receipt routing can be set in a terminal entry and overridden per payment:

```json
{
  "dx-prod": {
    "terminalModel": "DX8000",
    "terminalProfile": "dx8000-default",
    "ip": "192.168.50.33",
    "enabled": true,
    "receiptHandling": {
      "customer": "external",
      "merchant": "external"
    }
  }
}
```

| Value | Behavior |
| --- | --- |
| `external` | The terminal sends the copy to the proxy over the device callback channel. The selected storage backend stores it. |
| `local` | A terminal with a printer handles the copy locally. |

DX8000 defaults to local receipt handling because it has a built-in printer. Printerless EX8000 and A77 profiles default to external handling. When external handling is used, the ECR is responsible for displaying, sending, printing, or skipping the customer copy.

`STORAGE=json` writes receipt files under `data/receipts`. `STORAGE=db` writes receipt records into the SQLite database selected by `DB_URL`. Retrieve receipts through the same HTTP endpoints in either mode. See [Storage and Receipts](05-storage-and-receipts.md) for the complete flow.

Service requests such as `/init` and `/config` do not use card-request printer status fields. Any service receipt returned by the terminal is captured through the device callback channel.

The Docker image can be overridden with `OPI_PROXY_IMAGE`. Leave it unset to use the version pinned in the release bundle.

# Upgrading

## Upgrade Flow

1. Stop or drain new terminal operations.
2. Back up `config/`, `data/`, and `.env`.
3. Review every release note between the installed and target versions.
4. Pull the new image or load the matching archive.
5. Preserve environment-owned files and compare `.env` with the new `.env.example`.
6. Review whether `terminal-models.json` should be refreshed from the release.
7. Restart with `docker compose up -d`.
8. Check startup logs and `GET /health`.
9. Verify one terminal with `/ping`, `/status`, and a controlled payment/receipt flow.

## Integration Notes

- DCC payments return `transaction.dcc` when DCC was offered. Clients should accept the nested object and use `accepted` to distinguish the cardholder's choice.
- Async clients must continue polling after `202 Accepted`. A fast final `PrintLastTicket` is returned immediately as an error instead of first appearing as pending.
- Repeated standalone `/abort` calls return `409 NO_ACTIVE_TRANSACTION` without contacting the terminal. A successful abort returns the affected original transaction.
- Reversal changes the original transaction through `reversing` to `reversed`; integrations that validate status enums should accept both values.
- Terminal-level `receiptHandling` correctly overrides profile/model defaults. Confirm whether each receipt copy should be `external` or `local` before the first payment.
- `forceAcceptance: false` and redundant conclusive `overallResult` fields are omitted from transaction responses. Clients should not rely on their presence.

Version 1.0.0 includes these behaviors. Preserve your runtime data when replacing the distribution.

## Deployment Checks

### Ping Route And Timeout Recovery

`GET /terminals/:terminalAlias/ping` provides a network-level reachability check. It returns `204 No Content` when the terminal answers and `503 Service Unavailable` when it does not. It does not prove that the OPI application is active; use `/status` for that.

`REQUEST_TIMEOUT_MS` is an idle timeout refreshed by device activity. `REQUEST_MAX_DURATION_MS` is the hard cap even when callbacks continue. Review upstream client and reverse-proxy timeouts if using synchronous response mode.

Lost-final-response cases with clear unsupported, declined, invalid-card, card-read-failure, cancellation, or timeout prompts are classified as `failed` or `aborted`. Ambiguous cases remain `unknown` and require reconciliation.

### Reachability And Peer Validation

Async operations return `202 Accepted` only after the OPI request is written and the short `ASYNC_ACCEPT_GRACE_MS` window passes. Immediate `DeviceUnavailable` and `PrintLastTicket` results return errors. `RETRY_AFTER_DEVICE_UNAVAILABLE_MS` controls the short retry cooldown.

The proxy validates terminal peer IPs on both pay-channel responses and device callbacks. Review terminal IPs before upgrading, especially after terminal replacement, VLAN changes, or NAT changes.

### Storage And Async Mode

Runtime fallbacks remain `STORAGE=json` and `RESPONSE_MODE=sync`, while the public `.env.example` recommends SQLite and async mode for new deployments.

To migrate existing JSON data:

```bash
docker compose run --rm proxy node src/index.js storage migrate --from json --to db
docker compose run --rm proxy node src/index.js storage verify
```

In async mode, `GET /transactions/:transactionId` is the source of truth. `status=unknown` means the proxy cannot safely prove the terminal result and the transaction must be reconciled operationally.

Mutating routes support `Idempotency-Key`. Reusing the same key with identical request content returns the same proxy response; different content returns HTTP `409`.

## File Ownership During Upgrades

| Preserve | Review or refresh from the release |
| --- | --- |
| `.env` | `.env.example` |
| `config/auth.json` | `config/terminal-models.json` |
| `config/terminal-profiles.json` | `config/terminal-profiles.json.example` |
| `config/terminals.json` | `config/terminals.json.example` |
| `data/`, `logs/`, `certs/` | `openapi.yaml` and documentation |

Never replace `auth.json`, `terminals.json`, or `terminal-profiles.json` with release examples during an upgrade.

## Image Commands

Pull the new image:

```bash
docker compose pull
```

Or load the matching offline archive:

```bash
docker load -i image/opi-proxy-<version>-linux-arm64.tar
```

Restart and follow the logs:

```bash
docker compose up -d
docker compose logs -f proxy
```

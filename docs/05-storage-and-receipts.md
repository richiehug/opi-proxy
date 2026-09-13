# Storage And Receipts

The shipped public configuration already uses SQLite and external volumes, so most new installations can keep the defaults. Read this page when choosing between SQLite and JSON, migrating existing data, changing receipt handling, or understanding where transactions and receipts are stored.

## Runtime Storage

The public Docker bundle mounts four environment-owned directories outside the container image:

| Directory | Contents |
| --- | --- |
| `config/` | terminal, profile, model, and authentication configuration |
| `data/` | JSON transaction/receipt files or the SQLite database |
| `logs/` | application logs and extended OPI traces |
| `certs/` | TLS certificate and private key files |

That means configuration, auth, transactions, receipts, and logs survive container restarts and image upgrades.

## Choose A Storage Backend

| Backend | Configuration | Best suited to | Receipt location |
| --- | --- | --- | --- |
| SQLite | `STORAGE=db` | New deployments and easier durable querying | `receipts` table in `DB_URL` |
| JSON files | `STORAGE=json` | Backward-compatible existing deployments and direct file inspection | `data/receipts/` |

The public `.env.example` recommends SQLite:

```env
STORAGE=db
DB_URL=file:/app/data/opi-proxy.sqlite
```

If `STORAGE` is omitted, the runtime falls back to `json` for backward compatibility. Existing JSON installations do not need to migrate merely to upgrade the proxy.

To continue using files:

```env
STORAGE=json
```

SQLite enables WAL mode and creates its tables on startup. It stores transactions and receipts only; terminal configuration, authentication, profiles, terminal models, certificates, and logs remain files.

## Migrate Or Export Storage

Stop transaction traffic before migration, take a backup of `data/`, then use the Docker helper commands:

```bash
docker compose run --rm proxy node src/index.js storage migrate --from json --to db
docker compose run --rm proxy node src/index.js storage export --from db --to json
docker compose run --rm proxy node src/index.js storage verify
```

Run `storage verify` after a migration or export before restarting normal traffic.

## Transaction Storage

Transactions are stored by the selected backend and retrieved through `GET /transactions/:transactionId`. The HTTP response is the same whether the backing store is JSON or SQLite.

SQLite transaction rows keep first-class columns for the common reporting fields: dates (`created_at`, `updated_at`), reference, amount, total amount, tip, currency, status, payment method, approval code, terminal alias, terminal model, device terminal id/TID, proxy transaction id, and terminal transaction reference number. The full transaction object is still preserved in `raw_json` for fields that are less common or terminal-specific.

The transaction is the anchor for capture, cancellation, refund, reversal lifecycle, and receipt lookup. Follow-up transactions reference their source through `parentTransactionId` where applicable.

### Transaction Statuses

| Status | Storage meaning |
| --- | --- |
| `pending` | The request reached the terminal and the terminal flow has not ended. |
| `communication_error` | The request did not reach the terminal. |
| `failed` or `aborted` | A final unsuccessful outcome is known. |
| `unknown` | The request may have reached the terminal, but its financial result requires reconciliation. |
| `reversing` | A reversal request reached the terminal and is awaiting its final result. |

Do not resubmit an `unknown` payment until it has been reconciled. A retry could create a second financial transaction.

## Receipt Storage

Receipts are stored as bundles linked to a transaction or terminal service operation, not as one loose "last receipt" file.

That means each stored transaction can reference its own receipt bundle directly, and receipt lookups remain stable even after later transactions happen on the same terminal.

Use `receiptHandling: { "customer": "external", "merchant": "external" }` for ECR-managed receipts. `external` maps to OPI `Available`; the proxy stores the receipt and the ECR can retrieve it before printing. Local printing is an explicit choice for terminals with printers.

### How A Receipt Reaches Storage

1. A payment request resolves `receiptHandling` from the request, terminal, profile, and model configuration.
2. `external` tells the terminal that the ECR/proxy is available for that receipt copy; `local` tells a terminal with a printer to handle it locally.
3. For external handling, the terminal sends `Printer`, `JournalPrinter`, or `E-Journal` output over the OPI device callback channel.
4. The proxy associates the callback with the active operation and stores the receipt in the selected backend.
5. The ECR retrieves the stored bundle through the HTTP receipt endpoints.

Changing `receiptHandling` does not move existing receipts between backends. `STORAGE` determines whether newly captured content is written to JSON files or SQLite.

When external handling is selected for a terminal with a built-in printer, the ECR owns the customer-facing decision: display, send, print, or skip the captured customer copy.

### Reprint And Recovery

`POST /terminals/:terminalAlias/reprint` uses the same callback path: the terminal sends the last receipt bundle back as OPI device output (`Printer`, `JournalPrinter`, and `E-Journal` when available), and the proxy stores those copies under the created reprint transaction. The proxy sends `TicketReprint` even for printerless terminals such as RX5000, IM15, and IM30, because the terminal may still replay the last receipt bundle over the callback channel. That means reprint is not just a terminal-side physical print action; it is also the proxy-visible recovery path for `PrintLastTicket` situations and for cases where a terminal completed a payment but the ECR or proxy lost the original final response.

When a reprint succeeds while a previous card transaction is still `pending` or `unknown`, the proxy inspects the reprinted receipt evidence. If the receipt clearly matches the previous sale payment, including successful receipt result, approval/reference numbers, matching amount/currency, and a transaction time close to the original request, the proxy automatically reconciles the previous transaction to `succeeded` and records the reprint evidence in the transaction response. If that evidence is missing or does not match, the previous transaction remains `unknown` and must be reconciled manually.

For unattended IM15 and IM30 deployments, proxy-stored receipts are the default integration pattern because the terminals do not include printers. Merchants and integrators should retrieve stored receipts from the proxy and handle customer delivery outside the payment flow.

### Service Receipts

Activation, deactivation, configuration, initialisation, submission, and close-day operations can also return slips over the callback channel. These are stored under the terminal service operation rather than falsely attaching them to the latest payment. Terminal-level receipt endpoints include both transaction and service bundles.

## Receipt Endpoints

| Route | Use it for |
| --- | --- |
| `GET /transactions/:transactionId/receipts` | Receipts linked to one known transaction |
| `GET /terminals/:terminalAlias/receipts/latest` | Latest receipt-producing transaction or service operation on a terminal |
| `GET /terminals/:terminalAlias/receipts?limit=5` | Recent receipt bundles across transaction and service flows |

## Receipt Roles

| Role | Meaning |
| --- | --- |
| `customer` | Cardholder/customer copy |
| `merchant` | Merchant copy |
| `journal` | Terminal or ECR journal output |
| `other` | Output that cannot be mapped safely to another role |

The role is derived from the underlying OPI receipt target, not guessed from receipt text.

# API Overview

The proxy exposes a focused HTTP and JSON interface on top of OPI. The sections below explain response behavior and common request bodies. The bundled [`openapi.yaml`](../openapi.yaml) is the complete machine-readable contract for every request field, response schema, error, and client-generation workflow.

All routes except `/health` require bearer authentication.

## Health Route

| Route | Purpose |
| --- | --- |
| `GET /health` | Confirms that the proxy is running and returns the service version. |

## Payment Routes

| Route | Purpose |
| --- | --- |
| `POST /terminals/:terminalAlias/payment` | Starts a sale or pre-authorisation. |
| `POST /terminals/:terminalAlias/refund` | Starts a standalone terminal refund. |
| `POST /terminals/:terminalAlias/reversal` | Reverses the last terminal payment through `PaymentReversal`. |
| `POST /transactions/:transactionId/capture` | Captures an earlier stored pre-authorisation. |
| `POST /transactions/:transactionId/cancel` | Cancels an earlier stored pre-authorisation. |
| `POST /transactions/:transactionId/refund` | Refunds a specific stored transaction. |

Payment request examples, cashback, DCC, Force Acceptance, reversal, and abort behavior are explained below. Use the OpenAPI contract for the exhaustive field definitions.

## Receipt Routes

| Route | Purpose |
| --- | --- |
| `GET /transactions/:transactionId/receipts` | Returns receipts linked to one transaction. |
| `GET /terminals/:terminalAlias/receipts/latest` | Returns the latest transaction or service receipt bundle for a terminal. |
| `GET /terminals/:terminalAlias/receipts?limit=5` | Lists recent receipt bundles for a terminal. |
| `POST /terminals/:terminalAlias/reprint` | Requests the terminal's last receipt again. |
| `POST /terminals/:terminalAlias/repeat` | Synchronously asks the terminal to repeat its last card-service result for recovery. |

`/reprint` is the recovery step after `PrintLastTicket` or when a terminal may have completed a payment but the final response was lost. The proxy sends `TicketReprint` even for printerless terminals because the terminal can replay receipt output over the callback channel.

`/repeat` sends OPI `RepeatLastMessage`. It is always synchronous and does not create a new financial transaction. The proxy compares the terminal's `OriginalHeader.RequestID` with its latest financial transaction and uses `OriginalHeader.OverallResult` to verify or reconcile that transaction. The response returns the same complete `transaction` representation as a transaction read plus `transactionUpdated`: `true` means the stored transaction was changed, while `false` means it was verified as already current. Inconclusive results return an error instead of `false`.

## Terminal Management Routes

| Route | Purpose |
| --- | --- |
| `GET /terminals/:terminalAlias/ping` | Checks network reachability without opening an OPI session. |
| `GET /terminals/:terminalAlias/info` | Reads terminal identity, software, network, acquirer, and card-circuit information. |
| `GET /terminals/:terminalAlias/status` | Reads live OPI status and cached proxy state. |
| `POST /terminals/:terminalAlias/activate` | Opens the terminal session. |
| `POST /terminals/:terminalAlias/deactivate` | Closes the terminal session. |
| `POST /terminals/:terminalAlias/abort` | Cancels the currently pending card flow. |
| `POST /terminals/:terminalAlias/submit` | Submits completed terminal-side transactions. |
| `POST /terminals/:terminalAlias/close` | Performs final balance and logoff with `CloseDay`. |
| `POST /terminals/:terminalAlias/config` | Refreshes terminal configuration through `ContactTMS`. |
| `POST /terminals/:terminalAlias/init` | Refreshes merchant and acquirer initialisation through `ContactAcq`. |
| `POST /terminals/:terminalAlias/reset` | Restarts the terminal application. |

## Transaction Routes

| Route | Purpose |
| --- | --- |
| `GET /transactions/:transactionId` | returns the persisted source-of-truth transaction state |
| `PATCH /transactions/:transactionId` | manually reconciles a transaction after external verification; admin recovery only |

Follow-up transactions include `parentTransactionId`, which points to the original stored transaction. The `PATCH` route changes proxy state only; it does not perform terminal-side reconciliation.

Manual reconciliation accepts only `status`, `amount`, and `tip`, with at least one field required. Amounts must be positive. When a tip is supplied, `totalAmount` is derived from the reconciled amount plus tip; callers cannot patch `totalAmount` directly. Status is constrained by transaction type; for example, a sale can be reconciled to `succeeded`, `failed`, `declined`, `refunded`, or `reversed`. A conclusive status also clears the proxy's recovery cooldown and lock for that transaction.

```json
{
  "status": "succeeded",
  "amount": 12.50,
  "tip": 2.00
}
```

## Transaction Response Shape

Synchronous successes, errors, and transaction reads keep transaction facts under `transaction`. Request metadata such as `requestId`, `retryAfter`, and `error` stays at the top level.

- `amount` is the merchant-requested amount.
- `totalAmount` is included when the terminal returns a different final total.
- `tip` is the positive difference between `totalAmount` and `amount`.
- `terminalModel` is included when it is known from configuration.
- `forceAcceptance` is present only when the merchant requested it.
- `overallResult` is omitted when the public `status` already communicates a conclusive successful or aborted result.

### DCC

When the terminal offers Dynamic Currency Conversion, synchronous payment responses and `GET /transactions/:transactionId` include `transaction.dcc` immediately after the transaction currency:

```json
{
  "amount": "0.10",
  "currency": "CHF",
  "dcc": {
    "offered": true,
    "accepted": false,
    "amount": "2.24",
    "currency": "MXN",
    "exchangeRate": "22.3862",
    "markupPercentage": "3.9500"
  }
}
```

| Field | Meaning |
| --- | --- |
| `offered` | DCC was presented to the cardholder. The `dcc` object is omitted when DCC was not offered. |
| `accepted` | `true` only when the cardholder selected the offered billing currency; `false` when they chose the transaction currency. |
| `amount` | Amount offered in the cardholder billing currency. |
| `currency` | Cardholder billing currency name/code returned by the terminal, for example `MXN`. |
| `exchangeRate` | Terminal-reported DCC exchange rate when supplied. |
| `markupPercentage` | Terminal-reported DCC markup when supplied. |

DCC is configured on the terminal/acquirer side. It cannot be requested through the payment JSON body.

### Payment Failure Details

Payment error responses use vendor-neutral public codes and include `transaction.failure` when terminal or host reason details are available.

| Terminal result | Public code |
| --- | --- |
| Decline reason `1` | `AUTHORIZATION_RESPONSE_UNKNOWN` |
| Decline reason `569` or `579` | `PROVIDER_UNAVAILABLE` |
| Force Acceptance with a clear "function not supported" prompt | `FORCE_ACCEPTANCE_UNSUPPORTED` |

## Sync And Async Responses

Runtime fallback remains `RESPONSE_MODE=sync` for backward compatibility, but the shipped `.env.example` recommends `RESPONSE_MODE=async` for new deployments.

### Asynchronous Acceptance

`RESPONSE_MODE=async` returns HTTP `202` only after the proxy has connected to the terminal, written the OPI request, and waited through `ASYNC_ACCEPT_GRACE_MS` for an immediate rejection. A `202` means **accepted for processing**, not paid successfully.

If the terminal is offline, the request cannot be written, or a final `DeviceUnavailable` or `PrintLastTicket` arrives during the grace window, the route returns an HTTP error instead. Fresh terminal-unavailable results also create the short `RETRY_AFTER_DEVICE_UNAVAILABLE_MS` cooldown, during which requests return `503` plus `Retry-After`.

Payment-style async responses include `Location: /transactions/{transactionId}` and:

```json
{
  "transactionId": "trx_...",
  "status": "pending",
  "terminalAlias": "t1"
}
```

Service-style operations such as `/config`, `/init`, `/submit`, `/close`, `/reprint`, `/abort`, and `/reset` return HTTP `202` with `Preference-Applied: respond-async` and no response body. `/repeat` is recovery-only and always waits for the final repeated result.

Per request, send `"responseMode": "async"` in the JSON body or `Prefer: respond-async` to override the environment default. Accepted values are `sync` and `async`; invalid values return HTTP `400`.

### Polling And Final Status

Poll `GET /transactions/:transactionId` until the transaction reaches a final state.

| Status | Meaning | Merchant action |
| --- | --- | --- |
| `pending` | The OPI request reached the terminal and is still running. | Keep polling; do not start reprint merely because it takes longer than expected. |
| `succeeded`, `authorized`, `captured`, `refunded`, `reversed` | The requested financial operation completed. | Continue the normal merchant flow. |
| `declined`, `failed`, `aborted` | A final unsuccessful outcome is known. | Show the appropriate result; retry only when operationally safe. |
| `communication_error` | The proxy could not reach the terminal before sending the request. | Correct connectivity, then retry with idempotency as appropriate. |
| `unknown` | The request may have reached the terminal, but the financial result cannot be proven. | Reconcile externally before retrying or correcting with `PATCH`. |
| `reversing` | A reversal reached the terminal but is not final yet. | Keep polling the original transaction. |

`failed` includes terminal-level timeouts without payment-outcome evidence and lost-final-response cases where captured prompts clearly show an unsupported/declined card, invalid card, card-read failure, or terminal timeout. `aborted` is used when the terminal or cardholder clearly cancelled. `unknown` is deliberately conservative when financial evidence exists but the final outcome cannot be trusted.

If a card operation is rejected with `BUSY` because a previous transaction still needs reconciliation, the response includes `activeTransactionId` when known, `pendingStartedAt`, `pendingRecoveryAt`, and an actionable message that points to `POST /terminals/{terminalAlias}/reprint` and `PATCH /transactions/{transactionId}`. If a terminal operation lock is still active, the `BUSY` response includes `activeOperation`, lock timestamps, and `activeTransactionId` only when the active lock belongs to a transaction-style operation rather than an internal service request.

`/reprint` may reconcile a previous `pending` or `unknown` sale to `succeeded` only when the replayed receipt contains strong matching evidence: a successful purchase result, approval/reference data, matching amount and currency, and a nearby transaction time. Without that evidence, the original transaction remains unresolved and must be reconciled manually.

## Force Acceptance

Force Acceptance is only sent for payment requests. Refunds do not support Force Acceptance; the proxy does not include `ForceAcceptance` in refund OPI requests.

Use Force Acceptance only when the merchant and terminal setup is approved for Own Risk processing and online authorization is unavailable. The proxy maps the relevant OPI decline reasons as:

| Terminal decline reason | Meaning |
| --- | --- |
| `1` | System error |
| `569`, `579` | No connection to payment server |

The merchant is responsible for deciding how long to keep requesting Force Acceptance. They can process a controlled number of Force Acceptance payments and then disable it manually, or preferably switch back automatically once `GET /transactions/:transactionId` or a synchronous payment response returns `systemAvailable: true`.

## Terminal Reachability

`GET /terminals/:terminalAlias/ping` is a lightweight network reachability check. It sends four ICMP ping packets to the terminal IP from `config/terminals.json`, updates the proxy's cached `lastKnownReachable` flag, returns `204 No Content` when the terminal answers, and returns `503 Service Unavailable` with an empty body when it does not. This route does not open an OPI session and does not prove the terminal application is active; use `/status` when you need the terminal's OPI status payload.

## Idempotency

Mutating routes support the `Idempotency-Key` header.

- reuse the same key with the same request method, route, async preference, and JSON body to receive the same proxy response again
- reusing the same key with different request content returns HTTP `409`
- the proxy currently keeps idempotency records for about 24 hours

This is useful for client retries after network interruptions or uncertain caller-side timeouts.

## Payment Operations

### Payment Request

```json
{
  "reference": "order-100045",
  "amount": 24.90,
  "currency": "CHF",
  "responseMode": "async",
  "paymentMethods": ["visa", "mastercard", "twint"],
  "receiptHandling": {
    "customer": "external",
    "merchant": "external"
  }
}
```

`ClerkId` and `ShiftNumber` are optional and are omitted from OPI when not supplied. `paymentMethods` limits processing for the request; it does not change the terminal's displayed brands. Tips and DCC are terminal-controlled and cannot be requested through this JSON model.

### Cashback

Request cashback on the normal payment route by sending the total EFT amount as `amount` and the cash withdrawal as `cashback`. The proxy sends OPI `TotalAmount=amount` plus `CashBackAmount=cashback`. `cashback` must be less than or equal to `amount`.

```json
{
  "reference": "order-100046",
  "amount": 50.00,
  "cashback": 20.00,
  "currency": "CHF"
}
```

### Reversal

`POST /terminals/:terminalAlias/reversal` starts OPI `PaymentReversal` to reverse the last terminal transaction. On Swiss ep2 terminals the proxy does not send the original approval code. If a previous successful proxy transaction exists for that terminal, its amount is reused; otherwise the reversal can still be forwarded without a stored proxy transaction.

By default, reversal confirmation remains terminal-side. If the integration explicitly sends `"autoConfirm": true` in the reversal request body, the proxy automatically answers OPI `GetConfirmation` prompts for that `PaymentReversal`.

After the reversal request reaches the terminal, the original transaction changes to `reversing`. A successful result changes it to `reversed`; an explicit failure restores its previous status; and an uncertain result leaves it `unknown`. The reversal operation has its own transaction ID, linked through the lifecycle metadata.

### Abort

`POST /terminals/:terminalAlias/abort` sends OPI `AbortRequest` only while the proxy has an active pending card transaction. Without one, it returns `409 NO_ACTIVE_TRANSACTION` locally and does not contact the terminal. This protects terminals that can become unavailable after repeated standalone abort requests.

A confirmed abort returns the affected original transaction with `status: aborted`; it does not create a second financial transaction. If the terminal reports `DeviceUnavailable` while the original operation is still pending, the proxy returns `202` with `abortStatus: unconfirmed` and `activeTransactionId`. Continue polling the original transaction.

## Capability Overrides

Capability overrides are best placed inside the terminal `capabilities` block in `config/terminals.json`. Example:

```json
{
  "rx": {
    "terminalModel": "RX5000",
    "terminalProfile": "rx5000-default",
    "ip": "192.168.0.60",
    "capabilities": {
      "supportsReversal": false,
      "supportsCashback": false
    }
  }
}
```

Resolution order is terminal `capabilities` override first, then terminal profile, then terminal model.

## Unattended Terminals

For unattended OPI proxy integrations, the proxy automatically acknowledges the basic `DeliveryBox` callback with success. On validated IM30 proxy flows, the terminal sends `DeliveryBox`, the proxy answers success, and the terminal then continues with its own delivery-success messaging such as `Product delivery successful`. Merchants should treat delivery, partial-delivery, and reversal decisions as application concerns outside the payment flow, or use pre-authorisation where that is the better fit. IM15 and IM30 terminals do not have printers, so use proxy-stored receipts and retrieve them through the receipt endpoints.

For IM15 and IM30 in the current proxy scope:

- card payments are validated in proxy mode
- reversals are available in proxy mode
- refunds may still be unavailable depending on the unattended terminal configuration, in which case the proxy returns `UNSUPPORTED_OPERATION`
- Force Acceptance applies to payments only and is not supported for refunds

## Terminal And Service Notes

### Workstation And Callback Routing

`workstationId` is the OPI `WorkstationID`. Configure a stable, unique value for every checkout or terminal context; if omitted, the terminal alias is used. Multiple terminals can share one callback port when their workstation IDs are distinct. Callbacks are accepted only from the configured terminal IP for the active context.

If another interactive request is active for the same workstation, the proxy returns `409 BUSY` with active-operation details when available.

### Activation And Maintenance

`/activate` and `/deactivate` normally return early when cached state already matches the request. Send `{ "force": true }` to bypass the cached state. The proxy tries `ActivateTerminal`/`DeactivateTerminal` first and falls back to `Login`/`LogOff` for terminal variants that require the legacy request type.

| Route | OPI operation | Result notes |
| --- | --- | --- |
| `/submit` | `TransmitTrx` | submits completed terminal-side transactions; `TransmitTrx` is the underlying OPI request type |
| `/close` | `CloseDay` | performs final balance and logoff |
| `/config` | `ContactTMS` | returns parsed configuration/status details and supported-brand data when present |
| `/init` | `ContactAcq` | returns parsed merchant/acquirer initialisation details |

These service routes, plus activation/deactivation, can store terminal slips separately from transaction receipts. Their immediate responses contain receipt metadata rather than full receipt bodies. Retrieve stored content through the terminal receipt endpoints described in [Storage and Receipts](05-storage-and-receipts.md).

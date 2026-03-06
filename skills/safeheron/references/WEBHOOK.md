# Webhook Integration Reference

## Overview

Safeheron pushes real-time event notifications to your configured webhook URL.
Configure the webhook URL in: **Safeheron Web Console → Settings → API → Webhook**.

Webhook payloads use the same AES+RSA encryption scheme as API responses.

---

## Event Types

### Transaction Events
| Event Type | Trigger |
|------------|---------|
| `TRANSACTION_CREATED` | New transaction submitted |
| `TRANSACTION_STATUS_CHANGED` | Any status transition |
| `TRANSACTION_CONFIRMATION_COUNT` | Custom block confirmation count reached |

### MPC Sign Events
| Event Type | Trigger |
|------------|---------|
| `MPC_SIGN_CREATED` | New MPC sign request |
| `MPC_SIGN_STATUS_CHANGED` | Status transition |

### Web3 Events
| Event Type | Trigger |
|------------|---------|
| `WEB3_SIGN_CREATED` | New Web3 sign request |
| `WEB3_SIGN_STATUS_CHANGED` | Status transition |

### Whitelist Events
| Event Type | Trigger |
|------------|---------|
| `WHITELIST_ADDED` | New whitelist entry created |
| `WHITELIST_UPDATED` | Whitelist entry modified |
| `WHITELIST_REMOVED` | Whitelist entry deleted |

### Security Events
| Event Type | Trigger |
|------------|---------|
| `ILLEGAL_IP_REQUEST` | API called from non-whitelisted IP |
| `NO_MATCHING_TRANSACTION_POLICY` | Transaction has no matching policy |

### Other Events
| Event Type | Trigger |
|------------|---------|
| `GAS_BALANCE_WARNING` | Gas service balance below threshold |

---

## Webhook Handler Example (Spring Boot)

```java
@PostMapping("/safeheron/webhook")
public ResponseEntity<String> handleWebhook(@RequestBody String rawBody) {
    // 1. Decrypt payload using same method as API response decryption
    // 2. Parse event type and dispatch

    // Always respond with HTTP 200 quickly — do heavy work async
    return ResponseEntity.ok("OK");
}
```

---

## Re-push API

If your server missed events, you can re-request delivery:

```
POST /v1/webhook/transaction/repush   — re-push a specific transaction's events
POST /v1/webhook/pushFailed           — retry all previously failed webhook events
```

---

## Best Practices
- Return HTTP 200 as fast as possible; process events asynchronously (queue/message bus).
- Store raw payload in a database before processing, for auditability.
- Implement idempotency in your handler — Safeheron may retry on non-200 responses.
- Schedule a periodic call to `/v1/webhook/pushFailed` as a safety net.

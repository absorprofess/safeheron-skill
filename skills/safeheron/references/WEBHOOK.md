# Webhook Integration Reference

## Overview

Safeheron pushes real-time event notifications to your configured webhook URL. Configure the webhook URL in: Safeheron Web Console → Settings → API → Webhook.

## Webhook Request Verification

All webhook requests are signed by Safeheron. **Always verify the signature** before processing:

```java
@PostMapping("/safeheron/webhook")
public ResponseEntity<String> handleWebhook(
    @RequestBody String rawBody,
    @RequestHeader("X-Safeheron-Sig") String sig
) {
    // Verify: SHA256WithRSA using Safeheron's public key
    boolean valid = SafeheronWebhookVerifier.verify(rawBody, sig, safeheronPublicKey);
    if (!valid) {
        return ResponseEntity.status(403).body("Invalid signature");
    }
    
    WebhookEvent event = objectMapper.readValue(rawBody, WebhookEvent.class);
    handleEvent(event);
    return ResponseEntity.ok("OK");
}
```

Webhook payloads use the same AES+RSA encryption as API responses — decrypt `bizContent` using the same process as regular API responses.

---

## Event Types

### Transaction Events
| Event | Trigger |
|-------|---------|
| `TRANSACTION_CREATED` | New transaction submitted |
| `TRANSACTION_STATUS_CHANGED` | Any status transition |
| `TRANSACTION_CONFIRMATION_COUNT` | Custom block confirmation reached |

### MPC Sign Events
| Event | Trigger |
|-------|---------|
| `MPC_SIGN_CREATED` | New MPC sign request |
| `MPC_SIGN_STATUS_CHANGED` | Status transition |

### Web3 Events
| Event | Trigger |
|-------|---------|
| `WEB3_SIGN_CREATED` | New Web3 sign request |
| `WEB3_SIGN_STATUS_CHANGED` | Status transition |

### Whitelist Events
| Event | Trigger |
|-------|---------|
| `WHITELIST_ADDED` | New whitelist entry created |
| `WHITELIST_UPDATED` | Whitelist entry modified |
| `WHITELIST_REMOVED` | Whitelist entry deleted |

### Security Events
| Event | Trigger |
|-------|---------|
| `ILLEGAL_IP_REQUEST` | API called from non-whitelisted IP |
| `NO_MATCHING_TRANSACTION_POLICY` | Transaction has no matching policy |

### Other Events
| Event | Trigger |
|-------|---------|
| `GAS_BALANCE_WARNING` | Gas service balance below threshold |

---

## TransactionParam Object (Key Fields)
```json
{
  "txKey": "tx_safeheron_id",
  "customerRefId": "your_ref_id",
  "transactionStatus": "SUCCESS",
  "transactionSubStatus": "TRANSACTION_CONFIRMED",
  "sourceAccountKey": "account_key",
  "coinKey": "ETHEREUM_ETH",
  "txAmount": "0.05",
  "txFee": "0.001",
  "destinationAddress": "0x...",
  "blockHeight": 18234567,
  "txHash": "0x...",
  "createdTime": 1623038312000,
  "completedTime": 1623038350000
}
```

---

## Webhook Re-push API

If your server missed events, you can re-request them:

### Re-push Transaction Webhook Events
```
POST /v1/webhook/transaction/repush
```
```java
RepushWebhookRequest req = new RepushWebhookRequest();
req.setTxKey(txKey);
webhookService.repushTransaction(req);
```

### Push All Failed Webhook Events
```
POST /v1/webhook/pushFailed
```
Forces Safeheron to retry all events that your server did not acknowledge (non-200 response).

---

## Webhook Handler Pattern (Spring Boot)
```java
@Component
public class SafeheronWebhookHandler {
    
    @Autowired
    private TransactionService transactionService;
    
    public void handleEvent(WebhookEvent event) {
        switch (event.getType()) {
            case "TRANSACTION_STATUS_CHANGED":
                handleTxStatusChange(event.getParam());
                break;
            case "WHITELIST_ADDED":
                handleWhitelistAdded(event.getParam());
                break;
            default:
                log.warn("Unhandled webhook event type: {}", event.getType());
        }
    }
    
    private void handleTxStatusChange(TransactionParam param) {
        if ("SUCCESS".equals(param.getTransactionStatus())) {
            // Credit user, update order status, etc.
            orderService.confirmDeposit(param.getCustomerRefId(), param.getTxAmount());
        } else if ("FAILED".equals(param.getTransactionStatus())) {
            // Handle failure — refund, alert, etc.
            orderService.failWithdrawal(param.getCustomerRefId());
        }
    }
}
```

---

## Best Practices
- Respond with HTTP 200 quickly — do heavy processing asynchronously (e.g. via a queue).
- Store raw webhook payloads in a database before processing for auditability.
- Implement idempotency in your handler — Safeheron may retry events on non-200 responses.
- Set up the `/v1/webhook/pushFailed` call in a scheduled job as a safety net.

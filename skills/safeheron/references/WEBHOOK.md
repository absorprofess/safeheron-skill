# Webhook Integration Reference

## Overview

Safeheron pushes real-time event notifications to your configured webhook URL when transactions, MPC sign requests, Web3 sign requests, whitelist changes, or security events occur.

Configure the webhook URL in: **Safeheron Web Console → Settings → API → Webhook**

- Supported protocols: HTTP or HTTPS
- URL must start with `http://` or `https://`
- Note: Some public domains (ngrok, webhook.site) may be blocked by Safeheron's network filters

---

## Webhook Payload Structure

Webhook payloads use the **same AES+RSA encryption scheme** as API responses. The encrypted payload is a JSON body with these fields:

```json
{
  "timestamp": "1623038312088",
  "sig":        "<RSA-signature-base64>",
  "key":        "<RSA-encrypted-AES-key-base64>",
  "bizContent": "<AES-encrypted-payload-base64>",
  "rsaType":    "ECB_OAEP",
  "aesType":    "GCM_NOPADDING"
}
```

### Decryption Steps (same as API response decryption)

1. Build signature string: sort all fields by key ascending (exclude `rsaType`, `aesType`):
   `bizContent=...&code=...&key=...&timestamp=...`
2. Verify `sig` using **Safeheron's RSA public key** — reject if invalid.
3. Decrypt `key` field using **your RSA private key** → 48 bytes (AES key + IV).
4. Decrypt `bizContent` using AES/GCM/NoPadding → plaintext JSON event payload.

---

## Webhook Handler Example (Spring Boot)

```java
import com.safeheron.client.webhook.WebhookParam;

@PostMapping("/safeheron/webhook")
public ResponseEntity<String> handleWebhook(@RequestBody String rawBody) {
    try {
        // 1. Parse the outer envelope
        ObjectMapper mapper = new ObjectMapper();
        WebhookParam webhookParam = mapper.readValue(rawBody, WebhookParam.class);

        // 2. Verify signature and decrypt bizContent
        //    The SDK does NOT auto-decrypt webhooks — implement using your config:
        //    Step A: verify sig using safeheronRsaPublicKey
        //    Step B: decrypt 'key' field with your rsaPrivateKey → aesKey+iv (48 bytes)
        //    Step C: AES/GCM/NoPadding decrypt bizContent

        // 3. Parse decrypted payload
        String decryptedJson = decrypt(webhookParam, safeheronConfig);
        JsonNode event = mapper.readTree(decryptedJson);

        // 4. Route by event type — process asynchronously
        String txStatus = event.path("transactionStatus").asText();
        String txKey    = event.path("txKey").asText();
        processEventAsync(txKey, txStatus, event);

    } catch (Exception e) {
        log.error("Webhook processing error", e);
    }

    // Always respond HTTP 200 immediately — do NOT block
    return ResponseEntity.ok("OK");
}
```

---

## Event Types

### Transaction Events

| Event Type | Trigger |
|------------|---------|
| `TRANSACTION_CREATED` | New transaction submitted |
| `TRANSACTION_STATUS_CHANGED` | Any status transition |
| `TRANSACTION_CONFIRMATION_COUNT` | Custom block confirmation count reached |

**Transaction Event Payload (decrypted bizContent) Key Fields:**
| Field | Description |
|-------|-------------|
| `txKey` | Safeheron transaction key |
| `customerRefId` | Your reference ID |
| `txHash` | On-chain transaction hash |
| `transactionStatus` | Current status |
| `transactionSubStatus` | Sub-status |
| `coinKey` | Coin identifier |
| `txAmount` | Amount (string) |
| `sourceAccountKey` | Sender wallet |
| `sourceAddress` | Sender address |
| `destinationAddress` | Recipient address |
| `txFee` | Transaction fee paid |
| `blockHeight` | Confirmed block height |
| `createTime` | Unix timestamp (ms) |
| `completedTime` | Unix timestamp (ms) of completion |
| `customerExt1` | Custom field 1 |
| `customerExt2` | Custom field 2 |
| `amlLock` | AML status: `YES` / `NO` |

---

### MPC Sign Events

| Event Type | Trigger |
|------------|---------|
| `MPC_SIGN_CREATED` | New MPC sign request created |
| `MPC_SIGN_STATUS_CHANGED` | Status transition |

**MPC Sign Event Payload Key Fields:**
| Field | Description |
|-------|-------------|
| `txKey` | MPC sign request key |
| `customerRefId` | Your reference ID |
| `transactionStatus` | Status |
| `transactionSubStatus` | Sub-status |
| `sourceAccountKey` | Wallet used for signing |
| `signAlg` | `Secp256k1` or `Ed25519` |
| `dataList` | List of `{data, sig}` items (sig available on SUCCESS) |

---

### Web3 Sign Events

| Event Type | Trigger |
|------------|---------|
| `WEB3_SIGN_CREATED` | New Web3 sign request |
| `WEB3_SIGN_STATUS_CHANGED` | Status transition |

**Web3 Sign Event Payload Key Fields:**
| Field | Description |
|-------|-------------|
| `txKey` | Web3 sign request key |
| `customerRefId` | Your reference ID |
| `transactionStatus` | Status |
| `subjectType` | `ETH_SIGN`, `PERSONAL_SIGN`, `ETH_SIGNTYPEDDATA`, `ETH_SIGNTRANSACTION` |
| `accountKey` | Web3 wallet key |
| `sourceAddress` | Signing address |
| `message` / `messageHash` / `transaction` | Signed content (type-dependent) |

---

### Whitelist Events

| Event Type | Trigger |
|------------|---------|
| `WHITELIST_ADDED` | New whitelist entry created |
| `WHITELIST_UPDATED` | Whitelist entry modified |
| `WHITELIST_REMOVED` | Whitelist entry deleted |

---

### Security & System Events

| Event Type | Trigger |
|------------|---------|
| `ILLEGAL_IP_REQUEST` | API called from non-whitelisted IP |
| `NO_MATCHING_TRANSACTION_POLICY` | Transaction has no matching approval policy |
| `GAS_BALANCE_WARNING` | Gas station balance below threshold |

---

## Re-push API

If your server missed events or returned non-200, you can request re-delivery:

```java
// Re-push events for a specific transaction
TransactionApiService transactionApi = ServiceCreator.create(TransactionApiService.class, config);
// POST /v1/webhook/transaction/repush

// Retry all previously failed webhook events
// POST /v1/webhook/pushFailed
```

Safeheron retry schedule on non-200 response:
`30s → 1m → 5m → 1h → 12h → 24h` (7 total attempts, then stops)

---

## Custom Block Confirmation Count

Configure custom confirmation thresholds per blockchain in Safeheron Console. Once enabled, it applies to **both incoming and outgoing** transactions on that chain. A `TRANSACTION_CONFIRMATION_COUNT` event fires when the threshold is reached.

---

## Manual Webhook Trigger Rate Limit

Manual webhook re-triggers: **10 per minute** per team.

---

## Webhook URL Configuration Notes

- Modifying the webhook URL takes effect immediately — new events go to the new URL.
- Webhook only stops if the URL is **deleted** (not just modified).
- New/modify/delete webhook URLs require no admin approval — changes are instant.
- Webhook URL must be publicly accessible from Safeheron's IP range:
  `18.162.105.64`, `18.167.22.59`, `18.167.21.182`

---

## Webhook Filtering

- **No amount filtering** — any successful deposit, regardless of amount, triggers a webhook (including dust attacks / small transfers).
- All transaction types trigger webhooks: ordinary transfers, Auto Sweep (归集), gas refill (加油), speed-up, batch transfers.

---

## Configuration Change Behavior

- **Modifying** a webhook URL: events immediately route to the new URL. No approval required. Takes effect instantly.
- **Deleting** a webhook URL: webhook delivery stops completely.
- Adding/modifying/deleting Webhook URLs and RSA public keys **require no admin approval** — changes are immediate.
- Manual webhook re-trigger in Web Console requires **"Initiate Transaction" permission** and is rate-limited to **10 times/minute**.

---

## Best Practices

- Return HTTP 200 as fast as possible — offload all processing to a queue/async worker.
- Store the raw encrypted payload in a database before processing, for auditability.
- Implement idempotency by checking `txKey` before acting — Safeheron may deliver duplicates on retry.
- Schedule a periodic call to `/v1/webhook/pushFailed` as a safety net.
- Auto Sweep (归集), Gas refill, speed-up, and batch transfers all generate standard webhook events.

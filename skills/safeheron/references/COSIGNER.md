# API Co-Signer & Approval Callback

## What is the API Co-Signer?

The API Co-Signer is a service you deploy that **automatically approves or rejects transactions** based on your business logic, replacing manual approval workflows for programmatic use cases.

## How It Works
1. A transaction is submitted to Safeheron.
2. Safeheron calls your **Approval Callback Service** (a webhook endpoint you implement).
3. Your service evaluates the transaction and responds `APPROVE` or `REJECT`.
4. Safeheron proceeds accordingly.

---

## Approval Callback Service

### Security
Safeheron signs all callback requests. Verify using the same RSA signature method as webhook verification.

### Callback Request (from Safeheron to your server)

Key fields in the decrypted callback payload:
| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Safeheron transaction ID |
| `customerRefId` | String | Your reference ID |
| `txType` | String | `TRANSACTION` / `MPC_SIGN` / `WEB3_SIGN` |
| `txAmount` | String | Transaction amount |
| `coinKey` | String | Coin identifier |
| `destinationAddress` | String | Recipient address |
| `sourceAccountKey` | String | Sender wallet |

### Callback Response (your server to Safeheron)
| Field | Type | Description |
|-------|------|-------------|
| `approveStatus` | String | `APPROVE` or `REJECT` |
| `customerRefId` | String | Echo back the request's customerRefId |
| `rejectReason` | String | Optional reason when rejecting |

---

## Approval Logic Examples

### Whitelist-only destinations
```java
private boolean shouldApprove(CallbackRequest req) {
    if (!"TRANSACTION".equals(req.getTxType())) return true;
    Set<String> approved = loadApprovedAddresses(req.getCoinKey());
    return approved.contains(req.getDestinationAddress());
}
```

### Amount limit
```java
private boolean shouldApprove(CallbackRequest req) {
    BigDecimal amount = new BigDecimal(req.getTxAmount());
    BigDecimal limit = getTransactionLimit(req.getSourceAccountKey());
    return amount.compareTo(limit) <= 0;
}
```

---

## Best Practices
- Respond quickly and synchronously — Safeheron has a timeout.
- Log every decision with `txKey` and reason for audit purposes.
- Test with low-value transactions first.
- Handle errors gracefully — configure the default behavior in Safeheron Console.

## Official Callback Demo
See the SDK repository for a working callback demo:
https://github.com/Safeheron/safeheron-api-sdk-java

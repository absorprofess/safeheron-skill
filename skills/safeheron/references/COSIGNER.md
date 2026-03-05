# API Co-Signer & Approval Callback

## What is the API Co-Signer?

The API Co-Signer is a component you deploy alongside Safeheron that **automatically approves or rejects transactions** based on your business logic — without human intervention. It replaces manual approval workflows for programmatic use cases.

## How It Works
1. A transaction is submitted to Safeheron.
2. Instead of waiting for a human approver, Safeheron sends the transaction details to your **Approval Callback Service** (a webhook you implement).
3. Your service evaluates the transaction and responds with `APPROVE` or `REJECT`.
4. Safeheron proceeds accordingly.

## Approval Callback Service

### Security Authentication
Safeheron signs its callback requests. Verify using the same RSA signature method as webhook verification.

### Callback Endpoint
Your service must expose a POST endpoint that Safeheron calls:

```java
@PostMapping("/safeheron/approval-callback")
public ApprovalCallbackResponse handleApproval(@RequestBody String encryptedBody) {
    // 1. Decrypt and verify the request (same as API response decryption)
    CallbackRequest req = safeheronClient.decryptCallbackRequest(encryptedBody);
    
    // 2. Apply your business rules
    boolean shouldApprove = evaluateTransaction(req);
    
    // 3. Return encrypted response
    ApprovalCallbackResponse resp = new ApprovalCallbackResponse();
    resp.setApproveStatus(shouldApprove ? "APPROVE" : "REJECT");
    resp.setCustomerRefId(req.getCustomerRefId());
    return resp;
}
```

### CallbackV3 Request Data Fields
| Field | Type | Description |
|-------|------|-------------|
| txKey | String | Safeheron transaction ID |
| customerRefId | String | Your reference ID |
| txType | String | `TRANSACTION` \| `MPC_SIGN` \| `WEB3_SIGN` |
| txAmount | String | Transaction amount |
| coinKey | String | Coin identifier |
| destinationAddress | String | Recipient address |
| sourceAccountKey | String | Sender wallet account |

### Response Fields
| Field | Type | Description |
|-------|------|-------------|
| approveStatus | String | `APPROVE` or `REJECT` |
| customerRefId | String | Echo back the request's customerRefId |
| rejectReason | String | Optional reason for rejection |

---

## Approval Decision Examples

### Allow only whitelisted destinations
```java
private boolean evaluateTransaction(CallbackRequest req) {
    if (!"TRANSACTION".equals(req.getTxType())) return true;
    
    Set<String> whitelist = loadApprovedAddresses(req.getCoinKey());
    return whitelist.contains(req.getDestinationAddress());
}
```

### Amount-based approval
```java
private boolean evaluateTransaction(CallbackRequest req) {
    BigDecimal amount = new BigDecimal(req.getTxAmount());
    BigDecimal limit = getTransactionLimit(req.getSourceAccountKey());
    return amount.compareTo(limit) <= 0;
}
```

### Time-based restriction
```java
private boolean evaluateTransaction(CallbackRequest req) {
    LocalTime now = LocalTime.now(ZoneOffset.UTC);
    // Only allow transactions during business hours
    return now.isAfter(LocalTime.of(9, 0)) && now.isBefore(LocalTime.of(17, 0));
}
```

---

## Best Practices
- **Respond within the timeout** — implement fast, synchronous checks. Offload async work.
- **Log every decision** with txKey and rationale for audit purposes.
- **Start with APPROVE for all** and add restrictions incrementally.
- **Test with low-value transactions** first in a sandbox environment.
- **Handle errors gracefully** — if your callback throws, the default behavior is configurable in Safeheron Console.

## Callback Demo
The official callback demo is available in the SDK repository:
`/src/test/java/com/safeheron/demo/api/` — look for callback-related test classes.

# Transaction API Reference

## Imports
```java
import com.safeheron.client.api.TransactionApiService;
import com.safeheron.client.config.SafeheronConfig;
import com.safeheron.client.request.*;
import com.safeheron.client.response.*;
import com.safeheron.client.utils.ServiceCreator;
import com.safeheron.client.utils.ServiceExecutor;
```

## Create API Service
```java
TransactionApiService transactionApi = ServiceCreator.create(TransactionApiService.class, safeheronConfig);
```

---

## Create a Transaction
The exact request/response class names for transactions follow the same naming convention.
Consult the official source at:
https://github.com/Safeheron/safeheron-api-sdk-java/blob/main/src/test/java/com/safeheron/demo/api/transaction/TransactionTest.java

General pattern:
```java
// Request class names follow the API endpoint, e.g. CreateTransactionsRequest
// All calls follow: ServiceExecutor.execute(transactionApi.someMethod(request))
```

---

## Transaction Status Reference

### Main Statuses
| Status | Description |
|--------|-------------|
| SUBMITTED | Transaction received by Safeheron |
| WAIT_AUDIT | Pending policy/manual approval |
| WAIT_SIGN | Approved, pending MPC signing |
| BROADCASTING | Submitted to blockchain |
| PENDING | On-chain, awaiting confirmations |
| SUCCESS | Confirmed on-chain |
| FAILED | Transaction failed |
| REVOKED | Cancelled |
| REJECTED | Rejected by approver |

### Status Flow
```
SUBMITTED
    → WAIT_AUDIT (if policy requires approval)
    → WAIT_SIGN
    → BROADCASTING
    → PENDING
    → SUCCESS
           └─ FAILED
    ↓ (at WAIT_AUDIT or WAIT_SIGN)
    → REVOKED / REJECTED
```

---

## Key Rules
- Always set `customerRefId` (your unique ID) — Safeheron uses it for idempotency.
- All amounts are **strings** (e.g. `"0.05"`) — never floats.
- `txFeeLevel`: `"LOW"` | `"MIDDLE"` | `"HIGH"`
- `txKey` is Safeheron's transaction identifier — store it for status polling.

---

## Best Practices

### Withdrawal
1. Set `customerRefId` from your database record ID before calling create.
2. Store the returned `txKey`.
3. Monitor status via webhook (preferred) or polling.
4. Credit/debit only on `SUCCESS` status.

### Idempotency
If your call times out and you retry with the same `customerRefId`, Safeheron returns the original transaction instead of creating a duplicate.

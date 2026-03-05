# Transaction API Reference

## Create a Transaction (V3) — Recommended
```
POST /v3/transactions/create
```
```java
CreateTransactionV3Request req = new CreateTransactionV3Request();
req.setCustomerRefId(UUID.randomUUID().toString()); // Your unique idempotency key
req.setCustomerExt1("order-123");                   // Optional metadata
req.setCustomerExt2("user-456");
req.setSourceAccountKey("account_key_here");
req.setCoinKey("ETHEREUM_ETH");
req.setTxAmount("0.05");                             // String, not float!
req.setDestinationAddress("0xRecipientAddress");
req.setTxFeeLevel("MIDDLE");                         // LOW | MIDDLE | HIGH
// Optional for ERC-20:
req.setTokenAddress("0xContractAddress");

CreateTransactionV3Response resp = txService.createTransactionV3(req);
String txKey = resp.getTxKey();  // Safeheron transaction ID
```

### Fee Level Options
| Value | Description |
|-------|-------------|
| LOW   | Slow, cheap |
| MIDDLE | Balanced (recommended) |
| HIGH  | Fast, expensive |

---

## Retrieve a Transaction
```
POST /v1/transactions/one
```
```java
GetTransactionRequest req = new GetTransactionRequest();
req.setTxKey(txKey);
// OR use your reference:
req.setCustomerRefId("your-ref-id");

GetTransactionResponse resp = txService.getTransaction(req);
String status = resp.getTransactionStatus();
```

---

## Transaction List (V2)
```
POST /v2/transactions/list
```
```java
ListTransactionV2Request req = new ListTransactionV2Request();
req.setAccountKey(accountKey);
req.setTransactionStatus("SUCCESS");  // optional filter
req.setPage(1);
req.setPageSize(20);
req.setPageDirection("NEXT");         // NEXT | PREV

ListTransactionV2Response resp = txService.listTransactionsV2(req);
```

---

## Cancel Transaction
```
POST /v1/transactions/cancel
```
```java
CancelTransactionRequest req = new CancelTransactionRequest();
req.setTxKey(txKey);
txService.cancelTransaction(req);
```
Only transactions in `WAIT_AUDIT` or `WAIT_SIGN` status can be cancelled.

---

## Estimate Transaction Fee
```
POST /v1/transactions/fee
```
```java
EstimateFeeRequest req = new EstimateFeeRequest();
req.setSourceAccountKey(accountKey);
req.setCoinKey("ETHEREUM_ETH");
req.setTxAmount("0.05");
req.setDestinationAddress("0x...");

EstimateFeeResponse resp = txService.estimateFee(req);
String feeAmount = resp.getFeeAmount();
String feeCoinKey = resp.getFeeCoinKey();
```

---

## Retrieve Transaction Approval Details
```
POST /v1/transactions/approvalDetail
```

## Speed Up Transactions (EVM chains)
```
POST /v1/transactions/speedUp
```

## UTXO Batch Transaction
```
POST /v1/transactions/utxo/batch
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

### Substatus (Key Values)
| Substatus | Meaning |
|-----------|---------|
| TRANSACTION_PENDING_APPROVAL | In approval queue |
| TRANSACTION_APPROVING | Being reviewed |
| TRANSACTION_APPROVED | Approved, will sign |
| TRANSACTION_MPC_SIGNING | MPC signing in progress |
| TRANSACTION_SIGNED | Signed, ready to broadcast |

---

## Best Practices

### Deposit Monitoring
1. Use webhook `Transaction Status Change Event` to receive real-time updates.
2. Poll `GET /v1/transactions/one` with exponential backoff as fallback.
3. Credit user accounts only on `SUCCESS` status.
4. For high-value assets, wait for additional confirmations.

### Withdrawal Flow
1. Create transaction → store `txKey` and your `customerRefId`.
2. Use `customerRefId` for idempotency — same ref ID won't create duplicate transactions.
3. Monitor status via webhook.
4. Expose `txKey` (blockchain transaction hash equivalent) to end users.

### Idempotency
```java
// Always set customerRefId from your own database record ID:
req.setCustomerRefId(withdrawal.getId().toString());
// If the API call times out and you retry, Safeheron returns the original transaction
// instead of creating a duplicate.
```

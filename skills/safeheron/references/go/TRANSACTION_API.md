# Transaction API Reference

## Imports

```go
import (
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron"
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron/api"
    "github.com/google/uuid"
)
```

## Create API Instance

```go
transactionApi := api.TransactionApi{Client: sc}
```

---

## Create a Transaction

```go
req := api.CreateTransactionsRequest{
    CustomerRefId:          uuid.New().String(), // your unique ID
    CoinKey:                "ETHEREUM_ETH",
    TxAmount:               "0.01",              // string, never float
    SourceAccountKey:       accountKey,
    SourceAccountType:      "VAULT_ACCOUNT",
    DestinationAccountType: "ONE_TIME_ADDRESS",
    DestinationAddress:     "0xRecipientAddress",
    TxFeeLevel:             "MIDDLE", // "LOW" | "MIDDLE" | "HIGH"
}

var resp api.TxKeyResult
if err := transactionApi.CreateTransactions(req, &resp); err != nil {
    panic(fmt.Errorf("failed to create transaction: %w", err))
}

txKey := resp.TxKey // save this -- Safeheron transaction identifier
fmt.Printf("Transaction created, txKey: %s\n", txKey)
```

### CreateTransactionsRequest Key Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `CustomerRefId` | Yes | `string` | Your unique ID for idempotency (max 100 chars) |
| `CoinKey` | Yes | `string` | Coin to transfer (e.g. `ETHEREUM_ETH`) |
| `TxAmount` | Yes | `string` | Amount as string -- never use float |
| `SourceAccountKey` | Yes | `string` | Sender wallet key |
| `SourceAccountType` | Yes | `string` | `VAULT_ACCOUNT` |
| `DestinationAccountType` | Yes | `string` | `ONE_TIME_ADDRESS`, `VAULT_ACCOUNT`, or `WHITELISTING_ACCOUNT` |
| `DestinationAddress` | Cond. | `string` | Required when `DestinationAccountType=ONE_TIME_ADDRESS` |
| `DestinationAccountKey` | Cond. | `string` | Required when type is `VAULT_ACCOUNT` (accountKey) or `WHITELISTING_ACCOUNT` (whitelistKey) |
| `TxFeeLevel` | No | `string` | `LOW` / `MIDDLE` / `HIGH` |
| `Note` | No | `string` | Transaction note (max 180 chars) |
| `CustomerExt1` | No | `string` | Custom field 1 (max 255 chars) |
| `CustomerExt2` | No | `string` | Custom field 2 (max 255 chars) |

---

## Retrieve a Single Transaction

```go
req := api.OneTransactionsRequest{
    TxKey: txKey,
    // OR: CustomerRefId: customerRefId,
}

var resp api.OneTransactionsResponse
if err := transactionApi.OneTransactions(req, &resp); err != nil {
    panic(fmt.Errorf("failed to get transaction: %w", err))
}

fmt.Printf("Status: %s\n", resp.TransactionStatus)
fmt.Printf("Sub-status: %s\n", resp.TransactionSubStatus)
fmt.Printf("TxHash: %s\n", resp.TxHash)
```

---

## List Transactions (V2 -- Cursor-based)

```go
req := api.ListTransactionsV2Request{
    Limit:             20,
    TransactionStatus: "SUCCESS",     // optional filter
    CoinKey:           "ETHEREUM_ETH", // optional filter
}

var txList []api.TransactionsResponse
if err := transactionApi.ListTransactionsV2(req, &txList); err != nil {
    panic(fmt.Errorf("failed to list transactions: %w", err))
}

for _, tx := range txList {
    fmt.Printf("%s %s\n", tx.TxKey, tx.TransactionStatus)
}
```

---

## Estimate Transaction Fee

```go
req := api.TransactionsFeeRateRequest{
    CoinKey:            "ETHEREUM_ETH",
    Value:              "0.01",
    SourceAccountKey:   accountKey,
    DestinationAddress: "0xRecipient",
}

var resp api.TransactionsFeeRateResponse
if err := transactionApi.TransactionFeeRate(req, &resp); err != nil {
    panic(fmt.Errorf("failed to estimate fee: %w", err))
}

fmt.Printf("Low: %s, Middle: %s, High: %s\n",
    resp.LowFeeRate, resp.MiddleFeeRate, resp.HighFeeRate)
```

---

## Cancel a Transaction

```go
req := api.CancelTransactionRequest{
    TxKey: txKey,
}

var resp api.ResultResponse
if err := transactionApi.CancelTransactions(req, &resp); err != nil {
    panic(fmt.Errorf("failed to cancel transaction: %w", err))
}
```

> Only transactions in `WAIT_AUDIT` or `WAIT_SIGN` status can be cancelled.

---

## Speed Up a Transaction (Replace-by-Fee)

```go
req := api.RecreateTransactionRequest{
    TxKey:      originalTxKey,
    TxFeeLevel: "HIGH",
}

var resp api.TxKeyResult
if err := transactionApi.RecreateTransactions(req, &resp); err != nil {
    panic(fmt.Errorf("failed to speed up transaction: %w", err))
}

newTxKey := resp.TxKey
```

---

## Transaction Status Reference

| Status | Description |
|--------|-------------|
| `SUBMITTED` | Received by Safeheron |
| `WAIT_AUDIT` | Pending policy / manual approval |
| `WAIT_SIGN` | Approved, pending MPC signing |
| `BROADCASTING` | Submitted to blockchain |
| `PENDING` | On-chain, awaiting confirmations |
| `SUCCESS` | Confirmed on-chain |
| `FAILED` | Transaction failed |
| `REJECTED` | Rejected by approver |

### Status Flow

```
SUBMITTED
    -> WAIT_AUDIT
    -> WAIT_SIGN
    -> BROADCASTING
    -> PENDING
    -> SUCCESS
    |-- FAILED

    At WAIT_AUDIT or WAIT_SIGN:
    -> REJECTED (denied by approver)
```

---

## Chain-Specific Transaction Behaviors

### Speed-Up Support

| Category | Supported Chains |
|----------|-----------------|
| UTXO chains | BTC, BCH, DASH, LTC, DOGE |
| Non-UTXO chains | All EVM chains, FIL, Aptos, CFX |
| NOT supported | NEAR, SUI, TRON, SOL, TON |

### Chain-Specific Notes

- **Solana**: Minimum rent-exempt balance ~0.002 SOL cannot be transferred.
- **TRON**: Fee has no LOW/MID/HIGH tiers; `sourceAddress` required for fee estimation.
- **Contract addresses**: API transfers to contract addresses require `FailOnContract = false`.
- **destinationAccountKey rules**:
  - `VAULT_ACCOUNT` -> pass `accountKey`
  - `WHITELISTING_ACCOUNT` -> pass `whitelistKey`
  - `ONE_TIME_ADDRESS` -> do NOT pass `DestinationAccountKey`; use `DestinationAddress`

---

## Best Practices

1. Always set `CustomerRefId` from your own DB record ID before calling create.
2. Store the returned `TxKey` -- it is Safeheron's unique identifier.
3. Monitor status via **Webhook** (preferred) and call `/v1/transactions/one` periodically as fallback.
4. Credit/debit accounts only on `SUCCESS` status.
5. All amounts are **strings** (e.g. `"0.05"`) -- never use float.
6. On timeout, **retry with the same `CustomerRefId`** -- Safeheron returns the original tx (idempotency). Error code `9001` means the refId already exists.

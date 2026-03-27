# Wallet Account API Reference

## Imports

```go
import (
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron"
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron/api"
)
```

## Create API Instance

```go
accountApi := api.AccountApi{Client: sc}
```

---

## Create a Wallet Account

```go
req := api.CreateAccountRequest{
    AccountName: "my-wallet-account",
    HiddenOnUI:  false, // true = API-only wallet, hidden in Console
}

var resp api.CreateAccountResponse
if err := accountApi.CreateAccount(req, &resp); err != nil {
    panic(fmt.Errorf("failed to create wallet account: %w", err))
}
accountKey := resp.AccountKey // save this -- permanent wallet identifier
```

**CreateAccountRequest Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `AccountName` | Yes | `string` | Wallet display name |
| `HiddenOnUI` | No | `bool` | If true, wallet is hidden in Web Console / App |

**CreateAccountResponse Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `AccountKey` | `string` | Unique wallet identifier (save permanently) |
| `PubKeys` | `[]struct{SignAlg, PubKey}` | Account public key list |
| `CoinAddressList` | `[]struct{...}` | Coin address list per coin |

---

## List Wallet Accounts

```go
req := api.ListAccountRequest{
    PageNumber: 1,
    PageSize:   10,
}

var res api.ListAccountResponse
if err := accountApi.ListAccounts(req, &res); err != nil {
    panic(fmt.Errorf("failed to list accounts: %w", err))
}

for _, acct := range res.Content {
    fmt.Printf("%s %s\n", acct.AccountKey, acct.AccountName)
}
```

**ListAccountRequest Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `PageSize` | No | `int` | Items per page (default 10) |
| `PageNumber` | No | `int` | Page number, 1-indexed |

**AccountResponse Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `AccountKey` | `string` | Unique wallet identifier |
| `AccountName` | `string` | Wallet display name |
| `AccountIndex` | `int32` | Derivation path index |
| `AccountTag` | `string` | Tag: `DEPOSIT`, `NONE`, etc. |
| `HiddenOnUI` | `bool` | Hidden from UI |
| `AccountType` | `string` | `VAULT_ACCOUNT` or `WEB3_ACCOUNT` |

---

## Get a Single Wallet Account

```go
req := api.OneAccountRequest{
    AccountKey: accountKey,
}

var resp api.AccountResponse
if err := accountApi.OneAccounts(req, &resp); err != nil {
    panic(fmt.Errorf("failed to get account: %w", err))
}
```

> The method is `OneAccounts` (with 's'), matching the API endpoint convention.

---

## Batch Label Wallet Accounts

```go
req := api.BatchUpdateAccountTagRequest{
    AccountKeyList: []string{"your-account-key"},
    AccountTag:     "DEPOSIT", // or "NONE" to remove
}

var resp api.ResultResponse
if err := accountApi.BatchUpdateAccountTag(req, &resp); err != nil {
    panic(fmt.Errorf("failed to update account tag: %w", err))
}
```

**AccountTag Values:**

| Value | Description |
|-------|-------------|
| `DEPOSIT` | Mark as deposit wallet -- eligible for Auto Sweep |
| `NONE` | Remove DEPOSIT label |

---

## Add Coin to a Wallet Account

```go
req := api.AddCoinRequest{
    AccountKey: accountKey,
    CoinKey:    "ETH_GOERLI",
}

var resp api.AddCoinResponse
if err := accountApi.AddCoin(req, &resp); err != nil {
    panic(fmt.Errorf("failed to add coin: %w", err))
}

fmt.Printf("Address: %s\n", resp[0].Address)
```

**Behavior Notes:**
- Adding a token (e.g. USDT ERC-20) will also add ETH automatically.
- Re-adding an already-present coinKey returns the same result (idempotent).
- A coin manually disabled in the UI cannot be re-enabled via API.

---

## List Coins on a Wallet Account

```go
req := api.ListAccountCoinRequest{
    AccountKey: accountKey,
}

var coins []api.AccountCoinResponse
if err := accountApi.ListAccountCoin(req, &coins); err != nil {
    panic(fmt.Errorf("failed to list account coins: %w", err))
}

for _, coin := range coins {
    fmt.Printf("%-30s balance: %-20s\n", coin.CoinKey, coin.Balance)
}
```

---

## Best Practices

- `PageSize` and `PageNumber` are `int` type in Go (not `Long` like Java).
- `accountKey` is the permanent, immutable identifier for a wallet -- store it after creation.
- Web3 wallets (`AccountType = "WEB3_ACCOUNT"`) use a separate set of APIs -- see [WEB3_API.md](WEB3_API.md).
- `coinKey` format examples: `ETHEREUM_ETH`, `BITCOIN_BTC`, `USDT(ERC20)_ETHEREUM_USDT`, `ETH_SEPOLIA`.

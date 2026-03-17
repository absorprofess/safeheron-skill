# Wallet Account API Reference

## Imports

```java
import com.safeheron.client.api.AccountApiService;
import com.safeheron.client.config.SafeheronConfig;
import com.safeheron.client.request.*;
import com.safeheron.client.response.*;
import com.safeheron.client.utils.ServiceCreator;
import com.safeheron.client.utils.ServiceExecutor;
```

## Create API Service

```java
AccountApiService accountApi = ServiceCreator.create(AccountApiService.class, safeheronConfig);
```

---

## Create a Wallet Account

```java
CreateAccountRequest req = new CreateAccountRequest();
req.setAccountName("my-wallet-account");
req.setHiddenOnUI(false);  // true = API-only wallet, hidden in Console

CreateAccountResponse resp = ServiceExecutor.execute(accountApi.createAccount(req));
String accountKey = resp.getAccountKey();  // save this — permanent wallet identifier
```

**CreateAccountRequest Fields:**
| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `accountName` | Yes | String | Wallet display name |
| `hiddenOnUI` | No | Boolean | If true, wallet is hidden in Web Console / App |

**CreateAccountResponse Fields:**
| Field | Description |
|-------|-------------|
| `accountKey` | Unique wallet identifier (save permanently) |
| `accountName` | Wallet name |

---

## List Wallet Accounts

```java
ListAccountRequest req = new ListAccountRequest();
req.setPageSize(10L);   // Long, not int
req.setPageNumber(1L);  // Long, not int — 1-indexed

PageResult<AccountResponse> result = ServiceExecutor.execute(accountApi.listAccounts(req));
List<AccountResponse> accounts = result.getContent();
long total = result.getTotalElements();

for (AccountResponse acct : accounts) {
    System.out.println(acct.getAccountKey() + " " + acct.getAccountName());
}
```

**ListAccountRequest Fields:**
| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `pageSize` | No | Long | Items per page (default 10) |
| `pageNumber` | No | Long | Page number, 1-indexed |

**AccountResponse Key Fields:**
| Field | Description |
|-------|-------------|
| `accountKey` | Unique wallet identifier |
| `accountName` | Wallet display name |
| `accountIndex` | Derivation path index |
| `accountTag` | Tag: `DEPOSIT`, `NONE`, etc. |
| `hiddenOnUI` | Boolean |
| `accountType` | `VAULT_ACCOUNT` or `WEB3_ACCOUNT` |

---

## Get a Single Wallet Account

```java
OneAccountRequest req = new OneAccountRequest();
req.setAccountKey(accountKey);

AccountResponse resp = ServiceExecutor.execute(accountApi.oneAccount(req));
```

---

## Update a Wallet Account

```java
UpdateAccountRequest req = new UpdateAccountRequest();
req.setAccountKey(accountKey);
req.setAccountName("new-name");      // optional
req.setHiddenOnUI(true);             // optional
req.setAccountTag("DEPOSIT");        // optional: "DEPOSIT" or "NONE"

ServiceExecutor.execute(accountApi.updateAccount(req));
```

**AccountTag Values:**
| Value | Description |
|-------|-------------|
| `DEPOSIT` | Mark as deposit wallet — eligible for Auto Sweep |
| `NONE` | Remove DEPOSIT label |

> Cancelling `DEPOSIT` label: set `accountTag = "NONE"`. Can be re-applied later.

---

## Add Coin to a Wallet Account (V2 — Recommended)

Add up to 20 coins in a single call. Adding a token auto-adds its mainnet coin.

```java
CreateAccountCoinV2Request req = new CreateAccountCoinV2Request();
req.setAccountKey(accountKey);
req.setCoinKeyList(Arrays.asList(
    "ETHEREUM_ETH",
    "USDT(ERC20)_ETHEREUM_USDT",
    "BITCOIN_BTC"
));

List<CreateAccountCoinResponse> respList = ServiceExecutor.execute(accountApi.createAccountCoinV2(req));
for (CreateAccountCoinResponse coin : respList) {
    System.out.println(coin.getCoinKey() + " address: " + coin.getAddress());
}
```

> **V1 (single coin):** `CreateAccountCoinRequest` with `setCoinKey(String)` — still works.

**Behavior Notes:**
- Adding a token (e.g. USDT ERC-20) will also add ETH automatically.
- Re-adding an already-present coinKey returns the same result (idempotent).
- A coin manually disabled in the UI cannot be re-enabled via API.

---

## List Coins on a Wallet Account

Query balances and deposit addresses per coin for a specific account:

```java
ListAccountCoinRequest req = new ListAccountCoinRequest();
req.setAccountKey(accountKey);

List<AccountCoinResponse> coins = ServiceExecutor.execute(accountApi.listAccountCoin(req));
for (AccountCoinResponse coin : coins) {
    System.out.printf("%-30s balance: %-20s address: %s%n",
        coin.getCoinKey(), coin.getBalance(), coin.getAddress());
}
```

> Also queryable via: `GET /v1/account/coin/list`

**AccountCoinResponse Key Fields:**
| Field | Description |
|-------|-------------|
| `coinKey` | Coin identifier |
| `address` | Deposit address for this coin |
| `balance` | Available balance (string) |
| `frozenBalance` | Balance locked in pending transactions |
| `totalBalance` | balance + frozenBalance |

---

## Request / Response Class Summary

| Operation | Request Class | Response Class |
|-----------|--------------|----------------|
| Create account | `CreateAccountRequest` | `CreateAccountResponse` |
| List accounts | `ListAccountRequest` | `PageResult<AccountResponse>` |
| Get one account | `OneAccountRequest` | `AccountResponse` |
| Update account | `UpdateAccountRequest` | *(void / ResultResponse)* |
| Add coin V2 | `CreateAccountCoinV2Request` | `List<CreateAccountCoinResponse>` |
| Add coin V1 | `CreateAccountCoinRequest` | `List<CreateAccountCoinResponse>` |
| List coins | `ListAccountCoinRequest` | `List<AccountCoinResponse>` |

---

## Notes

- `pageSize` and `pageNumber` fields are **Long** type, not int.
- `accountKey` is the permanent, immutable identifier for a wallet — store it after creation.
- Web3 wallets (`accountType = WEB3_ACCOUNT`) use a separate set of APIs — see [WEB3_API.md](WEB3_API.md).
- `coinKey` format examples: `ETHEREUM_ETH`, `BITCOIN_BTC`, `USDT(ERC20)_ETHEREUM_USDT`, `ETH_SEPOLIA`.

# Wallet Account & Coin API Reference

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
req.setAccountName("first-wallet-account");
req.setHiddenOnUI(false);

CreateAccountResponse resp = ServiceExecutor.execute(accountApi.createAccount(req));
String accountKey = resp.getAccountKey();  // save this — it's the unique wallet ID
```

## Add a Coin to a Wallet Account
```java
// Request class: CreateAccountCoinRequest (not "AddCoinRequest")
// Response class: CreateAccountCoinResponse (List)
CreateAccountCoinRequest req = new CreateAccountCoinRequest();
req.setAccountKey(accountKey);
req.setCoinKey("ETH_GOERLI");  // or "ETHEREUM_ETH", "BITCOIN_BTC", etc.

List<CreateAccountCoinResponse> respList = ServiceExecutor.execute(accountApi.createAccountCoin(req));
String address = respList.get(0).getAddress();
```

## List Wallet Accounts
```java
ListAccountRequest req = new ListAccountRequest();
req.setPageSize(10L);   // Long, not int
req.setPageNumber(1L);  // Long, not int

PageResult<AccountResponse> result = ServiceExecutor.execute(accountApi.listAccounts(req));
List<AccountResponse> accounts = result.getContent();
```

---

## Request / Response Class Summary

| Operation | Request Class | Response Class |
|-----------|--------------|----------------|
| Create account | `CreateAccountRequest` | `CreateAccountResponse` |
| Add coin | `CreateAccountCoinRequest` | `List<CreateAccountCoinResponse>` |
| List accounts | `ListAccountRequest` | `PageResult<AccountResponse>` |

---

## Notes
- `pageSize` and `pageNumber` fields in `ListAccountRequest` are **Long** type.
- `coinKey` format examples: `ETHEREUM_ETH`, `BITCOIN_BTC`, `ETH_GOERLI`, `ETH(SEPOLIA)_ETHEREUM_SEPOLIA`.
- The response from `createAccountCoin` is a **List** — always access via `.get(0)` for the first address.
- `accountKey` is the permanent identifier for a wallet — store it after creation.

# Wallet Account & Coin API Reference

## Wallet Account Endpoints

### List Wallet Accounts
```
POST /v1/account/list
```
```java
ListAccountRequest req = new ListAccountRequest();
req.setPage(1);
req.setPageSize(10);
ListAccountResponse resp = accountService.listAccounts(req);
List<AccountBean> accounts = resp.getAccountList();
```

### Create a Wallet Account
```
POST /v1/account/create
```
```java
CreateAccountRequest req = new CreateAccountRequest();
req.setAccountName("Hot Wallet");
// Optional: req.setHiddenOnUI(false);
CreateAccountResponse resp = accountService.createAccount(req);
String accountKey = resp.getAccountKey();
```

### Batch Create Wallet Accounts (V2)
```
POST /v2/account/create/batch
```
```java
BatchCreateAccountV2Request req = new BatchCreateAccountV2Request();
req.setCount(5);
req.setAccountNamePrefix("Wallet");  // Creates: Wallet-1, Wallet-2 ...
BatchCreateAccountV2Response resp = accountService.batchCreateAccountV2(req);
```

### Retrieve a Single Wallet Account
```
POST /v1/account/one
```
```java
GetAccountRequest req = new GetAccountRequest();
req.setAccountKey("your_account_key");
GetAccountResponse resp = accountService.getAccount(req);
```

### Query Wallet Account by Address
```
POST /v1/account/queryAccountByAddress
```
```java
QueryAccountByAddressRequest req = new QueryAccountByAddressRequest();
req.setAddress("0x...");
req.setCoinKey("ETHEREUM_ETH");
QueryAccountByAddressResponse resp = accountService.queryAccountByAddress(req);
```

### Add Coins to a Wallet Account (V2)
```
POST /v2/account/coin/create
```
```java
AddCoinV2Request req = new AddCoinV2Request();
req.setAccountKey(accountKey);
req.setCoinKeyList(Arrays.asList("ETHEREUM_ETH", "BITCOIN_BTC"));
AddCoinV2Response resp = accountService.addCoinV2(req);
```

### List Coins Within a Wallet Account
```
POST /v1/account/coin/list
```
```java
ListCoinInAccountRequest req = new ListCoinInAccountRequest();
req.setAccountKey(accountKey);
ListCoinInAccountResponse resp = accountService.listCoinsInAccount(req);
```

### Retrieve Coin Balance
```
POST /v1/account/coin/balance
```
```java
GetCoinBalanceRequest req = new GetCoinBalanceRequest();
req.setAccountKey(accountKey);
req.setCoinKey("ETHEREUM_ETH");
GetCoinBalanceResponse resp = accountService.getCoinBalance(req);
BigDecimal balance = new BigDecimal(resp.getBalance());
```

### Retrieve The Balance of an Address
```
POST /v1/account/address/balance
```

### List Coin Address Group of a Wallet Account
```
POST /v1/account/coin/address/list
```

---

## Coin Endpoints

### Coin List (All Supported Coins)
```
POST /v1/coin/list
```
```java
CoinListRequest req = new CoinListRequest();
req.setCoinKey("ETHEREUM_ETH");  // optional filter
CoinListResponse resp = coinService.listCoins(req);
```

### Verify Coin Address
```
POST /v1/coin/address/verify
```
```java
VerifyAddressRequest req = new VerifyAddressRequest();
req.setCoinKey("ETHEREUM_ETH");
req.setAddress("0x...");
VerifyAddressResponse resp = coinService.verifyAddress(req);
boolean isValid = resp.isValid();
```

### Retrieve Current Block Height
```
POST /v1/coin/block/height
```

### Snapshot the Coin Balance
```
POST /v1/account/coin/balance/snapshot
```

---

## AccountBean Fields (Key Properties)
| Field | Type | Description |
|-------|------|-------------|
| accountKey | String | Unique account identifier |
| accountName | String | Display name |
| accountType | String | See Account Type data dictionary |
| accountTag | String | Account tag |
| hiddenOnUI | Boolean | Whether hidden in the App |
| coinList | List | Coins added to this account |

---

## Tips
- `accountKey` is the primary identifier — store it after creation.
- `coinKey` format: `{NETWORK}_{SYMBOL}` e.g. `ETHEREUM_ETH`, `BITCOIN_BTC`, `BSC_BNB`.
- Balances are always **strings** — parse with `new BigDecimal(balanceStr)`.
- Use `page` + `pageSize` for pagination; max `pageSize` is typically 100.

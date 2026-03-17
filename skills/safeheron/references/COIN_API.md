# Coin Management API Reference

## Overview

Coin management covers two distinct service interfaces:
- `CoinApiService` — global coin/network info, address validation, balance snapshots
- `AccountApiService` — per-account coin operations (add coin, list coins on an account)

---

## Imports

```java
import com.safeheron.client.api.CoinApiService;
import com.safeheron.client.api.AccountApiService;
import com.safeheron.client.config.SafeheronConfig;
import com.safeheron.client.request.*;
import com.safeheron.client.response.*;
import com.safeheron.client.utils.ServiceCreator;
import com.safeheron.client.utils.ServiceExecutor;
```

## Create API Services

```java
CoinApiService    coinApi    = ServiceCreator.create(CoinApiService.class,    safeheronConfig);
AccountApiService accountApi = ServiceCreator.create(AccountApiService.class, safeheronConfig);
```

---

## CoinApiService Operations

### List All Supported Coins

```java
// No request parameters required — pass null or empty request
List<CoinResponse> coins = ServiceExecutor.execute(coinApi.listCoin());
for (CoinResponse coin : coins) {
    System.out.println(coin.getCoinKey() + " / " + coin.getCoinFullName());
}
```

**CoinResponse Key Fields:**
| Field | Type | Description |
|-------|------|-------------|
| `coinKey` | String | Unique coin identifier (e.g. `ETHEREUM_ETH`, `BITCOIN_BTC`) |
| `coinFullName` | String | Full name (e.g. "Ethereum") |
| `coinName` | String | Symbol (e.g. "ETH") |
| `coinDecimal` | Integer | Decimal places |
| `feeCoinKey` | String | Coin used for gas fees |
| `feeUnit` | String | Fee unit name (Gwei, satoshis) |
| `feeDecimal` | Integer | Fee decimal |
| `blockChain` | String | Blockchain name |
| `network` | String | Mainnet / Testnet |
| `tokenIdentifier` | String | Contract address or NATIVE |
| `minTransferAmount` | String | Minimum transfer amount |
| `gasLimit` | Long | Default gas limit |
| `isUtxo` | String | YES/NO — UTXO-based chain |
| `isMemo` | String | YES/NO — Requires memo/tag |
| `blockchainType` | String | EVM, UTXO, etc. |
| `txRefUrl` | String | Block explorer TX URL template |

---

### Validate a Coin Address

Use before adding to whitelist or sending funds:

```java
CheckCoinAddressRequest req = new CheckCoinAddressRequest();
req.setCoinKey("ETHEREUM_ETH");
req.setAddress("0xAbCd...");

CheckCoinAddressResponse resp = ServiceExecutor.execute(coinApi.checkCoinAddress(req));
boolean isValid = resp.getIsValid();  // true or false
```

**CheckCoinAddressRequest Fields:**
| Field | Required | Description |
|-------|----------|-------------|
| `coinKey` | Yes | Coin identifier |
| `address` | Yes | Address to validate |

---

### Coin Balance Snapshot (Team-wide)

Query aggregate balance for specific coins across all accounts in the team:

```java
CoinBalanceSnapshotRequest req = new CoinBalanceSnapshotRequest();
req.setCoinKeys(Arrays.asList("ETHEREUM_ETH", "BITCOIN_BTC", "TRON_TRX"));

List<CoinBalanceSnapshotResponse> result = ServiceExecutor.execute(coinApi.coinBalanceSnapshot(req));
for (CoinBalanceSnapshotResponse item : result) {
    System.out.println(item.getCoinKey() + ": " + item.getCoinBalance());
}
```

> Also accessible via: `GET /v1/coin/balance/snapshot`

---

### Get Current Block Height

```java
CoinBlockHeightRequest req = new CoinBlockHeightRequest();
req.setCoinKeys(Arrays.asList("ETHEREUM_ETH", "BITCOIN_BTC"));

List<CoinBlockHeightResponse> result = ServiceExecutor.execute(coinApi.coinBlockHeight(req));
for (CoinBlockHeightResponse item : result) {
    System.out.println(item.getCoinKey() + " block height: " + item.getBlockHeight());
}
```

---

### List Coins Under Maintenance

```java
List<CoinMaintainResponse> maintained = ServiceExecutor.execute(coinApi.listCoinMaintain());
```

---

## AccountApiService — Per-Account Coin Operations

### Add Coin(s) to a Wallet Account (V2 — Recommended)

Supports adding up to 20 coins in a single call. Adding a token automatically adds its parent mainnet coin.

```java
CreateAccountCoinV2Request req = new CreateAccountCoinV2Request();
req.setAccountKey("your-account-key");
req.setCoinKeyList(Arrays.asList("USDT(ERC20)_ETHEREUM_USDT", "ETH_SEPOLIA"));

List<CreateAccountCoinResponse> respList = ServiceExecutor.execute(accountApi.createAccountCoinV2(req));
for (CreateAccountCoinResponse coin : respList) {
    System.out.println(coin.getCoinKey() + " address: " + coin.getAddress());
}
```

> **V1 (legacy, single coin):** `CreateAccountCoinRequest` with `setCoinKey(String)` — still supported.

**Notes:**
- If `coinKey` was already added, the response returns the same data as before (idempotent).
- If a previously added coin was manually disabled on the UI, re-adding via API will NOT re-enable it.

---

### List Coins for a Wallet Account

Query balances and addresses for all coins on a specific account:

```java
ListAccountCoinRequest req = new ListAccountCoinRequest();
req.setAccountKey("your-account-key");

List<AccountCoinResponse> coins = ServiceExecutor.execute(accountApi.listAccountCoin(req));
for (AccountCoinResponse coin : coins) {
    System.out.println(coin.getCoinKey() + " balance: " + coin.getBalance()
                       + " address: " + coin.getAddress());
}
```

> Endpoint: `POST /v1/account/coin/list`

**AccountCoinResponse Key Fields:**
| Field | Description |
|-------|-------------|
| `coinKey` | Coin identifier |
| `address` | Deposit address |
| `balance` | Available balance (string) |
| `frozenBalance` | Balance locked in pending txs |
| `totalBalance` | balance + frozenBalance |

---

## Request/Response Class Summary

| Operation | Service | Request Class | Response Class |
|-----------|---------|--------------|----------------|
| List all coins | `CoinApiService` | *(none)* | `List<CoinResponse>` |
| Validate address | `CoinApiService` | `CheckCoinAddressRequest` | `CheckCoinAddressResponse` |
| Balance snapshot | `CoinApiService` | `CoinBalanceSnapshotRequest` | `List<CoinBalanceSnapshotResponse>` |
| Block height | `CoinApiService` | `CoinBlockHeightRequest` | `List<CoinBlockHeightResponse>` |
| Coins in maintenance | `CoinApiService` | *(none)* | `List<CoinMaintainResponse>` |
| Add coin (V2) | `AccountApiService` | `CreateAccountCoinV2Request` | `List<CreateAccountCoinResponse>` |
| Add coin (V1) | `AccountApiService` | `CreateAccountCoinRequest` | `List<CreateAccountCoinResponse>` |
| List account coins | `AccountApiService` | `ListAccountCoinRequest` | `List<AccountCoinResponse>` |

---

## Automatic Coin Addition Rules

1. If a **mainnet coin** is enabled and a **token** of that chain is received as deposit → the token coin type is **automatically added**.
2. If you add a **token** coinKey via API → the corresponding **mainnet coin** is also automatically added.
3. If you add a **mainnet** coinKey → only that mainnet coin is added (no automatic token additions).

---

## Supported Test Networks

| Network | coinKey Example | Notes |
|---------|----------------|-------|
| Ethereum Sepolia | `ETH_SEPOLIA` | Test faucet tokens available |
| Bitcoin Testnet | `BITCOIN_BTC_TESTNET` | Faucet: https://coinfaucet.eu/en/btc-testnet4/ |
| TRON Shasta | TRON testnet coinKey | TLK token available on Shasta |

Test token contracts (Ethereum Sepolia):
- GSK: `0xF191b0720cb49DdAb6ECd72a65955a35b31fc944`
- USDC: `0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238` — Faucet: https://faucet.circle.com/

---

## Notes

- `coinKey` format examples: `ETHEREUM_ETH`, `BITCOIN_BTC`, `USDT(ERC20)_ETHEREUM_USDT`, `ETH_SEPOLIA`, `TRON_TRX`, `ETH(SEPOLIA)_ETHEREUM_SEPOLIA`.
- Always validate addresses with `checkCoinAddress` before sending funds or adding to whitelist.
- Team-level balance: use `coinBalanceSnapshot`. Per-account balance: use `listAccountCoin`.
- Exchange rates are sourced from **CoinGecko** and **CoinMarketCap (CMC)**.

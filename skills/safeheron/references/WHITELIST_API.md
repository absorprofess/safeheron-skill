# Whitelist API Reference

Whitelists allow you to pre-approve destination addresses for transactions. Transactions to non-whitelisted addresses can be blocked by policy.

## Create a Whitelist
```
POST /v1/whitelist/create
```
```java
CreateWhitelistRequest req = new CreateWhitelistRequest();
req.setCustomerRefId(UUID.randomUUID().toString());
req.setWhitelistName("Exchange Hot Wallet");
req.setCoinKey("ETHEREUM_ETH");
req.setAddress("0xDestinationAddress");
req.setAddressName("Binance ETH");

CreateWhitelistResponse resp = whitelistService.createWhitelist(req);
String whitelistKey = resp.getWhitelistKey();
```

## Create Whitelist Based on a Transaction
```
POST /v1/whitelist/createByTxKey
```
Automatically creates a whitelist entry from a previously confirmed transaction's destination.

## Modify a Whitelist
```
POST /v1/whitelist/update
```
```java
UpdateWhitelistRequest req = new UpdateWhitelistRequest();
req.setWhitelistKey(whitelistKey);
req.setWhitelistName("Updated Name");
whitelistService.updateWhitelist(req);
```

## Retrieve a Single Whitelist
```
POST /v1/whitelist/one
```
```java
GetWhitelistRequest req = new GetWhitelistRequest();
req.setWhitelistKey(whitelistKey);
GetWhitelistResponse resp = whitelistService.getWhitelist(req);
```

## List Whitelist Data
```
POST /v1/whitelist/list
```
```java
ListWhitelistRequest req = new ListWhitelistRequest();
req.setCoinKey("ETHEREUM_ETH");   // optional filter
req.setPage(1);
req.setPageSize(20);
ListWhitelistResponse resp = whitelistService.listWhitelists(req);
```

## Delete a Whitelist
```
POST /v1/whitelist/delete
```
```java
DeleteWhitelistRequest req = new DeleteWhitelistRequest();
req.setWhitelistKey(whitelistKey);
whitelistService.deleteWhitelist(req);
```

---

## Whitelist Fields
| Field | Description |
|-------|-------------|
| whitelistKey | Unique identifier |
| whitelistName | Display name |
| coinKey | Associated coin (e.g. ETHEREUM_ETH) |
| address | The destination address |
| addressName | Human-readable label |
| status | ACTIVE / INACTIVE |

---

## Best Practices
- Always verify address format with the [Verify Coin Address](WALLET_API.md) endpoint before adding to whitelist.
- Use descriptive `whitelistName` and `addressName` for audit trails.
- For regulated environments, require human approval before whitelist additions (configure in Safeheron Console policy).

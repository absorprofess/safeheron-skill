# API Co-Signer & Approval Callback

## What is the API Co-Signer?

The API Co-Signer is a service you deploy that **automatically approves or rejects transactions** based on your business logic. It replaces manual approval workflows for programmatic use cases and is required for unattended operation in production environments.

When a transaction requires approval, Safeheron calls your **Approval Callback Service** (an HTTP endpoint you implement) with an encrypted callback payload. Your service evaluates the transaction and responds `APPROVE` or `REJECT`.

---

## How It Works

```
1. Client creates transaction / MPC sign / Web3 sign
2. Safeheron: transaction enters WAIT_AUDIT status
3. API Co-Signer polls Safeheron (every 5s in v2.x, every 1s in v1.x)
4. API Co-Signer calls your Approval Callback Service (HTTP POST)
5. Your service returns APPROVE or REJECT
6. Transaction proceeds (WAIT_SIGN) or is REJECTED
```

**Approval Callback timeout:** Response timestamp must be within **5 seconds** of the API Co-Signer server's current time.

---

## Approval Callback Request

Safeheron sends an **encrypted HTTP POST** to your callback URL. The request body structure is identical to webhook payloads:

```json
{
  "timestamp": "1623038312088",
  "sig":        "<signed-with-cosigner-identity-private-key-base64>",
  "key":        "<RSA-encrypted-AES-key-base64>",
  "bizContent": "<AES-encrypted-callback-payload-base64>",
  "rsaType":    "ECB_OAEP",
  "aesType":    "GCM_NOPADDING"
}
```

### Key Difference from Regular Webhooks

- The callback `sig` is signed with the **API Co-Signer's identity private key** (not the Safeheron platform key).
- You verify the signature using the **Co-Signer's exported identity public key**:
  ```bash
  sudo ./cosigner export-public-key
  ```
- The `key` field is encrypted with your **Callback Public Key** (the public key uploaded when registering the callback URL in Console).
- Decrypt using the corresponding **Callback Private Key** (stored on your callback service).

---

## Decrypted Callback Payload Structure

After decryption, `bizContent` is a JSON object:

```json
{
  "type": "TRANSACTION_APPROVAL",
  "customerContent": { ... }
}
```

### Callback Types

| `type` | Description |
|--------|-------------|
| `TRANSACTION_APPROVAL` | Regular transaction approval |
| `WEB3_SIGN_APPROVAL` | Web3 signing approval |
| `MPC_SIGN_APPROVAL` | MPC raw signing approval |

---

## TRANSACTION_APPROVAL Payload (`customerContent`)

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Safeheron transaction key |
| `customerRefId` | String | Your reference ID |
| `txHash` | String | Transaction hash (if available) |
| `coinKey` | String | Coin identifier |
| `txAmount` | String | Transaction amount |
| `transactionType` | String | Transaction type |
| `transactionStatus` | String | Current status |
| `transactionSubStatus` | String | Sub-status |
| `sourceAccountKey` | String | Sender wallet key |
| `sourceAccountType` | String | e.g. `VAULT_ACCOUNT` |
| `sourceAddress` | String | Sender address |
| `destinationAccountKey` | String | Recipient wallet key |
| `destinationAccountType` | String | e.g. `ONE_TIME_ADDRESS` |
| `destinationAddress` | String | Recipient address |
| `destinationAddressList` | List | Multi-dest address list |
| `estimateFee` | String | Estimated fee |
| `feeCoinKey` | String | Fee coin |
| `note` | String | Transaction note |
| `customerExt1` | String | Custom field 1 |
| `customerExt2` | String | Custom field 2 |
| `amlLock` | String | AML status: `YES` / `NO` |
| `amlList` | List\<Aml\> | AML assessments (provider, riskLevel, etc.) |
| `createTime` | Long | Unix timestamp (ms) |

---

## WEB3_SIGN_APPROVAL Payload (`customerContent`)

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | Web3 sign request key |
| `customerRefId` | String | Your reference ID |
| `transactionStatus` | String | Status |
| `subjectType` | String | `ETH_SIGN`, `PERSONAL_SIGN`, `ETH_SIGNTYPEDDATA`, `ETH_SIGNTRANSACTION` |
| `accountKey` | String | Web3 wallet account key |
| `sourceAddress` | String | Signing address |
| `createTime` | Long | Unix timestamp (ms) |
| `note` | String | Note |
| `customerExt1` | String | Custom field 1 |
| `customerExt2` | String | Custom field 2 |
| `message` | Object | For personalSign/ethSignTypedData: `{chainId, data}` |
| `messageHash` | Object | For ethSign: `{chainId, data:[hashes]}` |
| `transaction` | Object | For ethSignTransaction: `{chainId, to, value, gasLimit, gasPrice, maxFeePerGas, maxPriorityFeePerGas, nonce, data}` |

---

## MPC_SIGN_APPROVAL Payload (`customerContent`)

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | String | MPC sign request key |
| `customerRefId` | String | Your reference ID |
| `transactionStatus` | String | Status |
| `sourceAccountKey` | String | Wallet key |
| `createTime` | Long | Unix timestamp (ms) |
| `customerExt1` | String | Custom field 1 |
| `customerExt2` | String | Custom field 2 |
| `signAlg` | String | `Secp256k1` or `Ed25519` |
| `hashs` | List | Hashes to sign: `[{note, hash}]` |
| `dataList` | List | Raw data list: `[{note, data}]` |

---

## Approval Callback Response

Your callback service must return an **encrypted response** using the same AES+RSA scheme, signed with your Callback Private Key:

```json
{
  "approveStatus": "APPROVE",
  "customerRefId": "<echo-back-from-request>",
  "rejectReason":  ""
}
```

| Field | Type | Values |
|-------|------|--------|
| `approveStatus` | String | `APPROVE` or `REJECT` |
| `customerRefId` | String | Echo back the incoming `customerRefId` |
| `rejectReason` | String | Optional reason when rejecting |

---

## Approval Logic Examples

### Allow Only Whitelisted Destinations

```java
private boolean shouldApprove(TransactionApproval req) {
    if (!"TRANSACTION_APPROVAL".equals(callbackType)) return true;
    Set<String> allowedAddresses = loadWhitelistedAddresses(req.getCoinKey());
    return allowedAddresses.contains(req.getDestinationAddress());
}
```

### Amount Limit per Account

```java
private boolean shouldApprove(TransactionApproval req) {
    BigDecimal amount = new BigDecimal(req.getTxAmount());
    BigDecimal limit  = getAccountLimit(req.getSourceAccountKey());
    return amount.compareTo(limit) <= 0;
}
```

### AML Risk Check

```java
private boolean shouldApprove(TransactionApproval req) {
    for (Aml aml : req.getAmlList()) {
        if ("HIGH".equalsIgnoreCase(aml.getRiskLevel())) {
            return false;
        }
    }
    return true;
}
```

---

## API Co-Signer Deployment Notes

| Topic | Detail |
|-------|--------|
| Polling interval | v2.x: every 5s; v1.x: every 1s |
| Callback timeout | Response must arrive within 5s of Co-Signer server time |
| Production | Callback URL is **required** for production teams |
| Test environment | Callback URL is optional |
| KMS support | AWS KMS, GCP KMS (Alibaba Cloud KMS not supported) |
| CLI commands | `sudo ./cosigner start`, `sudo ./cosigner stop`, `sudo ./cosigner logs -f` |
| Export Co-Signer public key | `sudo ./cosigner export-public-key` |
| Minimum server specs | Refer to Safeheron Console deployment guide |
| Config changes | Modify `.env` then run `stop` + `start` to reload |
| Update Callback URL | Web Console update takes effect within 5 min — no Co-Signer restart needed |
| IP whitelist | Add Co-Signer host IP to the API Key IP whitelist in Console |

---

## New vs. Old Co-Signer Version Differences

| Aspect | New Version (v2.x+) | Old Version (v1.x) |
|--------|--------------------|--------------------|
| RSA key pairs required | **1 pair** | 2 pairs |
| Polling interval | 5 seconds | 1 second |
| Cancel transaction in DB | Automatic | Manual SQL required |
| Config management | Secrets Manager (recommended) + .env | Config file |

**Old version** required a separate `biz_callback` (callback URL) and `biz_pubkey` (your public key) in the config file. The new version consolidates this into a single key pair registered in Web Console.

---

## CLI Commands Reference

```bash
sudo ./cosigner start               # Start Co-Signer
sudo ./cosigner stop                # Stop Co-Signer
sudo ./cosigner setup               # Initial setup (must run before start)
sudo ./cosigner export-public-key   # Export Co-Signer identity public key
./cosigner logs -f                  # Stream live logs
./cosigner logs -s                  # Export logs to file (for Support)
```

---

## Key Pair Configuration Summary

| Key | Owner | Purpose |
|-----|-------|---------|
| **Co-Signer Identity Private Key** | Safeheron (Co-Signer internal) | Signs callback requests sent to your service |
| **Co-Signer Identity Public Key** | Exported via `export-public-key` | You use this to verify callback request signatures |
| **Callback Public Key** | You generate & upload to Console | Safeheron Co-Signer encrypts the callback `key` field with this |
| **Callback Private Key** | You hold securely | Your callback service uses this to decrypt the AES key |

---

## Common Co-Signer Issues

| Issue | Cause | Resolution |
|-------|-------|------------|
| `Illegal IP` in logs | Co-Signer host IP not in API Key whitelist | Add IP to whitelist in Console, restart Co-Signer |
| `The final private key fragment is not exists` | Co-Signer not yet activated | Normal — proceed with activation workflow |
| Transaction stays in `WAIT_AUDIT` | Co-Signer not running or not polling | Check `./cosigner logs -f` for errors |
| `Timestamp out of range` | Server clock skew > 5s | Sync Co-Signer server time with NTP |
| Docker login failure | Token expired or firewall blocks `registry.gitlab.com` | Re-download CLI from Console; check firewall rules |

---

## Official SDK Demo

See the SDK repository for a working callback demo:
https://github.com/Safeheron/safeheron-api-sdk-java

---
name: safeheron-api
description: >
  Safeheron MPC wallet API integration skill. Use when working with Safeheron API to manage wallet accounts, create transactions, handle MPC signing, interact with Web3 wallet operations, manage whitelists, process webhooks, or integrate with the Java SDK (com.safeheron:api-sdk-java). Triggers: Safeheron, MPC wallet, crypto transaction via Safeheron, wallet account management, safeheron-api-sdk-java, API Co-Signer, webhook integration.
---

# Safeheron API Skill

A complete reference and integration guide for the Safeheron MPC Custody API and Java SDK.

## Quick Reference

- **API Base URL**: `https://api.safeheron.vip`
- **SDK Maven Artifact**: `com.safeheron:api-sdk-java:1.0.9`
- **Auth Scheme**: RSA-4096 (signature) + AES-256-GCM (payload encryption)
- **Docs**: https://docs.safeheron.com/api/en.html
- **SDK Repo**: https://github.com/Safeheron/safeheron-api-sdk-java

---

## 1. Authentication Overview

Safeheron uses a **hybrid RSA + AES encryption** scheme for every request and response. See [AUTH.md](references/AUTH.md) for the full step-by-step flow.

**Keys you need:**
| Key | Where to get | How used |
|-----|-------------|----------|
| `apiKey` | Safeheron Web Console → Settings → API | Sent in every request |
| Your RSA Private Key (4096-bit) | Generated locally via OpenSSL | Signs request params |
| Your RSA Public Key | Uploaded to Safeheron Console | Safeheron verifies your requests |
| Safeheron RSA Public Key | Downloaded from Safeheron Console | Encrypts AES key; verify responses |

**Generate RSA keypair:**
```bash
openssl genpkey -out api_private.pem -algorithm RSA -pkeyopt rsa_keygen_bits:4096
openssl rsa -in api_private.pem -out api_public.pem -pubout
```

---

## 2. Java SDK Setup

See [SDK_SETUP.md](references/SDK_SETUP.md) for full Maven/Gradle setup and config.

**Maven dependency:**
```xml
<dependency>
    <groupId>com.safeheron</groupId>
    <artifactId>api-sdk-java</artifactId>
    <version>1.0.9</version>
</dependency>
```

**Minimal config (YAML):**
```yaml
apiKey: YOUR_API_KEY
privateKey: YOUR_BASE64_PRIVATE_KEY
safeheronPublicKey: SAFEHERON_RSA_PUBLIC_KEY
baseUrl: https://api.safeheron.vip
requestTimeout: 20000
```

---

## 3. Core API Areas

| Area | Key Operations | Reference |
|------|---------------|-----------|
| Wallet Account | List, Create, Query by address, Add coins | [WALLET_API.md](references/WALLET_API.md) |
| Transaction | Create V3, List, Retrieve, Cancel, Estimate fee | [TRANSACTION_API.md](references/TRANSACTION_API.md) |
| Whitelist | Create, Modify, Delete, List | [WHITELIST_API.md](references/WHITELIST_API.md) |
| Web3 | ethSign, personalSign, ethSignTypedData, ethSignTransaction | [WEB3_API.md](references/WEB3_API.md) |
| MPC Sign | Create wallet, Sign transaction, Query status | [MPC_SIGN_API.md](references/MPC_SIGN_API.md) |
| Webhook | Transaction events, Whitelist events, Security events | [WEBHOOK.md](references/WEBHOOK.md) |
| Coins | Coin list, Verify address, Snapshot balance | [WALLET_API.md](references/WALLET_API.md) |

---

## 4. Common Task Recipes

### Create a wallet account
```java
SafeheronClient client = new SafeheronClient(config);
AccountService accountService = new AccountService(client);

CreateAccountRequest req = new CreateAccountRequest();
req.setAccountName("My Wallet");
CreateAccountResponse resp = accountService.createAccount(req);
String accountKey = resp.getAccountKey();
```

### Create a transaction (V3)
```java
TransactionService txService = new TransactionService(client);
CreateTransactionV3Request req = new CreateTransactionV3Request();
req.setCustomerRefId(UUID.randomUUID().toString());
req.setSourceAccountKey(accountKey);
req.setDestinationAddress("0x...");
req.setCoinKey("ETHEREUM_ETH");
req.setTxAmount("0.01");
CreateTransactionV3Response resp = txService.createTransactionV3(req);
```

### MPC Sign flow
See [MPC_SIGN_API.md](references/MPC_SIGN_API.md) for the full ERC-20 token transfer pattern.

---

## 5. Transaction & Signing Lifecycle

For the complete status flow diagrams and best practices for deposit/withdrawal scenarios, see [TRANSACTION_API.md](references/TRANSACTION_API.md).

Key statuses: `SUBMITTED → WAIT_AUDIT → WAIT_SIGN → BROADCASTING → SUCCESS | FAILED | REVOKED`

---

## 6. Webhook Integration

Safeheron pushes events for transaction state changes, whitelist updates, and security alerts. See [WEBHOOK.md](references/WEBHOOK.md) for all event types and verification steps.

---

## 7. Important Notes

- **IP Whitelisting** is mandatory — register the server IP in the Safeheron Console before calling the API.
- **Idempotency**: Use `customerRefId` (your unique ID) to avoid duplicate transactions.
- All monetary amounts are **strings** (e.g. `"0.01"`) — never floats.
- Always verify response signatures before processing `bizContent`.
- The API Co-Signer enables **automated approval** for programmatic workflows — see [COSIGNER.md](references/COSIGNER.md).

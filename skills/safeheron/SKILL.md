---
name: safeheron-api
description: >
  Safeheron MPC wallet API integration skill. Use when working with Safeheron API to manage wallet accounts, create transactions, handle MPC signing, interact with Web3 wallet operations, manage whitelists, process webhooks, or integrate with the Java SDK (com.safeheron:api-sdk-java). Triggers: Safeheron, MPC wallet, crypto transaction via Safeheron, wallet account management, safeheron-api-sdk-java, API Co-Signer, webhook integration.
---

# Safeheron API Skill

A complete reference and integration guide for the Safeheron MPC Custody API and Java SDK.

## ⚠️ Critical SDK Pattern (Read First)

The Java SDK does **NOT** use service classes like `SafeheronClient`, `AccountService`, or `TransactionService`.
The correct pattern is:

1. Build a `SafeheronConfig` using its **builder**
2. Use `ServiceCreator.create(XxxApiService.class, config)` to obtain an API interface
3. Call methods on that interface, wrapping every call in `ServiceExecutor.execute(...)`

All API interfaces live under `com.safeheron.client.api.*`
All request/response classes live under `com.safeheron.client.request.*` / `com.safeheron.client.response.*`

---

## Quick Reference

- **API Base URL**: `https://api.safeheron.vip`
- **SDK Maven Artifact**: `com.safeheron:api-sdk-java:1.0.9`
- **Auth Scheme**: RSA-4096 (signature) + AES-256-GCM (payload encryption)
- **SDK Repo**: https://github.com/Safeheron/safeheron-api-sdk-java
- **API Docs**: https://docs.safeheron.com/api/en.html

---

## SDK API Service Classes

| API Area | Interface Class | Package |
|----------|----------------|---------|
| Wallet Account | `AccountApiService` | `com.safeheron.client.api` |
| Transaction | `TransactionApiService` | `com.safeheron.client.api` |
| MPC Sign | `MPCSignApiService` | `com.safeheron.client.api` |
| Web3 | `Web3ApiService` | `com.safeheron.client.api` |
| Whitelist | `WhitelistApiService` | `com.safeheron.client.api` |

---

## Minimal Working Example

```java
import com.safeheron.client.api.AccountApiService;
import com.safeheron.client.config.SafeheronConfig;
import com.safeheron.client.request.CreateAccountRequest;
import com.safeheron.client.response.CreateAccountResponse;
import com.safeheron.client.utils.ServiceCreator;
import com.safeheron.client.utils.ServiceExecutor;

// 1. Build config with builder
SafeheronConfig config = SafeheronConfig.builder()
        .baseUrl("https://api.safeheron.vip")
        .apiKey("YOUR_API_KEY")
        .rsaPrivateKey("YOUR_BASE64_PKCS8_PRIVATE_KEY")
        .safeheronRsaPublicKey("SAFEHERON_BASE64_PUBLIC_KEY")
        .requestTimeout(20000L)
        .build();

// 2. Create API service via ServiceCreator
AccountApiService accountApi = ServiceCreator.create(AccountApiService.class, config);

// 3. Execute — always wrap with ServiceExecutor.execute()
CreateAccountRequest req = new CreateAccountRequest();
req.setAccountName("my-wallet");
req.setHiddenOnUI(false);

CreateAccountResponse resp = ServiceExecutor.execute(accountApi.createAccount(req));
System.out.println("accountKey: " + resp.getAccountKey());
```

---

## Reference Files

| Topic | File |
|-------|------|
| RSA+AES auth flow | [AUTH.md](references/AUTH.md) |
| Maven/Gradle setup, config | [SDK_SETUP.md](references/SDK_SETUP.md) |
| Wallet account & coin APIs | [WALLET_API.md](references/WALLET_API.md) |
| Transaction creation & status | [TRANSACTION_API.md](references/TRANSACTION_API.md) |
| Web3 signing (ethSign, EIP-712) | [WEB3_API.md](references/WEB3_API.md) |
| MPC Sign + ERC-20 example | [MPC_SIGN_API.md](references/MPC_SIGN_API.md) |
| Whitelist CRUD | [WHITELIST_API.md](references/WHITELIST_API.md) |
| Webhook events & handler | [WEBHOOK.md](references/WEBHOOK.md) |
| API Co-Signer / Approval Callback | [COSIGNER.md](references/COSIGNER.md) |

---

## Important Notes

- **IP Whitelisting** is mandatory — register server IP in Safeheron Console first.
- **Idempotency** — always set `customerRefId` to your own unique reference ID.
- All monetary amounts are **strings** — never use float/double.
- `requestTimeout` is **Long** (milliseconds), not int.
- `SafeheronConfig` builder: platform public key → `.safeheronRsaPublicKey(...)`, your private key → `.rsaPrivateKey(...)`.

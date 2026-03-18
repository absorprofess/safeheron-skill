---
name: safeheron-api
description: >
  Safeheron MPC wallet API integration skill. Use when working with Safeheron API to set up from scratch, manage wallet accounts, create transactions, handle MPC signing, interact with Web3 wallet operations, manage whitelists, process webhooks, integrate with the Java SDK (com.safeheron:api-sdk-java), use Gas Station, perform AML/compliance checks, use Tools API, or handle API Co-Signer callbacks. Triggers: Safeheron, MPC wallet, crypto transaction via Safeheron, wallet account management, safeheron-api-sdk-java, API Co-Signer, webhook integration, Gas Service, KYT, AML checker, whitelist management, getting started, first API call, RSA key generation, SDK setup.
---

# Safeheron API Skill

A complete reference and integration guide for the Safeheron MPC Custody API and Java SDK.

## 🚀 New Here? Start with the Getting Started Guide

**→ [GETTING_STARTED.md](references/GETTING_STARTED.md)**

Covers all 5 onboarding steps:
1. Generate RSA key pair
2. Configure API account in Safeheron Console
3. Add SDK to project dependencies
4. Configure environment & inject credentials
5. First API request

---

## 🔐 Security-First: Read Before Writing Any Code

**These security principles are non-negotiable and must be reflected in all generated code:**

1. **Credential injection** — Always load API keys, RSA private keys, and platform public keys from environment variables or a secrets manager. Never hardcode them.
2. **`customerRefId` first** — Generate and persist `customerRefId` to your DB **before** calling any Safeheron create API. This enables safe retry on network timeout.
3. **Webhook signature verification** — Every webhook payload must have its `sig` verified using Safeheron's RSA public key before any processing. Never skip this.
4. **No status rollback** — Webhook handlers must never downgrade a transaction status (e.g., `COMPLETED` → `CONFIRMING`). Terminal statuses are final.
5. **Webhook IP whitelist** — Only accept webhook traffic from Safeheron's egress IPs: `18.162.105.64`, `18.167.22.59`, `18.167.21.182`
6. **Webhook + REST API polling** — Always implement both: Webhook as primary, REST API polling as fallback. Never rely on only one.
7. **`failOnAml: true` by default** — Always include AML checks; only disable explicitly when the business case is confirmed.
8. **`failOnContract: true` by default** — Block contract address destinations unless explicitly required by the use case.
9. **Policy: least privilege** — API Keys, approval policies, and server permissions must be scoped to the minimum required.
10. **Minimum deposit filter** — Filter dust/address-pollution deposits by enforcing a minimum deposit amount in your system.

→ **[SECURITY_CHECKLIST.md](references/SECURITY_CHECKLIST.md)** — Complete pre-launch checklist
→ **[POLICY_STRATEGY.md](references/POLICY_STRATEGY.md)** — Policy configuration guide with approval tiers
→ **[BUSINESS_PATTERNS.md](references/BUSINESS_PATTERNS.md)** — Deposit/withdrawal/sweep patterns with security baked in

---

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
- **Safeheron Egress IPs** (for webhook/callback firewall rules): `18.162.105.64`, `18.167.22.59`, `18.167.21.182`

---

## SDK API Service Classes

| API Area | Interface Class | Package |
|----------|----------------|---------|
| Wallet Account | `AccountApiService` | `com.safeheron.client.api` |
| Coin Management | `CoinApiService` | `com.safeheron.client.api` |
| Transaction | `TransactionApiService` | `com.safeheron.client.api` |
| MPC Sign | `MPCSignApiService` | `com.safeheron.client.api` |
| Web3 | `Web3ApiService` | `com.safeheron.client.api` |
| Whitelist | `WhitelistApiService` | `com.safeheron.client.api` |
| Compliance (KYT) | `ComplianceApiService` | `com.safeheron.client.api` |
| Gas Station | `GasApiService` | `com.safeheron.client.api` |
| Tools (AML Checker) | `ToolsApiService` | `com.safeheron.client.api` |

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
        .apiKey(System.getenv("SAFEHERON_API_KEY"))
        .rsaPrivateKey(System.getenv("SAFEHERON_RSA_PRIVATE_KEY"))
        .safeheronRsaPublicKey(System.getenv("SAFEHERON_PLATFORM_PUBLIC_KEY"))
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
| **Getting Started (0 → first call)** | **[GETTING_STARTED.md](references/GETTING_STARTED.md)** |
| **Security pre-launch checklist** | **[SECURITY_CHECKLIST.md](references/SECURITY_CHECKLIST.md)** |
| **Policy configuration & approval tiers** | **[POLICY_STRATEGY.md](references/POLICY_STRATEGY.md)** |
| **Deposit/withdrawal/sweep patterns** | **[BUSINESS_PATTERNS.md](references/BUSINESS_PATTERNS.md)** |
| RSA+AES auth flow & key generation | [AUTH.md](references/AUTH.md) |
| Maven/Gradle setup, Spring Boot config | [SDK_SETUP.md](references/SDK_SETUP.md) |
| Wallet account CRUD & coin management | [WALLET_API.md](references/WALLET_API.md) |
| Coin list, address validation, balance snapshot | [COIN_API.md](references/COIN_API.md) |
| Transaction create, query, list, fee, cancel, speed-up | [TRANSACTION_API.md](references/TRANSACTION_API.md) |
| MPC Sign flow + ERC-20 example | [MPC_SIGN_API.md](references/MPC_SIGN_API.md) |
| Web3 signing (ethSign, personalSign, EIP-712, signTx) | [WEB3_API.md](references/WEB3_API.md) |
| Whitelist CRUD with exact class names | [WHITELIST_API.md](references/WHITELIST_API.md) |
| Compliance / KYT report | [COMPLIANCE_API.md](references/COMPLIANCE_API.md) |
| Gas Station status & auto-refill | [GAS_API.md](references/GAS_API.md) |
| Tools API — AML address checker | [TOOLS_API.md](references/TOOLS_API.md) |
| Webhook events, handler, re-push | [WEBHOOK.md](references/WEBHOOK.md) |
| API Co-Signer / Approval Callback + security deployment | [COSIGNER.md](references/COSIGNER.md) |
| Error codes & troubleshooting | [ERROR_CODES.md](references/ERROR_CODES.md) |
| FAQ — real-world Q&A (CN+EN) | [FAQ.md](references/FAQ.md) |

---

## Important Notes

### SDK Usage
- **IP Whitelisting** is mandatory — register server IP in Safeheron Console first. API calls from unregistered IPs are rejected.
- **Idempotency** — generate `customerRefId` (UUID) and **save to DB before calling Safeheron**. On timeout, retry with the same ID. Error `9001` = duplicate refId (query existing instead of creating new).
- All monetary amounts are **strings** — never use float/double.
- `requestTimeout` is **Long** (milliseconds), not int. Use `20000L`.
- `pageSize` / `pageNumber` in list requests are **Long**, not int.
- `SafeheronConfig` builder: Safeheron platform public key → `.safeheronRsaPublicKey(...)`, your PKCS8 private key → `.rsaPrivateKey(...)`. Do NOT swap them.
- **Web3 API** requires a Web3 wallet (`accountType=WEB3_ACCOUNT`). Vault account keys cause "account not found".
- **MPC Sign** requires a special policy — contact Safeheron Support to enable (error `9028` = policy not found).
- Common auth errors: `1010` = wrong Safeheron platform public key; `1012` = wrong private key or not PKCS8 format.
- SDK is **backward compatible** — upgrading only adds new methods, never removes old ones.

### Security (Generated Code Must Always Follow These)
- **Never hardcode** API keys, RSA keys, or any credentials — use env vars or a secrets manager.
- **Webhook signature must be verified** on every incoming request before processing.
- **Webhook server must IP-whitelist** Safeheron's egress IPs: `18.162.105.64`, `18.167.22.59`, `18.167.21.182`.
- **Never roll back transaction status** — if DB status is already terminal (`COMPLETED`/`FAILED`/`REJECTED`), ignore late-arriving Webhook events.
- **Always implement both** Webhook (primary) and REST API polling (fallback) for transaction state sync.
- **`failOnAml: true`** is the secure default — only disable when explicitly required.
- **`failOnContract: true`** is the secure default — only disable when sending to contract addresses is intentional.
- **Deposit wallets** should use `hiddenOnUI: true` and `accountTag: "DEPOSIT"` for Auto-Sweep compatibility.
- **Minimum deposit amount** must be configured in your system to filter dust/address-pollution attacks.
- **Policy: minimum privilege** — restrict API Key permissions, source wallets, and approval chains to what each use case strictly needs.
- **API Co-Signer host** must be in a private isolated network with no public internet inbound access.
- Subscribe to `ILLEGAL_IP_REQUEST`, `NO_MATCHING_TRANSACTION_POLICY`, and `GAS_BALANCE_WARNING` webhook events as operational alerts.

---
name: safeheron
description: >
  Use when working with Safeheron API to set up from scratch, manage wallet accounts, create transactions, handle MPC signing, interact with Web3 wallet operations, manage whitelists, process webhooks, integrate with the Java SDK (com.safeheron:api-sdk-java), use Gas Station, perform AML/compliance checks, use Tools API, or handle API Co-Signer callbacks.
---

# Safeheron API Skill

A complete reference and integration guide for the Safeheron MPC Custody API and Java SDK.

**When NOT to use:** For generic blockchain/Web3 questions unrelated to Safeheron's API or SDK; for other MPC custody providers.

## 🚀 New Here? Start with the Getting Started Guide

**→ [GETTING_STARTED.md](references/GETTING_STARTED.md)**

Covers all 5 onboarding steps:
1. Generate RSA key pair
2. Configure API account in Safeheron Console
3. Add SDK to project dependencies
4. Configure environment & inject credentials
5. First API request

---

## 🔐 Security-First: Mandatory Checklist Before Writing Any Code

**These security principles are non-negotiable and must be reflected in all generated code. Use TaskCreate to create one task per item below before writing ANY code — including bulk "generate all features" requests. Items that are N/A must state a reason; they cannot be silently omitted. When user requests involve credentials, transfers, Webhooks, Co-Signer, or deployment, proactively explain the applicable requirements without waiting to be asked.**

### Credential & Key Management
- [ ] **Credential injection** — Always load API keys, RSA private keys, and platform public keys from a secrets manager. Never hardcode them in source code or config files.
- [ ] **Use the right secrets manager for the deployment target** — AWS KMS or GCP KMS for cloud deployments; HashiCorp Vault for self-hosted; a file outside the project directory (never committed to git) for local development only.

### Transfer & Whitelist Security
- [ ] **`customerRefId` first** — Generate and persist `customerRefId` to your DB **before** calling any Safeheron create API. On timeout, retry with the same ID. Error `9001` = already exists, query instead of recreating.
- [ ] **Whitelist addresses for formal transfers** — `ONE_TIME_ADDRESS` must only be used for genuinely temporary, one-off payments. All recurring or formal transfers (exchange hot wallets, partner addresses, internal accounts) require a whitelisted address.
- [ ] **AML check before every transfer** — Call `ToolsApiService` to screen the destination address before creating any outbound transaction. Intercept or alert on high-risk addresses.
- [ ] **Validate address format** — Call `CoinApiService.checkCoinAddress()` before adding an address to the whitelist or initiating a transfer. Never trust user-supplied address strings directly.
- [ ] **Amounts: String in API, BigDecimal in code** — Never use `float` or `double` for monetary values. API request fields use `String`; application-side calculations use `BigDecimal`.
- [ ] **`failOnAml: true` by default** — Only disable when the business case is explicitly confirmed.
- [ ] **`failOnContract: true` by default** — Block contract address destinations unless explicitly required.

### Co-Signer Approval Callback
- [ ] **No blind approval** — The Co-Signer callback service must validate every transaction against the business system before returning `APPROVE`: verify that `customerRefId` matches a real pending order, that the amount matches, and that the destination address matches.
- [ ] **Co-Signer host in private network** — The Co-Signer must be deployed in a private isolated network with no public internet inbound access. The Approval Callback Service must only accept traffic from the Co-Signer host IP.
- [ ] **Production Co-Signer API Key must have Callback configured** — Never skip Callback URL configuration for a Co-Signer API Key in production.

### Webhook Security
- [ ] **Webhook signature verification** — Every webhook payload must have its `sig` verified using Safeheron's RSA public key before any processing. Reject events with invalid signatures immediately.
- [ ] **Webhook must use HTTPS** — Never expose an HTTP webhook endpoint in production.
- [ ] **Webhook handlers must be idempotent** — Safeheron retries up to 7 times. Never double-credit a deposit or double-process a withdrawal on duplicate events.
- [ ] **No status rollback; terminal state wins** — If the current DB status is already `COMPLETED`, `FAILED`,`CANCELLED`, or `REJECTED`, discard any later-arriving event for that transaction — even if it carries an intermediate status (out-of-order delivery is possible).
- [ ] **Webhook IP whitelist** — Only accept webhook traffic from Safeheron's egress IPs: `18.162.105.64`, `18.167.22.59`, `18.167.21.182`
- [ ] **Webhook + REST API polling** — Always implement both: Webhook as primary, REST API polling as fallback. Call `/v1/transactions/one` periodically to re-request failed deliveries.

### Policy & Operational Alerts
- [ ] **Policy: least privilege** — API Keys, approval policies, and server permissions must be scoped to the minimum required.
- [ ] **Catch-all blocking policy** — Add a "block all" rule at the bottom of the policy stack to intercept transactions that match no defined rule.
- [ ] **Subscribe to operational alerts** — Always subscribe to `ILLEGAL_IP_REQUEST`, `NO_MATCHING_TRANSACTION_POLICY`,`GAS_BALANCE_WARNING`, and `AML_KYT_ALERT` webhook events.
- [ ] **Minimum deposit filter** — Filter dust/address-pollution deposits by enforcing a minimum deposit amount in your system.

→ **[SECURITY_BEST_PRACTICES.md](references/SECURITY_BEST_PRACTICES.md)** — Detailed implementation guide with code examples (read this)
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
| **Security best practices (code-level, with examples)** | **[SECURITY_BEST_PRACTICES.md](references/SECURITY_BEST_PRACTICES.md)** |
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


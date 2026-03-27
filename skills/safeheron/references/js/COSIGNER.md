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

## Approval Callback Service (Express.js)

### Using SDK CoSignerConverter

```typescript
import express from 'express';
import { CoSignerConverter } from '@safeheron/api-sdk';
import { SafeheronCoSignerConfig } from '@safeheron/api-sdk';
import { readFileSync } from 'fs';
import path from 'path';

const app = express();
app.use(express.json());

// Configure the converter
const cosignerConfig: SafeheronCoSignerConfig = {
  approvalCallbackServicePrivateKey: readFileSync(
    path.resolve('./keys/callback_private.pem'), 'utf8'
  ),
  coSignerPubKey: readFileSync(
    path.resolve('./keys/cosigner_identity_public.pem'), 'utf8'
  ),
};

const converter = new CoSignerConverter(cosignerConfig);

app.post('/cosigner/callback', (req, res) => {
  try {
    // 1. Decrypt and verify signature
    //    requestV3convert() handles:
    //    - Verifies RSA signature using Co-Signer identity public key
    //    - Decrypts AES key using your callback private key
    //    - Decrypts bizContent
    const decrypted = converter.requestV3convert(req.body);
    const payload = JSON.parse(decrypted);

    // 2. Business validation
    const action = evaluateTransaction(payload);

    // 3. Encrypt response
    const encryptedResponse = converter.responseV3convert({
      action: action,  // "APPROVE" or "REJECT"
      approvalId: payload.approvalId,
    });

    res.json(encryptedResponse);
  } catch (e) {
    console.error('Callback error:', e);
    // On error, respond with REJECT to be safe
    const encryptedResponse = converter.responseV3convert({
      action: 'REJECT',
      approvalId: '',
    });
    res.json(encryptedResponse);
  }
});

app.listen(9999, () => {
  console.log('Approval Callback Service listening on port 9999');
});
```

---

## Approval Callback Request

Safeheron sends an **encrypted HTTP POST** to your callback URL. The request body structure:

```json
{
  "timestamp": "1623038312088",
  "sig":        "<signed-with-cosigner-identity-private-key-base64>",
  "key":        "<RSA-encrypted-AES-key-base64>",
  "bizContent": "<AES-encrypted-callback-payload-base64>",
  "version":    "<protocol-version>",
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
- The `key` field is encrypted with your **Callback Public Key**.
- Decrypt using the corresponding **Callback Private Key**.

---

## Callback Types

| `type` | Description |
|--------|-------------|
| `TRANSACTION` | Regular transaction approval |
| `MPC_SIGN` | MPC raw signing approval |
| `WEB3_SIGN` | Web3 signing approval |

---

## TRANSACTION_APPROVAL Payload Key Fields

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | string | Safeheron transaction key |
| `customerRefId` | string | Your reference ID |
| `coinKey` | string | Coin identifier |
| `txAmount` | string | Transaction amount |
| `sourceAccountKey` | string | Sender wallet key |
| `destinationAddress` | string | Recipient address |
| `destinationAccountType` | string | e.g. `ONE_TIME_ADDRESS` |
| `amlList` | Array | AML assessments (provider, riskLevel, etc.) |

---

## Approval Callback Response

Your callback service must return an **encrypted response**:

```json
{
  "approveStatus": "APPROVE",
  "customerRefId": "<echo-back-from-request>",
  "rejectReason":  ""
}
```

| Field | Type | Values |
|-------|------|--------|
| `approveStatus` | string | `APPROVE` or `REJECT` |
| `customerRefId` | string | Echo back the incoming `customerRefId` |
| `rejectReason` | string | Optional reason when rejecting |

---

## Approval Logic Examples

### Allow Only Whitelisted Destinations

```typescript
function evaluateTransaction(payload: any): string {
  const allowedAddresses = loadWhitelistedAddresses(payload.coinKey);
  if (!allowedAddresses.has(payload.destinationAddress)) {
    return 'REJECT';
  }
  return 'APPROVE';
}
```

### Amount Limit per Account

```typescript
function evaluateTransaction(payload: any): string {
  const amount = parseFloat(payload.txAmount);
  const limit = getAccountLimit(payload.sourceAccountKey);
  if (amount > limit) {
    return 'REJECT';
  }
  return 'APPROVE';
}
```

### AML Risk Check

```typescript
function evaluateTransaction(payload: any): string {
  for (const aml of payload.amlList || []) {
    if (aml.riskLevel === 'HIGH') {
      return 'REJECT';
    }
  }
  return 'APPROVE';
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
| KMS support | AWS KMS, GCP KMS, Alibaba Cloud KMS |
| CLI commands | `sudo ./cosigner start`, `sudo ./cosigner stop`, `sudo ./cosigner logs` |
| Export Co-Signer public key | `sudo ./cosigner export-public-key` |

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
| `The final private key fragment is not exists` | Co-Signer not yet activated | Normal -- proceed with activation workflow |
| Transaction stays in `WAIT_AUDIT` | Co-Signer not running or not polling | Check `./cosigner logs -f` for errors |
| `Timestamp out of range` | Server clock skew > 5s | Sync Co-Signer server time with NTP |
| Docker login failure | Token expired or firewall blocks `registry.gitlab.com` | Re-download CLI from Console; check firewall rules |

---

## Security Deployment Requirements (API Co-Signer Host)

### Mandatory Security Principles

| Principle | Requirement |
|-----------|------------|
| Strong passwords | All accounts must use randomly generated strong passwords |
| MFA everywhere | Enable 2FA/MFA on every account and service |
| Minimum privilege | Each account/role has only the permissions needed |
| Minimum exposure | Close all unnecessary ports. Only port `9999` needs to be open |
| Secure secret storage | Store all secrets in a dedicated secrets manager |

### Host Security Controls

| Control | Description |
|---------|-------------|
| Network isolation | Deploy Co-Signer in a **private isolated network**. No public internet inbound access |
| IP whitelist | Only Safeheron API gateway and your internal systems can reach the Co-Signer host |
| MFA (host login) | All host access requires MFA |

### Callback Service Security

The Approval Callback Service must:

1. **Verify the Co-Signer identity signature** on every request before processing
2. **Only accept connections from the Co-Signer host IP**
3. **Validate `customerRefId`** -- reject any callback for a `customerRefId` not found in your DB
4. **Validate amount and destination** -- cross-check against the original withdrawal order
5. **Respond within 5 seconds** -- Co-Signer will time out otherwise (sync server time via NTP)

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

## Approval Callback -- Python SDK Integration

### Configure CoSignerConverter

```python
from safeheron_api_sdk_python.cosigner.co_signer_converter import (
    CoSignerConverter,
    CoSignerResponseV3,
)

# Config for Co-Signer callback
cosigner_config = {
    'coSignerPubKey': 'MIICIjANBgkqhk...',  # Co-Signer identity public key (from export-public-key)
    'approvalCallbackServicePrivateKey': 'MIIJQgIBADANBgk...',  # Your callback private key (base64)
    # OR use PEM file:
    # 'approvalCallbackServicePrivateKeyPemFile': '/path/to/callback_private.pem',
}

converter = CoSignerConverter(cosigner_config)
```

### Flask Approval Callback Service (V3 -- Recommended)

```python
from flask import Flask, request, jsonify
import json
from decimal import Decimal

app = Flask(__name__)

@app.route('/cosigner/callback', methods=['POST'])
def handle_cosigner_callback():
    try:
        raw_body = request.get_json(force=True)

        # Decrypt and verify signature (V3 protocol)
        biz_content = converter.request_v3_convert(raw_body)

        callback_type = biz_content.get('type', '')
        customer_content = biz_content.get('customerContent', {})

        # Determine approval based on business logic
        action = evaluate_approval(callback_type, customer_content)

        # Build encrypted response
        response = CoSignerResponseV3()
        response.action = action  # "APPROVE" or "REJECT"
        response.approvalId = biz_content.get('approvalId', '')

        encrypted_response = converter.response_v3_converter(response)
        return jsonify(encrypted_response)

    except Exception as e:
        app.logger.error(f"Callback processing error: {e}")
        # On error, reject by default for safety
        response = CoSignerResponseV3()
        response.action = "REJECT"
        response.approvalId = ""
        encrypted_response = converter.response_v3_converter(response)
        return jsonify(encrypted_response)


def evaluate_approval(callback_type, content):
    """Implement your business validation logic here."""

    if callback_type == 'TRANSACTION':
        return evaluate_transaction_approval(content)
    elif callback_type == 'MPC_SIGN':
        return evaluate_mpc_sign_approval(content)
    elif callback_type == 'WEB3_SIGN':
        return evaluate_web3_sign_approval(content)

    # Unknown type -- reject
    return "REJECT"


def evaluate_transaction_approval(tx):
    customer_ref_id = tx.get('customerRefId', '')

    # 1. Verify customerRefId exists in your DB
    order = find_order_by_ref_id(customer_ref_id)
    if order is None:
        app.logger.warning(f"REJECT: unknown customerRefId {customer_ref_id}")
        return "REJECT"

    # 2. Verify amount matches
    if Decimal(tx.get('txAmount', '0')) != order['expected_amount']:
        app.logger.warning(f"REJECT: amount mismatch for {customer_ref_id}")
        return "REJECT"

    # 3. Verify destination address matches
    if tx.get('destinationAddress', '').lower() != order['destination_address'].lower():
        app.logger.warning(f"REJECT: destination mismatch for {customer_ref_id}")
        return "REJECT"

    # 4. Check AML risk
    for aml in tx.get('amlList', []):
        if aml.get('riskLevel', '').upper() == 'HIGH':
            return "REJECT"

    return "APPROVE"
```

---

## Callback Types

| `type` | Description |
|--------|-------------|
| `TRANSACTION` | Regular transaction approval |
| `MPC_SIGN` | MPC raw signing approval |
| `WEB3_SIGN` | Web3 signing approval |

---

## TRANSACTION Payload (`customerContent`)

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | str | Safeheron transaction key |
| `customerRefId` | str | Your reference ID |
| `coinKey` | str | Coin identifier |
| `txAmount` | str | Transaction amount |
| `transactionStatus` | str | Current status |
| `sourceAccountKey` | str | Sender wallet key |
| `sourceAccountType` | str | e.g. `VAULT_ACCOUNT` |
| `destinationAccountKey` | str | Recipient wallet key |
| `destinationAccountType` | str | e.g. `ONE_TIME_ADDRESS` |
| `destinationAddress` | str | Recipient address |
| `amlLock` | str | AML status: `YES` / `NO` |
| `amlList` | list | AML assessments |

---

## Approval Callback Response

Your callback service must return an **encrypted response**. The `CoSignerResponseV3` class and `response_v3_converter()` method handle the encryption automatically.

| Field | Type | Values |
|-------|------|--------|
| `action` | str | `APPROVE` or `REJECT` |
| `approvalId` | str | Echo back the incoming `approvalId` |

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
| Update Callback URL | Web Console update takes effect within 5 min -- no Co-Signer restart needed |
| IP whitelist | Add Co-Signer host IP to the API Key IP whitelist in Console |

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
| **Co-Signer Identity Public Key** | Exported via `export-public-key` | You use this to verify callback request signatures (`coSignerPubKey`) |
| **Callback Public Key** | You generate & upload to Console | Safeheron Co-Signer encrypts the callback `key` field with this |
| **Callback Private Key** | You hold securely | Your callback service uses this to decrypt (`approvalCallbackServicePrivateKey`) |

---

## Security Deployment Requirements (API Co-Signer Host)

### Mandatory Security Principles

| Principle | Requirement |
|-----------|------------|
| Strong passwords | All accounts must use randomly generated strong passwords |
| MFA everywhere | Enable 2FA/MFA on every account and service |
| Minimum privilege | Each account/role has only needed permissions |
| Minimum exposure | Close all unnecessary ports. Only port `9999` (business) needs to be open |
| Secure secret storage | Store all secrets in a dedicated secrets manager |

### Callback Service Security

1. **Verify the Co-Signer identity signature** on every request before processing
2. **Only accept connections from the Co-Signer host IP**
3. **Validate `customerRefId`** -- reject any callback for a `customerRefId` not found in your DB
4. **Validate amount and destination** -- cross-check against the original withdrawal order
5. **Respond within 5 seconds** -- Co-Signer will time out otherwise (sync server time via NTP)

---

## Common Co-Signer Issues

| Issue | Cause | Resolution |
|-------|-------|------------|
| `Illegal IP` in logs | Co-Signer host IP not in API Key whitelist | Add IP to whitelist in Console, restart Co-Signer |
| Transaction stays in `WAIT_AUDIT` | Co-Signer not running or not polling | Check `./cosigner logs -f` for errors |
| `Timestamp out of range` | Server clock skew > 5s | Sync Co-Signer server time with NTP |
| Docker login failure | Token expired or firewall blocks `registry.gitlab.com` | Re-download CLI from Console; check firewall rules |

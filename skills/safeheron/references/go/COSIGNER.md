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

## Approval Callback Service (Go Implementation)

### Using the SDK's CoSignerConverter

```go
import (
    "github.com/Safeheron/safeheron-api-sdk-go/safeheron/cosigner"
)

coSignerConverter := cosigner.CoSignerConverter{Config: cosigner.CoSignerConfig{
    CoSignerPubKey:                    "pems/cosigner_public.pem",
    ApprovalCallbackServicePrivateKey: "pems/callback_private.pem",
}}

// Decrypt and verify the callback request
bizContent, err := coSignerConverter.RequestV3Convert(callbackPayload)
if err != nil {
    log.Printf("Co-Signer verification failed: %v", err)
    return
}

// Build encrypted response
response := cosigner.CoSignerResponseV3{
    ApprovalId: bizContent.ApprovalId,
    Action:     "APPROVE", // or "REJECT"
}
encryptedResp, err := coSignerConverter.ResponseV3Converter(response)
if err != nil {
    log.Printf("Failed to encrypt response: %v", err)
    return
}
// Return encryptedResp to the Co-Signer
```

### Full net/http Handler

```go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "log"
    "net/http"

    "github.com/Safeheron/safeheron-api-sdk-go/safeheron/cosigner"
    "github.com/shopspring/decimal"
)

var coSignerConverter cosigner.CoSignerConverter

func init() {
    coSignerConverter = cosigner.CoSignerConverter{Config: cosigner.CoSignerConfig{
        CoSignerPubKey:                    "pems/cosigner_public.pem",
        ApprovalCallbackServicePrivateKey: "pems/callback_private.pem",
    }}
}

func callbackHandler(w http.ResponseWriter, r *http.Request) {
    body, err := io.ReadAll(r.Body)
    if err != nil {
        http.Error(w, "Bad request", http.StatusBadRequest)
        return
    }
    defer r.Body.Close()

    // Parse callback payload
    var callbackPayload cosigner.CoSignerCallBackV3
    if err := json.Unmarshal(body, &callbackPayload); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }

    // Decrypt and verify Co-Signer identity signature
    bizContent, err := coSignerConverter.RequestV3Convert(callbackPayload)
    if err != nil {
        log.Printf("Co-Signer verification failed: %v", err)
        http.Error(w, "Verification failed", http.StatusForbidden)
        return
    }

    // Business validation logic
    action := evaluateTransaction(bizContent)

    // Build encrypted response
    response := cosigner.CoSignerResponseV3{
        ApprovalId: bizContent.ApprovalId,
        Action:     action,
    }
    encryptedResp, err := coSignerConverter.ResponseV3Converter(response)
    if err != nil {
        log.Printf("Failed to encrypt response: %v", err)
        http.Error(w, "Internal error", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(encryptedResp)
}

func evaluateTransaction(bizContent string) string {
    // Parse the decrypted business content
    var content map[string]interface{}
    json.Unmarshal([]byte(bizContent), &content)

    customerRefId, _ := content["customerRefId"].(string)

    // 1. Verify customerRefId exists in your DB
    order := findOrderByCustomerRefId(customerRefId)
    if order == nil {
        log.Printf("REJECT: unknown customerRefId %s", customerRefId)
        return "REJECT"
    }

    // 2. Verify amount matches
    txAmount, _ := decimal.NewFromString(content["txAmount"].(string))
    if !txAmount.Equal(order.ExpectedAmount) {
        log.Printf("REJECT: amount mismatch for %s", customerRefId)
        return "REJECT"
    }

    // 3. Verify destination address matches
    destAddress, _ := content["destinationAddress"].(string)
    if destAddress != order.DestinationAddress {
        log.Printf("REJECT: destination mismatch for %s", customerRefId)
        return "REJECT"
    }

    // 4. Check AML risk
    if amlList, ok := content["amlList"].([]interface{}); ok {
        for _, aml := range amlList {
            amlMap, _ := aml.(map[string]interface{})
            if riskLevel, _ := amlMap["riskLevel"].(string); riskLevel == "HIGH" {
                return "REJECT"
            }
        }
    }

    return "APPROVE"
}

func main() {
    http.HandleFunc("/cosigner/callback", callbackHandler)
    fmt.Println("Callback service listening on :9090")
    log.Fatal(http.ListenAndServe(":9090", nil))
}
```

---

## Callback Types

| `type` | Description |
|--------|-------------|
| `TRANSACTION` | Regular transaction approval |
| `MPC_SIGN` | MPC raw signing approval |
| `WEB3_SIGN` | Web3 signing approval |

---

## TRANSACTION_APPROVAL Payload (`customerContent`)

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | `string` | Safeheron transaction key |
| `customerRefId` | `string` | Your reference ID |
| `coinKey` | `string` | Coin identifier |
| `txAmount` | `string` | Transaction amount |
| `sourceAccountKey` | `string` | Sender wallet key |
| `destinationAddress` | `string` | Recipient address |
| `amlList` | `[]Aml` | AML assessments |
| `createTime` | `int64` | Unix timestamp (ms) |

---

## WEB3_SIGN_APPROVAL Payload

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | `string` | Web3 sign request key |
| `customerRefId` | `string` | Your reference ID |
| `subjectType` | `string` | `ETH_SIGN`, `PERSONAL_SIGN`, `ETH_SIGNTYPEDDATA`, `ETH_SIGNTRANSACTION` |
| `accountKey` | `string` | Web3 wallet account key |

---

## MPC_SIGN_APPROVAL Payload

| Field | Type | Description |
|-------|------|-------------|
| `txKey` | `string` | MPC sign request key |
| `customerRefId` | `string` | Your reference ID |
| `signAlg` | `string` | `Secp256k1` or `Ed25519` |
| `dataList` | `[]struct` | Raw data list: `[{note, data}]` |

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

## Key Pair Configuration Summary

| Key | Owner | Purpose |
|-----|-------|---------|
| **Co-Signer Identity Private Key** | Safeheron (Co-Signer internal) | Signs callback requests sent to your service |
| **Co-Signer Identity Public Key** | Exported via `export-public-key` | You use this to verify callback request signatures |
| **Callback Public Key** | You generate & upload to Console | Safeheron Co-Signer encrypts the callback `key` field with this |
| **Callback Private Key** | You hold securely | Your callback service uses this to decrypt the AES key |

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

## Security Deployment Requirements

- The Co-Signer host must be in a **private isolated network** -- never exposed to the public internet.
- The Approval Callback Service must only accept requests from the Co-Signer host IP.
- Verify the Co-Signer identity signature on every request before processing.
- Validate `customerRefId` -- reject any callback for a `customerRefId` not found in your DB.
- Validate amount and destination -- cross-check against the original withdrawal order.
- Respond within 5 seconds -- Co-Signer will time out otherwise (sync server time via NTP).

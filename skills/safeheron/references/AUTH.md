# Safeheron API Authentication

## Request Encryption & Signing (Step-by-Step)

### Step 1: Prepare bizContent
Serialize your business parameters to a JSON string.

### Step 2: Generate AES key
Randomly generate:
- **32 bytes** AES key
- **16 bytes** IV (initialization vector)

### Step 3: Encrypt bizContent
```
Algorithm: AES/GCM/NoPadding
Input: JSON string
Output: base64(AES_GCM_Encrypt(jsonString, key, iv))  → bizContent field
```

### Step 4: Encrypt the AES key
```
Algorithm: RSA/ECB/OAEPWithSHA-256AndMGF1Padding
Input: 48 bytes (AES key + IV), encrypted with Safeheron's RSA Public Key
Output: base64(RSA_Encrypt(aesKey + iv))  → key field
```

### Step 5: Build signature string
Sort all parameters (excluding `rsaType` and `aesType`) by key ascending:
```
apiKey=...&bizContent=...&key=...&timestamp=...
```

### Step 6: Sign
```
Algorithm: SHA256WithRSA
Input: Signature string, signed with YOUR RSA Private Key
Output: base64(RSA_Sign(sigString))  → sig field
```

### Final Request Body
```json
{
  "apiKey": "YOUR_API_KEY",
  "timestamp": "1623038312088",
  "bizContent": "<aes-encrypted-base64>",
  "key": "<rsa-encrypted-aes-key-base64>",
  "sig": "<rsa-signature-base64>",
  "rsaType": "ECB_OAEP",
  "aesType": "GCM_NOPADDING"
}
```

---

## Response Verification & Decryption

### Step 1: Build response signature string
Sort by key ascending (exclude `rsaType`, `aesType`):
```
bizContent=...&code=...&key=...&message=...&timestamp=...
```

### Step 2: Verify signature
```
Algorithm: SHA256WithRSA
Verify using: Safeheron's RSA Public Key + sig field
```
If verification fails, reject the response.

### Step 3: Decrypt the AES key
```
Algorithm: RSA/ECB/OAEPWithSHA-256AndMGF1Padding
Decrypt key field using YOUR RSA Private Key
→ Yields 48-byte AES key+IV
```

### Step 4: Decrypt bizContent
```
Algorithm: AES/GCM/NoPadding
Decrypt bizContent using the recovered AES key and IV
→ Yields plaintext JSON
```

---

## Java SDK: Config Object
```java
SafeheronConfig config = new SafeheronConfig();
config.setApiKey("YOUR_API_KEY");
config.setPrivateKey("YOUR_BASE64_RSA_PRIVATE_KEY");
config.setSafeheronPublicKey("SAFEHERON_RSA_PUBLIC_KEY_BASE64");
config.setBaseUrl("https://api.safeheron.vip");
config.setRequestTimeout(20000);
```

The SDK handles all encryption/decryption/signing automatically.

---

## OpenSSL Key Generation
```bash
# Generate 4096-bit RSA private key
openssl genpkey -out api_private.pem -algorithm RSA -pkeyopt rsa_keygen_bits:4096

# Extract public key
openssl rsa -in api_private.pem -out api_public.pem -pubout

# Get base64-encoded private key for SDK config
openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt -in api_private.pem -out api_pkcs8.pem
```

Upload `api_public.pem` to the Safeheron Console. Use `api_pkcs8.pem` contents (stripped of headers) as `privateKey` in SDK config.

---

## Security Checklist
- [ ] RSA private key stored securely (never in source code)
- [ ] IP address registered in Console whitelist
- [ ] Always verify response `sig` before trusting `bizContent`
- [ ] Use HTTPS only (`https://api.safeheron.vip`)
- [ ] Rotate API keys periodically

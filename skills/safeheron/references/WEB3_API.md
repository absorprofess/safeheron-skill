# Web3 API Reference

Web3 Wallet Accounts support arbitrary EVM signing: raw messages, typed data (EIP-712), and raw EVM transactions.

## Imports
```java
import com.safeheron.client.api.Web3ApiService;
import com.safeheron.client.config.SafeheronConfig;
import com.safeheron.client.request.*;
import com.safeheron.client.response.*;
import com.safeheron.client.utils.ServiceCreator;
import com.safeheron.client.utils.ServiceExecutor;
```

## Create API Service
```java
Web3ApiService web3Api = ServiceCreator.create(Web3ApiService.class, safeheronConfig);
```

---

## Notes on Web3 Service

The exact request/response class names for Web3 operations (ethSign, personalSign, ethSignTypedData, ethSignTransaction) follow the same package conventions:
- Requests: `com.safeheron.client.request.*`
- Responses: `com.safeheron.client.response.*`
- All calls: `ServiceExecutor.execute(web3Api.someMethod(request))`

For authoritative class names, see the official source:
https://github.com/Safeheron/safeheron-api-sdk-java/tree/main/src/main/java/com/safeheron/client

---

## General Pattern for All Web3 Calls
```java
// 1. Create request object (class name matches the operation)
SomeWeb3Request req = new SomeWeb3Request();
req.setCustomerRefId(UUID.randomUUID().toString());
req.setAccountKey(web3AccountKey);
// ... set operation-specific fields

// 2. Execute
SomeWeb3Response resp = ServiceExecutor.execute(web3Api.someMethod(req));
String txKey = resp.getTxKey();

// 3. Poll for result
// ... poll with a "getOne" method until status == SUCCESS
// 4. Get signature from response
String signature = resp.getSignature();
```

---

## Web3 Sign Status Flow
```
SUBMITTED → WAIT_AUDIT → WAIT_SIGN → SUCCESS
                                    └─ FAILED | REVOKED | REJECTED
```

---

## Supported Sign Types
| Operation | Description |
|-----------|-------------|
| ethSign | Sign a 32-byte raw hash |
| personalSign | Sign an arbitrary message with Ethereum prefix |
| ethSignTypedData | Sign EIP-712 structured data (V1/V3/V4) |
| ethSignTransaction | Sign a raw EVM transaction |

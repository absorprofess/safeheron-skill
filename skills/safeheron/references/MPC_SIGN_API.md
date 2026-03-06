# MPC Sign API Reference

## Imports
```java
import com.safeheron.client.api.MPCSignApiService;
import com.safeheron.client.config.SafeheronConfig;
import com.safeheron.client.request.CreateMPCSignTransactionRequest;
import com.safeheron.client.request.OneMPCSignTransactionsRequest;
import com.safeheron.client.response.MPCSignTransactionsResponse;
import com.safeheron.client.response.TxKeyResult;
import com.safeheron.client.utils.ServiceCreator;
import com.safeheron.client.utils.ServiceExecutor;
```

## Create API Service
```java
MPCSignApiService mpcSignApi = ServiceCreator.create(MPCSignApiService.class, safeheronConfig);
```

---

## Create an MPC Sign Transaction
```java
CreateMPCSignTransactionRequest request = new CreateMPCSignTransactionRequest();
request.setCustomerRefId(UUID.randomUUID().toString());
request.setSourceAccountKey(accountKey);
request.setSignAlg("Secp256k1");  // "Secp256k1" or "Ed25519"

// Add hash(es) to sign — use the inner Date class
CreateMPCSignTransactionRequest.Date hashItem = new CreateMPCSignTransactionRequest.Date();
hashItem.setData(hashHexWithoutPrefix);  // 32-byte hex, no "0x" prefix
request.setDataList(Arrays.asList(hashItem));

TxKeyResult response = ServiceExecutor.execute(mpcSignApi.createMPCSignTransactions(request));
String txKey = response.getTxKey();
```

## Poll for MPC Sign Result
```java
OneMPCSignTransactionsRequest pollRequest = new OneMPCSignTransactionsRequest();
pollRequest.setCustomerRefId(customerRefId);   // OR setTxKey(txKey)

for (int i = 0; i < 100; i++) {
    MPCSignTransactionsResponse resp = ServiceExecutor.execute(
            mpcSignApi.oneMPCSignTransactions(pollRequest));

    String status = resp.getTransactionStatus();
    String subStatus = resp.getTransactionSubStatus();
    System.out.println("status: " + status + ", sub: " + subStatus);

    if ("FAILED".equalsIgnoreCase(status) || "REJECTED".equalsIgnoreCase(status)) {
        throw new RuntimeException("MPC sign failed/rejected");
    }
    if ("COMPLETED".equalsIgnoreCase(status) && "CONFIRMED".equalsIgnoreCase(subStatus)) {
        String sig = resp.getDataList().get(0).getSig();
        // sig = R (64 hex chars) + S (64 hex chars) + V (2 hex chars)
        break;
    }
    Thread.sleep(5000);
}
```

---

## Key Classes
| Class | Description |
|-------|-------------|
| `CreateMPCSignTransactionRequest` | Request to initiate MPC signing |
| `CreateMPCSignTransactionRequest.Date` | Inner class for each hash item (field: `data`) |
| `TxKeyResult` | Response from create — contains `txKey` |
| `OneMPCSignTransactionsRequest` | Request to query one MPC sign tx |
| `MPCSignTransactionsResponse` | Query response — contains status and signed data list |

---

## Signature Format
The `sig` field from `resp.getDataList().get(0).getSig()` is a concatenated hex string:
- Characters 0–63: `R`
- Characters 64–127: `S`
- Characters 128–129: `V` (hex, add 27 for legacy Ethereum)

```java
private Sign.SignatureData convertSig(String sig) {
    String sigR = sig.substring(0, 64);
    String sigS = sig.substring(64, 128);
    String sigV = sig.substring(128);
    Integer v = Integer.parseInt(sigV, 16) + 27;
    return new Sign.SignatureData(
            v.byteValue(),
            Numeric.hexStringToByteArray(sigR),
            Numeric.hexStringToByteArray(sigS));
}
```

---

## Full ERC-20 Transfer via MPC Sign (Web3j)

This is the exact pattern from the official SDK test (`MpcSignTest.java`):

```java
// 1. Build ERC-20 transfer function
Function function = new Function(
    "transfer",
    Arrays.asList(new Address(toAddress), new Uint256(tokenValue.toBigInteger())),
    Arrays.asList(new TypeReference<Type>() {}));
String data = FunctionEncoder.encode(function);

// 2. Build EIP-1559 raw transaction (web3j)
RawTransaction rawTx = RawTransaction.createTransaction(
    chainId.longValue(), nonce, gasLimit, contractAddress,
    BigInteger.ZERO, data, maxPriorityFeePerGas, maxFeePerGas);

// 3. Hash the encoded transaction
byte[] encodedRawTx = TransactionEncoder.encode(rawTx);
String hash = Numeric.toHexString(Hash.sha3(encodedRawTx));

// 4. Submit hash (without "0x") to Safeheron MPC Sign
String customerRefId = UUID.randomUUID().toString();
CreateMPCSignTransactionRequest req = new CreateMPCSignTransactionRequest();
req.setCustomerRefId(customerRefId);
req.setSourceAccountKey(accountKey);
req.setSignAlg("Secp256k1");
CreateMPCSignTransactionRequest.Date hashItem = new CreateMPCSignTransactionRequest.Date();
hashItem.setData(hash.substring(2));  // strip "0x"
req.setDataList(Arrays.asList(hashItem));
TxKeyResult txKeyResult = ServiceExecutor.execute(mpcSignApi.createMPCSignTransactions(req));

// 5. Poll until COMPLETED/CONFIRMED
OneMPCSignTransactionsRequest pollReq = new OneMPCSignTransactionsRequest();
pollReq.setCustomerRefId(customerRefId);
MPCSignTransactionsResponse pollResp = null;
for (int i = 0; i < 100; i++) {
    pollResp = ServiceExecutor.execute(mpcSignApi.oneMPCSignTransactions(pollReq));
    if ("COMPLETED".equalsIgnoreCase(pollResp.getTransactionStatus())
            && "CONFIRMED".equalsIgnoreCase(pollResp.getTransactionSubStatus())) break;
    Thread.sleep(5000);
}
String sig = pollResp.getDataList().get(0).getSig();

// 6. Assemble signed transaction and broadcast
byte[] signedMessage = TransactionEncoder.encode(rawTx, convertSig(sig));
String ethTxHash = web3j.ethSendRawTransaction(Numeric.toHexString(signedMessage))
        .send().getTransactionHash();
```

---

## Sign Algorithm Values
| Value | Curve | Use Case |
|-------|-------|----------|
| `"Secp256k1"` | secp256k1 | Ethereum, Bitcoin, BSC, Polygon |
| `"Ed25519"` | Ed25519 | Solana, Near, Aptos |

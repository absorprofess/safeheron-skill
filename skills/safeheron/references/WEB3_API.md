# Web3 API & MPC Sign Reference

## Web3 API

Web3 Wallet Accounts support arbitrary EVM signing operations: raw messages, typed data (EIP-712), and raw transactions.

### Create a Web3 Wallet Account
```
POST /v1/web3/account/create
```
```java
CreateWeb3AccountRequest req = new CreateWeb3AccountRequest();
req.setAccountName("DeFi Signer");
CreateWeb3AccountResponse resp = web3Service.createAccount(req);
String web3AccountKey = resp.getAccountKey();
String ethAddress = resp.getEthAddress();
```

### List Web3 Wallet Accounts
```
POST /v1/web3/account/list
```

### Retrieve a Single Web3 Wallet Account
```
POST /v1/web3/account/one
```

---

### ethSign (sign raw hash)
```
POST /v1/web3/sign/eth
```
```java
CreateEthSignRequest req = new CreateEthSignRequest();
req.setCustomerRefId(UUID.randomUUID().toString());
req.setAccountKey(web3AccountKey);
req.setDataToSign("0xDeadBeef...");  // 32-byte hex hash
CreateEthSignResponse resp = web3Service.createEthSign(req);
String signTxKey = resp.getTxKey();
```

### personalSign (sign message with prefix)
```
POST /v1/web3/sign/personal
```
```java
CreatePersonalSignRequest req = new CreatePersonalSignRequest();
req.setCustomerRefId(UUID.randomUUID().toString());
req.setAccountKey(web3AccountKey);
req.setMessage("Hello Safeheron");  // UTF-8 string
CreatePersonalSignResponse resp = web3Service.createPersonalSign(req);
```

### ethSignTypedData (EIP-712)
```
POST /v1/web3/sign/typedData
```
```java
CreateEthSignTypedDataRequest req = new CreateEthSignTypedDataRequest();
req.setCustomerRefId(UUID.randomUUID().toString());
req.setAccountKey(web3AccountKey);
req.setTypedDataJson("{\"types\":{...},\"domain\":{...},\"message\":{...}}");
req.setVersion("V4");  // V1 | V3 | V4
CreateEthSignTypedDataResponse resp = web3Service.createEthSignTypedData(req);
```

### ethSignTransaction (sign raw EVM tx)
```
POST /v1/web3/sign/transaction
```
```java
CreateEthSignTransactionRequest req = new CreateEthSignTransactionRequest();
req.setCustomerRefId(UUID.randomUUID().toString());
req.setAccountKey(web3AccountKey);
req.setTo("0xContractAddress");
req.setValue("0");            // in Wei as string
req.setNonce("42");
req.setGasLimit("100000");
req.setGasPrice("20000000000");
req.setData("0xABCDEF...");   // encoded calldata
req.setChainId("1");          // 1=mainnet, 137=polygon, etc.
CreateEthSignTransactionResponse resp = web3Service.createEthSignTransaction(req);
```

### Retrieve a Web3 Signature
```
POST /v1/web3/sign/one
```
```java
GetWeb3SignRequest req = new GetWeb3SignRequest();
req.setTxKey(signTxKey);
GetWeb3SignResponse resp = web3Service.getSign(req);
String signature = resp.getSignature();
String status = resp.getTransactionStatus();  // SUCCESS when signed
```

### Cancel Signature
```
POST /v1/web3/sign/cancel
```

### Web3 Sign Transaction List
```
POST /v1/web3/sign/list
```

---

## Web3 Sign Status Flow
```
SUBMITTED → WAIT_AUDIT → WAIT_SIGN → SUCCESS
                                    └─ FAILED | REVOKED | REJECTED
```

---

## MPC Sign API

MPC Sign allows arbitrary raw message signing using MPC key shares, without involving Safeheron's transaction routing.

### Create MPC Sign Wallet Account
```
POST /v1/mpcsign/account/create
```
Same pattern as regular account creation; returns `accountKey`.

### Create an MPC Sign Transaction
```
POST /v1/mpcsign/create
```
```java
CreateMpcSignRequest req = new CreateMpcSignRequest();
req.setCustomerRefId(UUID.randomUUID().toString());
req.setAccountKey(mpcAccountKey);
req.setSignAlg("SECP256K1");        // SECP256K1 | ED25519
req.setDataList(Arrays.asList(
    new MpcSignData("hash1_hex", "note1"),
    new MpcSignData("hash2_hex", "note2")
));
CreateMpcSignResponse resp = mpcService.createMpcSign(req);
String mpctTxKey = resp.getTxKey();
```

### Retrieve a Single MPC Sign Transaction
```
POST /v1/mpcsign/one
```
```java
GetMpcSignRequest req = new GetMpcSignRequest();
req.setTxKey(mpctTxKey);
GetMpcSignResponse resp = mpcService.getMpcSign(req);
String status = resp.getTransactionStatus();
// On SUCCESS:
List<SignedData> results = resp.getSignedDataList();
results.forEach(d -> {
    System.out.println("r: " + d.getR());
    System.out.println("s: " + d.getS());
    System.out.println("v: " + d.getV());
});
```

### MPC Sign Transaction List
```
POST /v1/mpcsign/list
```

---

## ERC-20 Token Transfer via MPC Sign (Full Example)

This pattern uses Web3j + Safeheron MPC Sign to transfer tokens without exposing private keys:

```java
// 1. Build ERC-20 transfer calldata
Function function = new Function(
    "transfer",
    Arrays.asList(new Address(toAddress), new Uint256(amount)),
    Collections.emptyList()
);
String encodedFunction = FunctionEncoder.encode(function);

// 2. Build unsigned transaction
RawTransaction rawTx = RawTransaction.createTransaction(
    nonce, gasPrice, gasLimit,
    erc20ContractAddress, BigInteger.ZERO, encodedFunction
);
byte[] encodedTx = TransactionEncoder.encode(rawTx, chainId);
byte[] txHash = Hash.sha3(encodedTx);

// 3. Send hash to Safeheron MPC Sign
CreateMpcSignRequest req = new CreateMpcSignRequest();
req.setCustomerRefId(UUID.randomUUID().toString());
req.setAccountKey(accountKey);
req.setSignAlg("SECP256K1");
req.setDataList(Arrays.asList(new MpcSignData(Numeric.toHexString(txHash), "erc20-transfer")));
mpcService.createMpcSign(req);

// 4. Poll for completion and recover signature
// ... poll getMpcSign until SUCCESS ...
SignedData sig = resp.getSignedDataList().get(0);

// 5. Assemble and broadcast signed transaction
Sign.SignatureData sigData = new Sign.SignatureData(
    Numeric.hexStringToByteArray(sig.getV()),
    Numeric.hexStringToByteArray(sig.getR()),
    Numeric.hexStringToByteArray(sig.getS())
);
byte[] signedTx = TransactionEncoder.encode(rawTx, sigData);
String txHash = web3j.ethSendRawTransaction(Numeric.toHexString(signedTx)).send().getTransactionHash();
```

# Tools API Reference (AML Address Checker)

## Overview

The Tools API provides utilities for address-level AML risk assessment *before* initiating a transaction. This is distinct from transaction-level KYT reports (see [COMPLIANCE_API.md](COMPLIANCE_API.md)).

Use Case: Check whether a recipient address is flagged as high-risk **before** sending funds.

---

## Imports

```java
import com.safeheron.client.api.ToolsApiService;
import com.safeheron.client.config.SafeheronConfig;
import com.safeheron.client.request.AmlCheckerRequestRequest;
import com.safeheron.client.request.AmlCheckerRetrievesRequest;
import com.safeheron.client.response.AmlCheckerRequestResponse;
import com.safeheron.client.response.AmlCheckerRetrievesResponse;
import com.safeheron.client.utils.ServiceCreator;
import com.safeheron.client.utils.ServiceExecutor;
```

## Create API Service

```java
ToolsApiService toolsApi = ServiceCreator.create(ToolsApiService.class, safeheronConfig);
```

---

## AML Address Risk Assessment (Two-Step Async Flow)

Assessment is asynchronous: submit the address, then poll for results.

### Step 1: Submit Address for AML Assessment

```java
AmlCheckerRequestRequest req = new AmlCheckerRequestRequest();
req.setNetwork("Ethereum");   // "Bitcoin", "Ethereum", or "Tron"
req.setAddress("0xAbCd1234..."); // Address to assess

AmlCheckerRequestResponse resp = ServiceExecutor.execute(toolsApi.amlCheckerRequest(req));
String requestId = resp.getRequestId();  // save this for polling
```

### AmlCheckerRequestRequest Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `network` | Yes | String | Blockchain network: `Bitcoin`, `Ethereum`, `Tron` |
| `address` | Yes | String | Blockchain address to assess |

### AmlCheckerRequestResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `requestId` | String | Async assessment request ID — use to poll for results |

---

### Step 2: Retrieve AML Assessment Result

```java
AmlCheckerRetrievesRequest req = new AmlCheckerRetrievesRequest();
req.setRequestId(requestId);

AmlCheckerRetrievesResponse resp = ServiceExecutor.execute(toolsApi.amlCheckerRetrieves(req));
System.out.println("Risk Level: " + resp.getRiskLevel());
System.out.println("Provider:   " + resp.getProvider());
System.out.println("Status:     " + resp.getStatus());
```

### AmlCheckerRetrievesResponse Fields

| Field | Type | Description |
|-------|------|-------------|
| `requestId` | String | The assessment request ID |
| `network` | String | Blockchain network |
| `address` | String | Assessed address |
| `provider` | String | AML provider name |
| `status` | String | Assessment status (PENDING / COMPLETED / FAILED) |
| `riskLevel` | String | `HIGH`, `MEDIUM`, `LOW`, `UNKNOWN` |
| `payload` | Object | Raw provider-specific risk data |
| `createTime` | Long | Unix timestamp (ms) when submitted |
| `updateTime` | Long | Unix timestamp (ms) of last update |

---

## Full Example: Check Address Before Sending

```java
// 1. Submit address for assessment
AmlCheckerRequestRequest submitReq = new AmlCheckerRequestRequest();
submitReq.setNetwork("Ethereum");
submitReq.setAddress(destinationAddress);
AmlCheckerRequestResponse submitResp = ServiceExecutor.execute(toolsApi.amlCheckerRequest(submitReq));
String requestId = submitResp.getRequestId();

// 2. Poll until result is ready
AmlCheckerRetrievesRequest pollReq = new AmlCheckerRetrievesRequest();
pollReq.setRequestId(requestId);
AmlCheckerRetrievesResponse result = null;
for (int i = 0; i < 30; i++) {
    result = ServiceExecutor.execute(toolsApi.amlCheckerRetrieves(pollReq));
    if ("COMPLETED".equalsIgnoreCase(result.getStatus())
            || "FAILED".equalsIgnoreCase(result.getStatus())) {
        break;
    }
    Thread.sleep(2000);
}

// 3. Block high-risk addresses
if (result != null && "HIGH".equalsIgnoreCase(result.getRiskLevel())) {
    throw new RuntimeException("AML check failed: destination address is high-risk");
}

// 4. Proceed with transaction
CreateTransactionRequest txReq = new CreateTransactionRequest();
txReq.setDestinationAddress(destinationAddress);
// ... set other fields
```

---

## Supported Networks

| Network Value | Supported Chains |
|---------------|-----------------|
| `Bitcoin` | Bitcoin (BTC) |
| `Ethereum` | Ethereum, ERC-20 tokens, and EVM-compatible chains |
| `Tron` | TRON (TRX), TRC-20 tokens |

---

## Notes

- AML checker uses external providers — results may take several seconds to minutes.
- For transaction-level KYT reports (after a transaction completes), use `ComplianceApiService.kytReport()` — see [COMPLIANCE_API.md](COMPLIANCE_API.md).
- The `failOnAml=true` flag in `CreateTransactionRequest` (default) enables automatic server-side AML blocking — the Tools API is for manual pre-screening before sending.
- API endpoint: `POST /v1/tools/aml-checker/request` and `POST /v1/tools/aml-checker/retrieves`.

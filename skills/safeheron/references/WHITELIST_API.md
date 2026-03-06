# Whitelist API Reference

## Imports
```java
import com.safeheron.client.api.WhitelistApiService;
import com.safeheron.client.config.SafeheronConfig;
import com.safeheron.client.request.*;
import com.safeheron.client.response.*;
import com.safeheron.client.utils.ServiceCreator;
import com.safeheron.client.utils.ServiceExecutor;
```

## Create API Service
```java
WhitelistApiService whitelistApi = ServiceCreator.create(WhitelistApiService.class, safeheronConfig);
```

---

## Notes

Whitelist request/response class names follow the same SDK conventions:
- Requests: `com.safeheron.client.request.*`
- Responses: `com.safeheron.client.response.*`
- All calls: `ServiceExecutor.execute(whitelistApi.someMethod(request))`

For authoritative class names, see the official source:
https://github.com/Safeheron/safeheron-api-sdk-java/tree/main/src/main/java/com/safeheron/client

---

## Whitelist Operations Summary
| Operation | API Endpoint | Notes |
|-----------|-------------|-------|
| Create whitelist | `/v1/whitelist/create` | Returns whitelistKey |
| Modify whitelist | `/v1/whitelist/update` | Pass whitelistKey |
| Retrieve one | `/v1/whitelist/one` | By whitelistKey |
| List all | `/v1/whitelist/list` | Supports pagination |
| Delete | `/v1/whitelist/delete` | By whitelistKey |

---

## Key Fields
| Field | Description |
|-------|-------------|
| `whitelistKey` | Unique identifier (returned on create) |
| `whitelistName` | Display name |
| `coinKey` | Associated coin (e.g. `ETHEREUM_ETH`) |
| `address` | The destination address |
| `addressName` | Human-readable label |

---

## Best Practices
- Verify address validity with the Coin API before adding to whitelist.
- Use descriptive `whitelistName` and `addressName` for audit trails.
- In regulated environments, configure Safeheron Console policy to require human approval before whitelist additions take effect.

# Java SDK Setup & Configuration

## Maven
```xml
<dependency>
    <groupId>com.safeheron</groupId>
    <artifactId>api-sdk-java</artifactId>
    <version>1.0.9</version>
</dependency>
```

## Gradle
```groovy
implementation 'com.safeheron:api-sdk-java:1.0.9'
```

## Clone and Build Locally
```bash
git clone https://github.com/Safeheron/safeheron-api-sdk-java.git
cd safeheron-api-sdk-java
mvn install -Dmaven.test.skip=true
```

---

## Configuration File (config.yaml)
```yaml
# Required
apiKey: 080d****e06e60
privateKey: MIIJRQIBA*******DtGRBdennqu8g95jcrMxCUhsifVgzP6vUyg==
safeheronPublicKey: MIICI****QuTOTECAwEAAQ==
baseUrl: https://api.safeheron.vip

# Optional
requestTimeout: 20000   # milliseconds, default 20000
```

## Loading Config in Java
```java
// From YAML file
SafeheronConfig config = SafeheronConfig.loadFromFile("path/to/config.yaml");

// Programmatically
SafeheronConfig config = new SafeheronConfig();
config.setApiKey(System.getenv("SAFEHERON_API_KEY"));
config.setPrivateKey(System.getenv("SAFEHERON_PRIVATE_KEY"));
config.setSafeheronPublicKey(System.getenv("SAFEHERON_PUBLIC_KEY"));
config.setBaseUrl("https://api.safeheron.vip");
config.setRequestTimeout(20000);
```

## Initialize Service Clients
```java
SafeheronClient client = new SafeheronClient(config);

// Available services
AccountService accountService = new AccountService(client);
TransactionService txService   = new TransactionService(client);
WhitelistService wlService     = new WhitelistService(client);
Web3Service web3Service        = new Web3Service(client);
MPCSignService mpcService      = new MPCSignService(client);
CoinService coinService        = new CoinService(client);
GasService gasService          = new GasService(client);
```

---

## Spring Boot Integration Example
```java
@Configuration
public class SafeheronConfig {

    @Bean
    public SafeheronClient safeheronClient(
        @Value("${safeheron.apiKey}") String apiKey,
        @Value("${safeheron.privateKey}") String privateKey,
        @Value("${safeheron.safeheronPublicKey}") String safeheronPublicKey
    ) {
        com.safeheron.config.SafeheronConfig config = new com.safeheron.config.SafeheronConfig();
        config.setApiKey(apiKey);
        config.setPrivateKey(privateKey);
        config.setSafeheronPublicKey(safeheronPublicKey);
        config.setBaseUrl("https://api.safeheron.vip");
        return new SafeheronClient(config);
    }
}
```

```yaml
# application.yml
safeheron:
  apiKey: ${SAFEHERON_API_KEY}
  privateKey: ${SAFEHERON_PRIVATE_KEY}
  safeheronPublicKey: ${SAFEHERON_PLATFORM_PUBLIC_KEY}
```

---

## Error Handling
```java
try {
    CreateAccountResponse resp = accountService.createAccount(req);
} catch (SafeheronException e) {
    // e.getCode() — Safeheron error code
    // e.getMessage() — error description
    log.error("Safeheron API error {}: {}", e.getCode(), e.getMessage());
} catch (Exception e) {
    log.error("Unexpected error calling Safeheron", e);
}
```

## Common Error Codes
| Code | Meaning |
|------|---------|
| 200  | Success |
| 401  | Authentication failed |
| 403  | IP not whitelisted |
| 429  | Rate limit exceeded |
| 500  | Safeheron internal error |

See full error code list: https://docs.safeheron.com/api/en.html#Error%20Code

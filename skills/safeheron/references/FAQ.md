# Safeheron API — Frequently Asked Questions (FAQ)

This file covers real-world questions from developers and operators integrating the Safeheron API and SDK.

---

## API Co-Signer

**Q: What is the difference between the new and old Co-Signer deployment, specifically the "callback URL" in Web Console?**

A (EN): The new version of API Co-Signer only requires you to generate **one RSA key pair**. The old version required **two RSA key pairs**. The "callback URL" and its associated public key in the new Web Console correspond to what was previously configured as `biz_callback` and `biz_pubkey` in the old config file. Generate the public key with:
```bash
openssl rsa -in api_private.pem -out api_public.pem -pubout
```

A (中文): 新版本部署 Co-Signer 只需生成一对 RSA 公私钥；老版本需要生成两对 RSA 公私钥对。Web Console 里填写的 callback URL 及公钥，对应的就是老版本配置文件中的 `biz_callback` 和 `biz_pubkey`。

---

**Q: Configuring the Approval Callback Service — which keys are needed? How to export the Co-Signer public key?**

A (EN): You need two keys:
1. **Co-Signer identity public key** — export with: `sudo ./cosigner export-public-key`
   Your callback service uses this to verify that callback requests truly come from Co-Signer.
2. **Callback private key** — the private key corresponding to the callback public key you uploaded in Console when registering the API Key.
   Your callback service uses this to decrypt the AES key in the callback payload.

A (中文): 配置 Callback Service 需要：① Co-Signer 身份认证公钥（`sudo ./cosigner export-public-key` 导出）用于验签；② 申请 API Key 时上传的 callback 公钥所对应的私钥，用于解密 AES key。

---

**Q: Why is a .env config file still required if Secrets Manager is already used?**

A (EN): Secrets Manager is recommended for managing sensitive data (private keys, etc.), but some configuration values cannot be managed through Secrets Manager and must still be set in the `.env` file.

A (中文): Secrets Manager 主要管理敏感数据，但仍有部分配置无法用 Secrets Manager 管理，所以两者同时需要。

---

**Q: After modifying the .env config file, how do I make the changes take effect?**

A (EN): Stop and restart Co-Signer:
```bash
sudo ./cosigner stop
sudo ./cosigner start
```

A (中文): 先 stop 再 start，配置即可生效。

---

**Q: What format is required when uploading the callback public key to Web Console?**

A (EN): Use PEM format — the key file generated via OpenSSL command, starting with `-----BEGIN PUBLIC KEY-----`. When uploading to "API Key" configuration, the key should be on a single line (no line breaks). The Co-Signer identity public key exported via `export-public-key` is already in single-line format.

A (中文): 需要使用 PEM 格式（以 `-----BEGIN PUBLIC KEY-----` 开头的源文件）。在"访问的 API"配置项中，公钥需换成一行；Co-Signer 身份认证公钥默认导出即为一行。

---

**Q: Can the callback service and Co-Signer be deployed on the same internal network, using an internal IP for the callback URL?**

A (EN): Yes, as long as the machine running the callback service is reachable from Co-Signer at that internal IP address.

A (中文): 可以，只要 Co-Signer 能通过该 IP 正常访问 callback 服务的机器即可。

---

**Q: When is the callback URL required vs. optional?**

A (EN):
- **Production teams**: Callback URL is **required**.
- **Test/sandbox teams**: Callback URL is **optional**.

A (中文): 正式团队 Callback URL 必填；测试环境团队 Callback URL 选填。

---

**Q: Co-Signer failed to start, or approval tasks are stuck in "Waiting for API Co-Signer". How to diagnose?**

A (EN): Check the logs:
```bash
./cosigner logs -f        # stream live logs
./cosigner logs -s        # export logs to file, send to Safeheron Support
```
The logs will contain specific error messages.

A (中文): 使用 `./cosigner logs -f` 实时查看日志；`./cosigner logs -s` 导出日志发给 Safeheron Support。

---

**Q: Co-Signer log shows: "The final private key fragment is not exists, please activate API Co-Signer first." — is this an error?**

A (EN): This is **normal** — it means Co-Signer has started successfully. You need to proceed with the Co-Signer activation workflow in the App.

A (中文): 这是正常日志，代表 Co-Signer 已正常启动，继续在 App 中完成激活流程即可。

---

**Q: What is the polling interval for Co-Signer to check for pending approval tasks?**

A (EN): v2.0+ (new version): **every 5 seconds**. v1.x (old version): **every 1 second**.

A (中文): v2.0 以上新版本 5s；v1.x 老版本 1s。

---

**Q: Which KMS providers are supported for Co-Signer deployment?**

A (EN): AWS KMS and GCP KMS are supported. **Alibaba Cloud (Aliyun) KMS is NOT supported** — because Aliyun KMS requires the server to be on the same VPC, which is incompatible.

A (中文): 支持 AWS KMS 和 GCP KMS。不支持阿里云 KMS（原因：阿里云 KMS 需要和服务器使用同一个 VPC）。

---

**Q: Co-Signer logs show "Illegal IP" after startup. Why?**

A (EN): The Co-Signer host machine's IP address has not been added to the API Key's IP whitelist in Safeheron Console. Add the IP, then restart Co-Signer.

A (中文): Co-Signer 宿主机 IP 未加入对应 API Key 的白名单，加入后重启 Co-Signer 即可。

---

**Q: When using AWS with MySQL configured, Co-Signer was started with the `enable-mysql` parameter — which config takes precedence?**

A (EN): When started with `--enable-mysql`, Co-Signer will use the MySQL configuration from AWS Secrets Manager (not the local `.env` file).

A (中文): 启用 `enable-mysql` 参数时，Co-Signer 优先使用 AWS Secrets Manager 上的 MySQL 配置。

---

**Q: CLI `setup` or `start` command fails with: "Failed to log in to the Docker image repository. The token may have expired."**

A (EN): Known causes:
1. **Built-in token expired** (less common) — re-download the CLI from Safeheron Web Console.
2. **`start` run before `setup`** — run `docker images` to confirm Docker is ready, then run `setup` first.
3. **Firewall blocking `registry.gitlab.com`** — whitelist this domain.
4. **OS image issue** — Known to occur on Debian 4.19.316 with error: `Cannot autolaunch D-Bus without X11 $DISPLAY`. Solution: switch to **Ubuntu 24.04**.

A (中文): 已知原因：① CLI token 过期，重新从 Console 下载 CLI；② 未执行 setup 就先执行了 start；③ 防火墙屏蔽了 `registry.gitlab.com`；④ Linux 系统为 Debian 4.19.316（改用 Ubuntu 24.04）。

---

**Q: After updating the Callback URL in Web Console, does Co-Signer need to be restarted?**

A (EN): **No restart needed.** The updated Callback URL takes effect within **5 minutes** automatically.

A (中文): 无需重启，等待 5 分钟即可生效。

---

**Q: What is the callback timeout? I see error: "Timestamp out of range".**

A (EN): Your callback service's response timestamp must be **within 5 seconds** of the Co-Signer server's current time. Ensure both servers' clocks are synchronized (NTP). The error log looks like: `Failed to execute Approval Callback, error: Timestamp out of range`.

A (中文): 要求响应时间戳与 Co-Signer 服务器时间差在 5s 以内。请确保服务器 NTP 时钟同步。

---

**Q: After cancelling a transaction via Web/API, why does Co-Signer keep calling the Callback Service?**

A (EN): In Co-Signer versions **before 2.x**, you must also manually update the Co-Signer database to cancel the stored audit task:
```sql
UPDATE `mpc_client_trans_audit_record`
SET audit_status = 2
WHERE trade_no = '${tx_key}';
```
In v2.x and later, this is handled automatically.

A (中文): 2.x 以前版本需要手动在 Co-Signer 数据库执行上述 SQL 取消审核任务；2.x 及以上版本已自动处理。

---

## API Integration

**Q: How do I add coins to a newly created wallet? What are the caveats?**

A (EN): Use the V2 API (recommended) — supports adding up to **20 coins at once** via `coinKeyList`:
- Adding a **token** coinKey automatically adds its **mainnet** coin too.
- Adding a **mainnet** coinKey only adds that mainnet coin.
- Re-adding an already-added coinKey returns the same response (idempotent).
- If a coinKey was manually disabled in the UI after being added, the API **cannot** re-enable it.

```java
CreateAccountCoinV2Request req = new CreateAccountCoinV2Request();
req.setAccountKey(accountKey);
req.setCoinKeyList(Arrays.asList("USDT(ERC20)_ETHEREUM_USDT", "ETHEREUM_ETH"));
CreateAccountCoinV2Response res = ServiceExecutor.execute(accountApi.createAccountCoinV2(req));
```

A (中文): 推荐使用 V2 接口，`coinKeyList` 最多一次添加 20 个 coinKey。token coinKey 会自动添加对应主网币种；主网 coinKey 只添加该主网。已添加过的 coinKey 响应数据与上次一致（幂等）。手动关闭过的 coinKey，API 不会重新打开。

---

**Q: How to distinguish deposit (inflow) vs. withdrawal (outflow) vs. internal transfer by transaction type?**

A (EN): Check `destinationAccountType` and `sourceAccountType`:

| Scenario | destinationAccountType | sourceAccountType |
|----------|----------------------|------------------|
| Withdrawal (outflow) | `ONE_TIME_ADDRESS` | `VAULT_ACCOUNT` |
| Internal transfer | `VAULT_ACCOUNT` | `VAULT_ACCOUNT` |
| Deposit (inflow) | — | `UNKNOWN` |

A (中文): 目标类型 `VAULT_ACCOUNT` = 内部转账；`ONE_TIME_ADDRESS` = 流出（提币）；源账户类型 `UNKNOWN` = 流入（充值）。

---

**Q: First time using MPC Sign, getting error 9028. Why?**

A (EN): MPC Sign is a low-level cryptographic operation with high security requirements. A **special MPC Sign policy** must be enabled on your team first. Contact Safeheron Support via email using the official template:
- CN: https://support.safeheron.com/help-center/jian-ti-zhong-wen/chan-pin/shen-ru-safeheron/mpc-sign-ce-le
- EN: https://support.safeheron.com/help-center/product-and-solution/dive-into-safeheron/mpc-sign-policy

A (中文): MPC Sign 需要申请专项策略，以邮件模版形式联系 Support 开通。

---

**Q: What are Safeheron's outbound (egress) IP addresses? (For configuring firewall whitelists to receive webhooks/callbacks)**

A (EN): Safeheron's outbound IPs for webhooks and callbacks:
- `18.162.105.64`
- `18.167.22.59`
- `18.167.21.182`

A (中文): Safeheron 出口 IP 列表：18.162.105.64 / 18.167.22.59 / 18.167.21.182

---

**Q: Failed to configure a Webhook URL — it keeps showing an error.**

A (EN): Ensure the Webhook URL is publicly accessible from the internet. For ngrok URLs, append `/webhook` to make it routable. Note: some public tunnel domains (ngrok, webhook.site, etc.) may be blocked by Akamai CDN.

A (中文): 检查 Webhook URL 是否在公网透出，ngrok 需加 `/webhook` 路径。部分内网穿透工具域名（ngrok、webhook.site 等）可能被 Akamai 拦截。

---

**Q: What timezone does Safeheron's server use?**

A (EN): **UTC+0 (UTC)**. All API timestamps are Unix milliseconds (timezone-agnostic).

A (中文): 服务端为 UTC 0 时区。

---

**Q: A transaction is stuck in PENDING. How to differentiate an original transaction from its speed-up transaction?**

A (EN): The original and speed-up are **independent transactions**. Distinguish them via:
- `replaceTxKey` — on the speed-up tx, references the original
- `replacedCustomerRefId` — on the speed-up tx, references the original's customerRefId
- `speedUpHistory` — on the original tx (from `/v1/transactions/one`), lists its speed-up transactions

The speed-up transaction will cause the original to fail.

A (中文): 原交易和加速交易是两笔独立交易。通过加速交易的 `replaceTxKey` / `replacedCustomerRefId` 与源交易关联；也可在原交易的 `speedUpHistory` 字段查看加速记录。

---

**Q: What protocol is required for Webhook URL and Callback URL?**

A (EN): HTTP or HTTPS. URL must start with `http://` or `https://`. Some specific domains may be blocked by Akamai (ngrok, webhook.site, other public tunnel tools).

A (中文): 支持 HTTP 或 HTTPS 协议，URL 须以 `http://` 或 `https://` 开头。

---

**Q: Web3 API returns "account not found". Why?**

A (EN): All Web3 API calls require a **Web3 wallet** (`accountType = WEB3_ACCOUNT`). Using a regular vault account key (`accountType = VAULT_ACCOUNT`) causes this error.

A (中文): 请求 Safeheron Web3 所有 API 接口，必须使用 Web3 钱包的 accountKey，不能用普通资产钱包 accountKey。

---

**Q: After Auto Sweep (归集) completes, does Safeheron send a webhook notification?**

A (EN): Yes. Sweep transactions, gas refill transactions, speed-up transactions, batch transfers — all are treated as regular transactions internally and all generate Webhook notifications.

A (中文): 无论是归集/加油/普通/加速/批量转账交易，系统内都是一笔交易，都会通过 Webhook 回调通知。

---

**Q: How to query the balance of USDT/ETH on a specific MPC wallet address?**

A (EN): Call `/v1/account/coin/list` and read the `balance` field in the response:
```java
ListAccountCoinRequest req = new ListAccountCoinRequest();
req.setAccountKey(accountKey);
List<AccountCoinResponse> coins = ServiceExecutor.execute(accountApi.listAccountCoin(req));
// coin.getBalance() — available balance
```

A (中文): 调用 `/v1/account/coin/list` 接口，查看响应中的 `balance` 字段。

---

**Q: How to query the aggregate balance of a specific coinKey across all wallets in the team?**

A (EN): Call `/v1/coin/balance/snapshot` and read `coinBalance`:
```java
CoinBalanceSnapshotRequest req = new CoinBalanceSnapshotRequest();
req.setGmt8Date("2026-01-01");
List<CoinBalanceSnapshotResponse> result = ServiceExecutor.execute(coinApi.coinBalanceSnapshot(req));
// result.get(0).getCoinBalance() — total balance across all accounts
```

A (中文): 调用 `/v1/coin/balance/snapshot` 接口，查看响应中的 `coinBalance` 字段。

---

**Q: Is the new SDK backward compatible with the old SDK?**

A (EN): Yes. SDK updates only **add** new methods — all existing methods are preserved. You can upgrade without breaking existing integrations.

A (中文): 兼容，SDK 更新只新增方法，不删除旧方法，升级不会破坏已有集成。

---

**Q: After enabling custom Webhook block confirmation count, does it only apply to incoming transactions?**

A (EN): No. Once a custom confirmation count is enabled for a chain, it applies to **both incoming and outgoing** transactions on that chain.

A (中文): 一旦开启某公链的自定义确认数，该链的入账和出账交易都会生效。

---

**Q: Calling an API with a duplicate customerRefId — what happens?**

A (EN): Safeheron returns **error code 9001** ("Merchant unique business ID already exists"). This is also the idempotency mechanism: if you retry a timed-out call with the **same** customerRefId, Safeheron returns the original transaction rather than creating a duplicate.

A (中文): 返回错误码 9001。这也是幂等机制：超时重试时使用相同 customerRefId，会返回原始交易，不会重复创建。

---

**Q: What is the priority between `txFeeLevel` and `feeRateDto`? What if only `gasLimit` is set in `feeRateDto`?**

A (EN): `txFeeLevel` takes **priority** over `feeRateDto` if both are provided. If only `feeRateDto` is used, you must provide at least **both `gasLimit` AND `feeRate`** — providing only `gasLimit` will cause the transaction creation to fail.

A (中文): 若两个都传，`txFeeLevel` 优先。只用 `feeRateDto` 时，至少要同时传 `gasLimit` 和 `feeRate`，只传 `gasLimit` 交易创建会失败。

---

**Q: What IP address formats are supported for API Key IP whitelist?**

A (EN): Both **IPv4** and **IPv6** are supported.

A (中文): 支持 IPv4 和 IPv6 格式。

---

**Q: Does Safeheron filter webhook notifications by transaction amount? Will small-amount or dust-attack deposits trigger webhooks?**

A (EN): **No filtering.** Any successful incoming transaction — regardless of amount, including dust attacks — triggers a Webhook notification.

A (中文): 没有过滤，只要有交易成功入账，不论金额大小，都会触发 Webhook 通知。

---

**Q: If `hiddenOnUI=false` is set when creating a wallet via API, is the wallet counted as visible or hidden in APP stats?**

A (EN): A wallet with `hiddenOnUI=false` is a **visible account**. Its assets are included in the visible account totals in APP/Web Console.

A (中文): `hiddenOnUI=false` 的钱包为可见账户，资产统计在可见账户里。

---

**Q: Do adding/modifying/deleting Webhook URL or RSA public key require admin approval?**

A (EN): **No approval required.** All changes (add/modify/delete) to Webhook URL and RSA public key take effect **immediately**.

A (中文): 无需管理员审批，新增/修改/删除立即生效。

---

**Q: How is transaction fee estimated for different blockchains?**

| Chain | Fee Formula | Special Notes |
|-------|-------------|--------------|
| Bitcoin (BTC) | `feeRate × byteSize` | No UTXOs = no balance → fee = 0 |
| Ethereum / EVM | `feeRate × gasLimit` | — |
| TRON | Pre-execution on-chain | `sourceAddress` required to check staking/energy; no LOW/MID/HIGH tiers |
| Solana | Fixed fee | 0.000005 SOL base fee |

A (中文): BTC 手续费 = feeRate × bytesSize；ETH = feeRate × gasLimit；TRON 为预执行链上返回数据（`sourceAddress` 必传以检查质押/能量），无低中高档位区别。

---

**Q: What happens when the webhook URL is modified? When does webhook delivery stop?**

A (EN): After URL modification, events are delivered to the **new URL**. Webhook delivery only **stops** when the webhook URL is **deleted** (not merely modified).

A (中文): 修改后 Webhook 推送到新 URL；只有删除 Webhook 配置后才会停止推送。

---

**Q: Speed-up (accelerate) transaction — how does it relate to the original transaction?**

A (EN): The original and speed-up are **independent transactions**. The speed-up will invalidate the original (original → FAILED). The speed-up is also reported via Webhook. Query speed-up history via `speedUpHistory` field in `/v1/transactions/one`.

A (中文): 原交易和加速交易为两笔独立交易，加速交易会顶掉原交易（原交易变 FAILED）。加速交易信息通过 `/v1/transactions/one` 中的 `speedUpHistory` 查询。

---

**Q: Can the DEPOSIT label on a wallet be removed? Can it be re-applied?**

A (EN): Yes. Pass `accountTag = "NONE"` to remove the DEPOSIT label. It can be re-applied later by passing `accountTag = "DEPOSIT"` again.
```java
atchUpdateAccountTagRequest req = new BatchUpdateAccountTagRequest();
req.setAccountKeyList(Arrays.asList("your-account-key"));
req.setAccountTag("NONE");   // remove
// req.setAccountTag("DEPOSIT"); // re-apply
ServiceExecutor.execute(accountApi.batchUpdateAccountTag(req));
```

A (中文): 通过传 `NONE` 可取消 DEPOSIT 标签，取消后可再次传 `DEPOSIT` 重新标记。

---

**Q: When `destinationAccountType` is `VAULT_ACCOUNT`, is `destinationAccountKey` required?**

A (EN): Rules:
- `destinationAccountType = VAULT_ACCOUNT` → `destinationAccountKey` must be the **accountKey**
- `destinationAccountType = WHITELISTING_ACCOUNT` → `destinationAccountKey` must be the **whitelistKey**
- `destinationAccountType = ONE_TIME_ADDRESS` → `destinationAccountKey` is **NOT used** (set `destinationAddress` instead)

A (中文): `VAULT_ACCOUNT` 时传 accountKey；`WHITELISTING_ACCOUNT` 时传 whitelistKey；`ONE_TIME_ADDRESS` 时不传 destinationAccountKey，改传 destinationAddress。

---

**Q: Webhook retry schedule — what happens when delivery fails?**

A (EN): Retry schedule after first failure:
`30s → 1min → 5min → 1h → 12h → 24h` — total **7 attempts**. After the 7th failure, no further retries.

Call `POST /v1/webhook/resend/failed` to manually retry all failed events.

A (中文): 失败后重试：30s、1m、5m、1h、12h、24h，共 7 次，最后一次失败则不再重试。可调用 `/v1/webhook/resend/failed` 手动重推所有失败事件。

---

## Transactions

**Q: SOL (Solana) transfer fails on-chain. Why?**

A (EN): Solana accounts have a **minimum rent-exempt balance** of approximately **0.002 SOL**. You cannot transfer the full balance — the maximum transfer amount is `(balance - 0.002 SOL)`.

A (中文): Solana 账户有约 0.002 SOL 的免租金额不能转走，最大转账金额需减去该部分。

---

**Q: Which test networks does Safeheron support in production? Where to get test tokens?**

A (EN):

| Network | Token | Details |
|---------|-------|---------|
| Ethereum Sepolia | GSK | Contract: `0xF191b0720cb49DdAb6ECd72a65955a35b31fc944` |
| Ethereum Sepolia | USDC | Contract: `0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238`, Faucet: https://faucet.circle.com/ |
| TRON Shasta | TLK | Contract: `TC2jGCcJUh33oAeQHYHBXu7KVVxrjitkHw` |
| Bitcoin Testnet | BTC | Faucet: https://coinfaucet.eu/en/btc-testnet4/ |

A (中文): 支持 Sepolia 测试网、BTC 测试网、TRON 测试网。见上表中各 contract 地址及水龙头链接。

---

**Q: Which chains support speed-up (transaction acceleration)?**

A (EN):
- **UTXO chains with speed-up support**: BTC, BCH, DASH, LTC, DOGE
- **Non-UTXO chains with speed-up support**: All EVM chains, FIL, Aptos, CFX
- **Chains WITHOUT speed-up**: NEAR, SUI, TRON, SOL, TON

A (中文): 支持加速的 UTXO：BTC、BCH、DASH、LTC、DOGE；支持加速的非 UTXO：EVM、FIL、Aptos、CFX；不支持加速：NEAR、SUI、TRON、SOL、TON。

---

**Q: What happens to an address flagged by AML? How to handle it?**

A (EN): The recommended approach is to **change the address** and return the contaminated assets to the original sender to maintain compliance. The transaction in Safeheron will have `amlLock = "YES"`.

A (中文): 建议更换地址，并将收到的有问题资产退回给用户，做到流程合规。

---

**Q: Under what circumstances are coins added automatically?**

A (EN):
1. If a mainnet coin is enabled and a **token** of that chain is received as deposit → the token coin type is automatically added.
2. If you add a **token** coinKey via API → the corresponding mainnet coin is also automatically added.

A (中文): 主网币种打开后，入账 token 会自动添加对应 token 币种；API 添加 token 会自动将该 token 对应的主网币种自动添加。

---

**Q: Nonce rules for EVM chains**

A (EN): Nonce is determined at **approval time** (when transaction enters WAIT_SIGN):
1. If `maxUseNonce = null` → only `chainNonce` is used.
2. `nonce < chainNonce` → error: "Nonce too low".
3. `nonce > maxUseNonce + 1` → error: "Nonce cannot exceed maxUseNonce+1".
4. `chainNonce ≤ nonce ≤ maxUseNonce` → confirmation prompt shown to user.

When multiple transactions are pending simultaneously, the one **approved first** gets the recommended nonce first.

A (中文): nonce 在审批时确认，先审批的先占用推荐 nonce。规则见上条目。

---

**Q: TRON fee estimation shows negative values. Why?**

A (EN): This occurs when the sender address has staking AND the conditions are: `bandwidth_available < 360` AND `energy_available > 360`. The fee calculation can yield a negative value in this specific edge case.

A (中文): 质押 from 地址的 bandwidth available < 360 且 energy available > 360 时，手续费计算会出现负数。

---

**Q: Does Aptos support token transfers?**

A (EN): **Not currently supported.** Aptos token transfers require the recipient address to first call `register` for that token type. This requires significant system changes on Safeheron's side, so it is not yet supported.

A (中文): 暂不支持。Aptos token 转账需要 to 地址先发起 register，系统改造较大暂未支持。

---

**Q: TRON fee estimation — is there a difference between LOW/MEDIUM/HIGH levels?**

A (EN): **No difference.** TRON fee is determined by **pre-execution on-chain** — whatever the chain returns is the fee. The LOW/MEDIUM/HIGH labels all yield the same value for TRON.

A (中文): TRON 手续费是预执行链上返回的值，无低中高区别，三档显示一样。

---

**Q: A wallet only has ETH and USDT added, but USDC was deposited — it doesn't show. Why?**

A (EN): The USDC coin type was not added to the wallet. The balance will update once USDC is added via `createAccountCoinV2`. The deposit record may need manual reconciliation.

A (中文): 钱包未添加 USDC 币种，需通过接口添加 USDC 后，余额会自动刷新；入账记录需手动补偿。

---

**Q: How to transfer funds to a contract address via API?**

A (EN): Set `failOnContract = false` in the transaction request. The default is `true` (blocked). Note: transferring to contract addresses carries risk — only disable this flag when you explicitly know the destination is a valid contract.

```java
CreateTransactionRequest req = new CreateTransactionRequest();
req.setFailOnContract(false);  // required to send to contract addresses
```

A (中文): API 转账到合约地址需要传 `failOnContract = false`，默认为 `true`（拦截）。

---

**Q: UTXO sweep (归集) basic rules**

A (EN): Manual/API sweep: threshold is 30 UTXOs. When UTXO count ≤ 30, sweep is not recommended; when > 30, sweep is recommended. Auto Sweep applies its own configured conditions.

A (中文): 手动归集/API 归集：当 UTXO 碎片数 ≤ 30 不建议归集；> 30 时建议归集。Auto Sweep 按配置的条件触发。

---

## Permissions

**Q: What permissions are required for the following operations?**

| Operation | Required Permission |
|-----------|-------------------|
| Create wallet, modify wallet name | "管理团队钱包" (Manage Team Wallets) |
| Add Web3 custom network | "管理团队钱包" (production team only) |
| Initiate transaction, cancel transaction | "发起交易" (Initiate Transaction) |
| Manual webhook trigger in Web Console | "发起交易" (Initiate Transaction) |
| Modify API Key, Webhook, confirmation count | "API 管理" (API Management) |
| Download Co-Signer CLI and deployment docs | "API 管理" (API Management) |
| Manage policy engine (add/edit/delete/sort strategies) | "管理交易策略" (Manage Transaction Policies) |
| Auto Sweep policy management | "管理交易策略" (Manage Transaction Policies) |
| Whitelist / Connect app management | "管理白名单/Connect 应用" |
| Enable/edit custom approval flow | Owner and Admin only |
| View audit logs | Admin and Owner only |
| AML Checker | No special permission required |
| Enable gas refill switch | "钱包管理" (Wallet Management) |
| Prepay energy rental, configure alert threshold | "发起交易" (Initiate Transaction) |
| Full scan (全量扫描) button | "管理交易策略" (Manage Transaction Policies) |
| Enter Co-Signer application | No permission required |
| Add approval node participants | "成员管理" (Member Management) |

---

## Policy & Strategy

**Q: What is the maximum number of whitelists supported?**

A (EN): Default maximum is **800 entries**. This can be increased internally by Safeheron Support (requires SQL change).

A (中文): 默认最多 800 条白名单，可联系 Safeheron Support 申请扩容。

---

**Q: "Strategy range cannot overlap" error when configuring policies**

A (EN): Policy amount ranges must be contiguous (no gaps, no overlaps). Example of correct configuration:
- Range 1: [0, 1,000) → Approval flow A
- Range 2: [1,000, 10,000) → Approval flow B
- Range 3: [10,000, ∞) → Approval flow C

Incorrect: Having `[0, 100,000,000)` and `[10,000,000, ∞)` — they overlap at 10,000,000.

A (中文): 策略区间不能重叠也不能跳开。示例：[0,1000)、[1000,10000)、[10000,∞)。

---

**Q: Transaction records, audit logs — how far back can you query?**

A (EN):
- Transaction records, energy rental records, prepay records: default **3 months**, max span **1 year**
- Audit logs: max **3 months**

A (中文): 交易记录等默认 3 个月，最大跨度 1 年；审计日志最大 3 个月。

---

**Q: Maximum number of policies supported per team?**

A (EN): Default teams support **max 10 policies**. Special requests can be accommodated up to **30 policies** per team by contacting Safeheron Support.

A (中文): 默认支持最多 10 条策略；特殊需求可联系 Support 增至最多 30 条。

---

## Web Console & App

**Q: How to find accountKey and coinKey in Web Console?**

A (EN):
- **accountKey**: Web Console → Wallets → search wallet name → Copy accountKey
- **coinKey**: Web Console → any Wallet → hover mouse over the coin → coinKey appears in tooltip

A (中文): accountKey 在钱包页搜索钱包名称后 Copy；coinKey 在钱包详情页鼠标悬浮在对应币种即可看到。

---

**Q: Auto Sweep is not executing automatically. How to troubleshoot?**

A (EN): Check these four conditions:
1. Target wallets have the **DEPOSIT** label set.
2. The Auto Sweep policy is **active**, and incoming transactions meet the configured threshold conditions.
3. If wallets already had token balances **before** the policy was configured, re-deposit or trigger a full scan.
4. Check if network **gas is too low** to cover the sweep transaction.

A (中文): 检查：① 待归集钱包是否打了 DEPOSIT 标签；② 策略已生效且入账满足条件；③ 策略配置前已有余额的需重新入账或全量触发；④ 网络 gas 是否偏低。

---


**Q: Can wallets be deleted?**

A (EN): **No**. Wallets cannot be deleted. You can rename, hide (`hiddenOnUI=true`), or archive a wallet.

A (中文): 不支持删除钱包，可修改名称、隐藏（`hiddenOnUI=true`）或归档。

---

**Q: APP shows max how many wallets? How are they sorted?**

A (EN): Max **1,000 wallets** displayed. Sorted by USD value descending. Wallets with no balance appear at the bottom. After 1,000 wallets, the first 1,000 by creation time are shown (not sorted by value then truncated).

A (中文): 最多显示 1000 个钱包，按 USD 价值从高到低排序，无余额钱包排最后。超过 1000 个取创建时间最早的 1000 个，不再排序后截取。

---

**Q: How is manual webhook trigger rate limited?**

A (EN): Manual webhook trigger rate limit: **10 times per minute**.

A (中文): 手动触发 Webhook 频率限制：10 次/分钟。

---

## Security

**Q: How should user permissions be assigned? (Principle of least privilege)**

A (EN): Permission assignment must follow these principles:
1. **Least privilege** — only enable the permissions actually required for the current business role. Do not grant all permissions by default.
2. **Separation of duties** — the person who initiates a transaction and the person who approves it must be different accounts. Never allow self-initiate + self-approve.
3. **Periodic review** — regularly audit permission assignments. Revoke permissions for users who have left the team or changed roles to avoid stale permission risk.

A (中文): 权限分配遵循三原则：① 最小权限原则，只开启当前业务实际需要的权限，不开启全部权限；② 职责分离，发起权限和审核权限分开，避免自发自审；③ 定期做权限审查，避免人员离职或岗位变更遗留权限风险。

---

**Q: What are the security requirements for Webhook integration?**

A (EN):
1. **HTTPS is mandatory** — never expose an HTTP Webhook endpoint in production. All Webhook URLs must use HTTPS.
2. **IP whitelist via VPC security group or firewall** — only allow inbound traffic from Safeheron's egress IPs: `18.162.105.64`, `18.167.22.59`, `18.167.21.182`. Block all other sources.
3. **Verify signature before processing** — validate the `sig` field in every Webhook payload using Safeheron's RSA public key before taking any action. Reject events with invalid signatures.
4. **Idempotent handler** — Safeheron retries up to 7 times; ensure duplicate events do not cause double-crediting or double-processing.
5. **Terminal state wins** — never downgrade a transaction status (e.g., `COMPLETED` → `CONFIRMING`). Out-of-order delivery is possible; if a terminal-state event was already applied, discard any later intermediate-state event for the same transaction.

A (中文): ① 必须使用 HTTPS，生产环境禁止暴露 HTTP 的 Webhook 接收接口；② 使用 VPC 安全组或防火墙只放行 Safeheron 出口 IP（18.162.105.64、18.167.22.59、18.167.21.182）；③ 每次接收 Webhook 必须先验签再处理，验签失败直接拒绝；④ 接口必须幂等，防止重复入账；⑤ 终态优先，不回滚交易状态，乱序场景以终态为准。

---

**Q: What are the security requirements for Co-Signer deployment?**

A (EN):
1. **Private network only** — the Co-Signer host must be deployed in an internal isolated network. It must not be directly exposed to the public internet.
2. **Approval Callback Service** — the callback service should only accept inbound traffic from the Co-Signer host IP, not from the public internet.
3. **Production API Key must configure Callback** — for any API Key used with Co-Signer in production, the Callback URL is **required**. Never skip Callback configuration in production.
4. **Never blindly approve** — the callback service must validate every transaction: `customerRefId` must match a real pending business order; amount must match; destination address must match. Reject any discrepancy.

A (中文): ① Co-Signer 必须部署在内网，不直接暴露公网；② Callback Service 仅接受来自 Co-Signer 宿主机 IP 的请求；③ 生产环境 Co-signer API Key 必须配置 Callback，严禁跳过；④ 禁止盲目通过，必须校验 customerRefId、金额、目标地址与业务系统一致。

---

**Q: What should I be careful about with API Key security?**

A (EN):
1. **Minimum permissions** — configure each API Key with only the permissions strictly required for its use case. Do not enable all permissions.
2. **No plaintext private keys in code or config** — RSA private keys must never be written into source code or configuration files. Non-essential personnel must not have access to these keys.
3. **Minimal IP whitelist** — only add the server IPs that genuinely need API access. Remove stale or decommissioned IPs promptly.
4. **Production Co-Signer API Key must have Callback configured** — this is mandatory in production, not optional.
5. **Rotate keys on suspected compromise** — if a key may have been exposed, rotate it immediately.

A (中文): ① 权限配置最小化，只开启业务实际需要的权限；② RSA 相关私钥不得明文写入代码或配置文件，非核心人员不要接触；③ IP 白名单仅保留业务实际需要的最少服务器 IP，及时清理废弃 IP；④ 生产环境 Co-signer API Key 必须配置 Callback，严禁跳过；⑤ 疑似泄露时立即轮换。

---

**Q: What are the security best practices for configuring transaction policies?**

A (EN):
1. **Tiered approval based on amount** — configure different approval flows for different amount ranges. Example: small amounts → Co-Signer auto-approve; large amounts → multi-person team approval.
2. **Catch-all blocking rule at the bottom** — always add a "block all" rule as the last entry in the policy stack to intercept any transactions that match no defined rule. This prevents unmatched transactions from slipping through.
3. **Test with small amounts after configuration** — after setting up policies, verify the approval flow works as expected by running a small-amount test transaction.
4. **Periodic security audit** — regularly review and audit policy rules to ensure they remain appropriate as business changes.
5. **Subscribe to `NO_MATCHING_TRANSACTION_POLICY` webhook** — use this event as an operational alert for transactions that hit no policy.

A (中文): ① 建议基于金额配置分级审批流程（如小额自动 Co-Signer 审批、大额团队多签）；② 在最底层配置一条"拦截所有"的兜底策略，拦截所有未匹配规则的异常交易；③ 策略配置完成后，通过小额测试验证审批流程是否符合预期；④ 定期对策略规则进行安全审计；⑤ 订阅 `NO_MATCHING_TRANSACTION_POLICY` Webhook 事件作为运营告警。

---

**Q: What should I be careful about when using the Safeheron App?**

A (EN):
1. **Use a dedicated device** — avoid installing the Safeheron App on a personal everyday phone. A compromised, infected, or accidentally misused personal device could result in asset loss.
2. **Enable 2FA** — two-factor authentication must be enabled for every account that accesses the Safeheron App.
3. **No VPN on the App device** — VPN apps carry traffic interception risk. Do not install or enable a VPN on the device running the Safeheron App.
4. **Offline backup of private key and mnemonic** — back up offline (handwritten on paper). Never screenshot, never upload to cloud storage, never transmit via any instant messaging tool.
5. **Keep App updated** — regularly check for and install the latest version of the Safeheron App to ensure security patches are applied.
6. **Avoid public Wi-Fi** — do not perform asset operations on public Wi-Fi (airports, cafés, etc.) due to man-in-the-middle attack risk.

A (中文): ① 避免在个人日常使用的手机上安装 Safeheron App，降低设备被攻击或误操作风险；② 必须开启双因素认证（2FA）；③ 禁止在 App 所在手机上安装或启用 VPN；④ 私钥与助记词必须离线安全备份（写纸留档），不得截屏、上传云盘或通过即时通讯工具传输；⑤ 保持 App 为最新版本；⑥ 不在公共 Wi-Fi 下进行资产操作，防范中间人攻击。

---

**Q: How should server permissions be assigned?**

A (EN): Server permissions must be strictly controlled:
1. Apply the **principle of least privilege** — only grant users the minimum permissions required to perform their role. Never grant broad administrative access by default.
2. **Regularly audit and revoke** — periodically review all permission assignments. Immediately revoke access for users who no longer need it (e.g., role change, departure).

A (中文): 服务器权限应严格分配：① 采用最小权限原则，只授予用户完成工作所需的最低权限；② 定期审查并更新权限分配，及时撤销不再需要访问权限的用户，降低安全风险。

---

**Q: What password strength is required?**

A (EN): All accounts and credentials involved in the deployment — including database passwords, cloud service account passwords, and API Key-related credentials — must use **strong randomly generated passwords**. Strong passwords must:
- Contain letters, digits, and special characters
- Avoid birthdates, names, or dictionary words
- Be rotated periodically

Additionally, **enable 2FA/MFA** on all services that support it to add a second layer of defense.

A (中文): 部署 API Co-Signer 期间涉及的所有账户和密码（包括数据库密码、云服务账户密码等）必须采用随机生成的强密码策略（包含字母、数字、特殊字符，避免生日/姓名/字典词，定期轮换）。强烈推荐为所有支持双因素或多因素认证（2FA/MFA）的服务启用该功能。

---

**Q: How should sensitive information be stored?**

A (EN): All sensitive information — passwords, private keys, API keys, mnemonics — must be stored using a secure management tool such as a password manager (e.g., **1Password**). Rules:
1. **Never share in plaintext** — do not transmit or share sensitive information via social media, messaging apps, email, or any unencrypted channel.
2. **Never store in plain files** — do not place sensitive values in easily accessible files, `.env` files committed to git, or unencrypted note-taking apps.
3. **Use a secrets manager in production** — use AWS Secrets Manager, GCP Secret Manager, or HashiCorp Vault for server-side credential management.
4. **Offline backup for mnemonics and key shards** — handwritten on paper, stored in a physically secure location (e.g., safe). Never photographed, never in cloud storage.

A (中文): 所有敏感信息（密码、私钥、API Key、助记词）应使用安全管理工具存储，如密码管理器（1Password）。① 严禁以任何形式在社交媒体或即时通讯工具上传输或分享明文敏感信息；② 严禁存放在普通易于访问的文件中；③ 生产环境服务端使用 Secrets Manager 管理凭证；④ 助记词和私钥分片必须离线备份（写纸），存放于物理安全场所，禁止拍照或上传云盘。

---

## Support Contact Templates

**Q: How to apply for MPC Sign policy?**

- CN: https://support.safeheron.com/help-center/jian-ti-zhong-wen/chan-pin/shen-ru-safeheron/mpc-sign-ce-le
- EN: https://support.safeheron.com/help-center/product-and-solution/dive-into-safeheron/mpc-sign-policy

**Q: How to apply for Web3 wallet policy?**

- CN: https://support.safeheron.com/help-center/jian-ti-zhong-wen/chan-pin/shen-ru-safeheron/web3-ce-le
- EN: https://support.safeheron.com/help-center/product-and-solution/dive-into-safeheron/web3-policy

**Q: How to request admin count change or decision mode change?**

Send email to support@safeheron.com, CC all other admins. Email template:
```
Subject: Team Admin Change
Body:
Team ID:
New number of admins: X
New decision mode: X/X
Remove admins (if needed):
```
Required approvals follow the current decision mode threshold (e.g. 3/5 requires 3 admin replies).

**Q: How to reset Google Authenticator (GA)?**

- CN: https://support.safeheron.com/help-center/jian-ti-zhong-wen/chan-pin/bang-zhu/wang-ji-gu-ge-yan-zheng
- EN: https://support.safeheron.com/help-center/product-and-solution/support/what-if-i-forget-google-authentication

# MPC Sign API Reference

See [WEB3_API.md](WEB3_API.md) — the MPC Sign section covers:

- Create MPC Sign Wallet Account
- Create an MPC Sign Transaction
- Retrieve a Single MPC Sign Transaction
- MPC Sign Transaction List
- Full ERC-20 Token Transfer Example (Web3j + MPC Sign)

## MPC Sign Status Flow
```
SUBMITTED → WAIT_AUDIT → WAIT_SIGN → SUCCESS
                                    └─ FAILED | REVOKED | REJECTED
```

## Signature Algorithms
| Value | Curve | Common Use |
|-------|-------|-----------|
| SECP256K1 | secp256k1 | Ethereum, Bitcoin, BSC |
| ED25519   | Ed25519   | Solana, Near, Aptos |

---
name: x402-merchant-onboard
description: x402 merchant onboarding on GOAT Testnet3 - register merchant, configure receiving addresses, generate API keys, create payment orders, and manage the full merchant lifecycle on GOAT Network x402 payment protocol (testnet environment)
argument-hint: "[action]"
---

# x402 Merchant Onboarding & Payment Flow (Testnet)

> **This skill operates on GOAT Testnet3.** All addresses, tokens, and API endpoints are testnet environments. Tokens have no real value. Do NOT use testnet credentials or configurations in production.

Guide the user through x402 merchant onboarding and payment operations on GOAT Network Testnet3.

## DIRECT vs DELEGATE Mode

x402 supports two payment modes. The merchant must choose one at registration time (`receive_type`), which determines the entire payment flow.

### DIRECT Mode

- **Flow**: Payer directly transfers ERC20 tokens to the merchant's receiving address
- **Verification**: Platform monitors on-chain ERC20 Transfer events and matches `from`, `to`, `amount`, `token`, `chain` against the order
- **Flow types**: `ERC20_DIRECT`, `SOL_DIRECT`
- **Callback support**: No
- **Security**: Basic — relies on event matching only. Since there is no cryptographic binding between the payment and the order, there are theoretical collision/replay risks (e.g., a coincidental transfer with matching parameters could be mistakenly matched to an order)
- **Use case**: Simple payment-gated access, low-value transactions, testing
- **Fee (Testnet3)**: $0.05 per order

### DELEGATE Mode

- **Flow**: Payer transfers tokens to a **TSS (Threshold Signature Scheme) address** and signs an **EIP-712 structured data** payload that cryptographically binds the payment to a specific order
- **Verification**: Platform first verifies the EIP-712 signature (which contains `orderId`, `amount`, `payer`, etc.), then the TSS committee settles on-chain. The signature cannot be reused for other orders
- **Flow types**: `ERC20_3009` (EIP-3009 transferWithAuthorization), `ERC20_APPROVE_XFER`, `SOL_APPROVE_XFER`
- **Callback support**: Yes — supports on-chain callback contracts implementing `IX402CallbackAdapter`, enabling NFT mints, staking, DeFi actions, etc.
- **Security**: Strong — payment is cryptographically bound to the order via EIP-712 signature, preventing forgery, replay, and collision attacks
- **Use case**: Production payments, advanced settlement workflows, on-chain callbacks
- **Fee (Testnet3)**: $0.15 per order

### Callback Contract Interface (DELEGATE only)

```solidity
interface IX402CallbackAdapter {
    function onX402Callback(
        address payer,
        address token,
        uint256 amount,
        bytes calldata calldata_
    ) external returns (bool success);
}
```

### Which mode to choose?

| Scenario | Recommended Mode |
|----------|-----------------|
| Testing / prototyping | DIRECT |
| Simple one-time payments | DIRECT |
| Production with real value | DELEGATE |
| Need on-chain callbacks (NFT, DeFi) | DELEGATE |
| High-security requirements | DELEGATE |

## Environment (Testnet)

- **Merchant Portal API**: `https://x402-api-lx58aabp0r.testnet3.goat.network`
  - Merchant management endpoints: `/merchant/v1/...`
  - Payment gateway endpoints: `/api/v1/...`
- **Merchant Portal Frontend**: `https://x402-merchant-lx58aabp0r.testnet3.goat.network`
- **RPC**: `https://rpc.testnet3.goat.network`
- **Explorer**: `https://explorer.testnet3.goat.network`
- **Chain ID**: 48816 (GOAT Testnet3)

## Supported Tokens (GOAT Testnet3)

| Token | Contract Address | Decimals |
|-------|-----------------|----------|
| USDC  | `0x29d1ee93e9ecf6e50f309f498e40a6b42d352fa1` | 6 |
| USDT  | `0xdce0af57e8f2ce957b3838cd2a2f3f3677965dd3` | 6 |

## Faucet

Contract: `0x89e7dfd01a86e5393ce6d8A78c9aa6653Ee113A6` (GoatTestnetFaucet, ERC1967Proxy)

Claim test tokens:
```bash
cast send 0x89e7dfd01a86e5393ce6d8A78c9aa6653Ee113A6 \
  "claimToken(address)" <token_contract> \
  --rpc-url https://rpc.testnet3.goat.network \
  --private-key <PRIVATE_KEY>
```

Check eligibility:
```bash
cast call 0x89e7dfd01a86e5393ce6d8A78c9aa6653Ee113A6 \
  "canClaim(address,address)(bool,string)" <token_contract> <wallet_address> \
  --rpc-url https://rpc.testnet3.goat.network
```

## Step 1: Register Merchant

Ask the user for: `merchant_id`, `name`, `email`, `password`, `receive_type` (DIRECT or DELEGATE).

```bash
curl -s -X POST https://x402-api-lx58aabp0r.testnet3.goat.network/merchant/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "merchant_id": "<merchant_id>",
    "name": "<name>",
    "email": "<email>",
    "password": "<password>",
    "receive_type": "<DIRECT|DELEGATE>"
  }'
```

Registration requires **admin approval** before the account can be used. Registration alone does NOT activate the account — you must contact the platform admin to:
1. **Approve the merchant account** — without approval, login will fail
2. **Top up initial fee balance** — fee balance starts at $0, and orders cannot be created without sufficient fee balance (e.g., $0.05 per DIRECT order). The initial fee balance is topped up by the admin

Alternatively, use invite code registration (auto-approved, but initial fee balance still requires admin top-up):

```bash
curl -s -X POST https://x402-api-lx58aabp0r.testnet3.goat.network/merchant/v1/auth/register/invite \
  -H "Content-Type: application/json" \
  -d '{"invite_code":"<code>","email":"<email>","password":"<password>"}'
```

## Step 2: Login

```bash
curl -s -X POST https://x402-api-lx58aabp0r.testnet3.goat.network/merchant/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"<email>","password":"<password>"}'
```

Returns `access_token` (expires in 3600s) and `refresh_token`. Use the access_token as `Authorization: Bearer <token>` for subsequent requests.

## Step 3: Configure Receiving Address

Ask the user for: wallet address, chain, token.

```bash
curl -s -X POST https://x402-api-lx58aabp0r.testnet3.goat.network/merchant/v1/addresses \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "chain_id": 48816,
    "symbol": "USDC",
    "token_contract": "0x29d1ee93e9ecf6e50f309f498e40a6b42d352fa1",
    "address": "<wallet_address>"
  }'
```

To update, first remove the old address then add a new one:
```bash
curl -s -X DELETE https://x402-api-lx58aabp0r.testnet3.goat.network/merchant/v1/addresses/48816/USDC \
  -H "Authorization: Bearer <access_token>"
```

## Step 4: Generate API Keys

```bash
curl -s -X POST https://x402-api-lx58aabp0r.testnet3.goat.network/merchant/v1/api-keys/rotate \
  -H "Authorization: Bearer <access_token>"
```

Returns `api_key` and `api_secret`. These are used for HMAC-signed gateway API calls.

## Step 5: Fund Fee Balance

Creating orders requires merchant fee balance (e.g., $0.05 per DIRECT order on GOAT Testnet3).

Fee config query:
```bash
curl -s https://x402-api-lx58aabp0r.testnet3.goat.network/merchant/v1/fees/config \
  -H "Authorization: Bearer <access_token>"
```

**Initial fee balance is topped up by the platform admin.** If balance is $0, the merchant must contact the admin to allocate fee balance before creating orders. Without fee balance, order creation will fail with: `insufficient fee balance: available=$0.000000, required=$0.050000`.

Check balance:
```bash
curl -s https://x402-api-lx58aabp0r.testnet3.goat.network/merchant/v1/balance \
  -H "Authorization: Bearer <access_token>"
```

## Step 6: Create Order (HMAC-Signed)

Gateway API requires HMAC-SHA256 authentication. The signing process:

1. Collect request body fields + `api_key` + `timestamp` + `nonce`
2. Remove empty values, sort keys alphabetically
3. Build `k1=v1&k2=v2` string
4. HMAC-SHA256 with `api_secret`, hex-encode

```bash
API_KEY="<api_key>"
API_SECRET="<api_secret>"
TIMESTAMP=$(date +%s)
NONCE=$(uuidgen | tr '[:upper:]' '[:lower:]')

DAPP_ORDER_ID="dapp_${TIMESTAMP}"
CHAIN_ID="48816"
TOKEN_SYMBOL="USDC"
FROM_ADDRESS="<payer_address>"
AMOUNT_WEI="1000000"  # 1.0 USDC (6 decimals)

PAYLOAD="amount_wei=${AMOUNT_WEI}&api_key=${API_KEY}&chain_id=${CHAIN_ID}&dapp_order_id=${DAPP_ORDER_ID}&from_address=${FROM_ADDRESS}&nonce=${NONCE}&timestamp=${TIMESTAMP}&token_symbol=${TOKEN_SYMBOL}"

SIGN=$(echo -n "$PAYLOAD" | openssl dgst -sha256 -hmac "$API_SECRET")

curl -s -X POST "https://x402-api-lx58aabp0r.testnet3.goat.network/api/v1/orders" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ${API_KEY}" \
  -H "X-Timestamp: ${TIMESTAMP}" \
  -H "X-Nonce: ${NONCE}" \
  -H "X-Sign: ${SIGN}" \
  -d "{\"dapp_order_id\":\"${DAPP_ORDER_ID}\",\"chain_id\":${CHAIN_ID},\"token_symbol\":\"${TOKEN_SYMBOL}\",\"from_address\":\"${FROM_ADDRESS}\",\"amount_wei\":\"${AMOUNT_WEI}\"}"
```

Returns order with `order_id`, `payToAddress`, `amountWei`, and payment instructions.

## Step 7: Payment & Verification

After order creation, the payer transfers tokens to the `payToAddress`. The platform monitors on-chain ERC20 Transfer events and matches:
- `from` address = order's `from_address`
- `to` address = order's `payToAddress`
- Amount and token contract match
- Same chain

### Order Status Flow
```
CHECKOUT_VERIFIED → PAYMENT_CONFIRMED → INVOICED (complete)
                                      → FAILED
                 → EXPIRED
                 → CANCELLED
```

Query order status:
```bash
curl -s "https://x402-api-lx58aabp0r.testnet3.goat.network/api/v1/orders/<order_id>" \
  -H "X-API-Key: ${API_KEY}" \
  -H "X-Timestamp: ${TIMESTAMP}" \
  -H "X-Nonce: ${NONCE}" \
  -H "X-Sign: ${SIGN}"
```

## Payment Modes

Refer to the "DIRECT vs DELEGATE Mode" section at the top of this document for detailed comparison. Key takeaway:
- **DIRECT**: Simple transfer + event matching. Good for testing, but weaker security.
- **DELEGATE**: EIP-712 signature + TSS settlement + optional callbacks. Recommended for production.

## Other Merchant Portal APIs

All require `Authorization: Bearer <access_token>`:

| Action | Method | Endpoint |
|--------|--------|----------|
| Dashboard stats | GET | `/merchant/v1/dashboard/stats` |
| Profile | GET | `/merchant/v1/profile` |
| Update profile | PUT | `/merchant/v1/profile` |
| List orders | GET | `/merchant/v1/orders` |
| Order detail | GET | `/merchant/v1/orders/<order_id>` |
| List addresses | GET | `/merchant/v1/addresses` |
| Balance transactions | GET | `/merchant/v1/balance/transactions` |
| List webhooks | GET | `/merchant/v1/webhooks` |
| Create webhook | POST | `/merchant/v1/webhooks` |
| List invite codes | GET | `/merchant/v1/invite-codes` |
| Create invite code | POST | `/merchant/v1/invite-codes` |
| Audit logs | GET | `/merchant/v1/audit-logs` |
| Supported tokens | GET | `/merchant/v1/supported-tokens` |

## Handling $ARGUMENTS

If the user provides an action argument, jump directly to that step:
- `register` → Step 1
- `login` → Step 2
- `address` → Step 3
- `api-keys` → Step 4
- `fund` → Step 5
- `create-order` → Step 6
- `status` → Step 7
- No argument → Start from Step 1 and guide through the full flow

## Learn More

If you have further questions about x402 merchant integration, payment flows, or GOAT Network, refer to the project source code at https://github.com/GOATNetwork/agentkit for complete documentation, SDK usage, and examples.

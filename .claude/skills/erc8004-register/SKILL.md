---
name: erc8004-register
description: Register an ERC-8004 on-chain agent identity on GOAT Network, publish merchant info via Agent URI, and manage agent metadata and reputation. Use when the user wants to register an agent, update agent URI, set metadata, or query reputation on GOAT testnet or mainnet.
argument-hint: "[action]"
---

# ERC-8004 Agent Registration & Management (GOAT Network)

> **Testnet vs Mainnet**: This skill supports both GOAT Testnet3 and Mainnet. Testnet tokens have no real value. On mainnet, the user must fund the wallet with real BTC for gas fees.

Guide the user through ERC-8004 agent registration, merchant info publishing, and identity management on GOAT Network.

## Overview

ERC-8004 is an on-chain agent identity and reputation standard. It consists of two registries:

- **Identity Registry** — Register agents, manage Agent URI and metadata
- **Reputation Registry** — Submit, query, and revoke feedback/ratings for agents

By registering an ERC-8004 agent and setting the Agent URI to your x402 merchant info, other agents and users can discover your merchant on-chain.

## Contract Addresses

### GOAT Testnet3 (Chain ID: 48816)

| Contract | Address |
|----------|---------|
| Identity Registry | `0x556089008Fc0a60cD09390Eca93477ca254A5522` |
| Reputation Registry | `0xd9140951d8aE6E5F625a02F5908535e16e3af964` |

- **RPC**: `https://rpc.testnet3.goat.network`
- **Explorer**: `https://explorer.testnet3.goat.network`
- **Faucet (Web)**: https://bridge.testnet3.goat.network/faucet
- **Faucet (Contract)**: `0x89e7dfd01a86e5393ce6d8A78c9aa6653Ee113A6`

### GOAT Mainnet (Chain ID: 2345)

| Contract | Address |
|----------|---------|
| Identity Registry | `0x8004A169FB4a3325136EB29fA0ceB6D2e539a432` |
| Reputation Registry | `0x8004BAa17C55a88189AE136b182e5fdA19dE9b63` |

- **RPC**: `https://rpc.goat.network`
- **Explorer**: `https://explorer.goat.network`

## Prerequisites

### 1. EVM Wallet

You need an EVM wallet with BTC (native gas token on GOAT Network). Install the evm-wallet-skill with GOAT Network support:

```bash
# Clone and install
git clone https://github.com/eigmax/evm-wallet-skill.git .claude/skills/evm-wallet-skill
cd .claude/skills/evm-wallet-skill && npm install
```

Generate a wallet (only needed once):
```bash
SKILL_DIR=".claude/skills/evm-wallet-skill"
cd "$SKILL_DIR" && node src/setup.js --json
```

This creates a wallet at `~/.evm-wallet.json` and returns the wallet address. Supported chains include `goat` (mainnet) and `goat-testnet` (testnet3).

### 2. Fund the Wallet

Registering an agent requires gas (approximately 0.0004 BTC for registration + 0.00025 BTC for setAgentURI).

**Testnet** — Use the web faucet: https://bridge.testnet3.goat.network/faucet
- Paste your wallet address and claim multiple times until you have at least **0.001 BTC**
- Faucet contract (`0x89e7dfd01a86e5393ce6d8A78c9aa6653Ee113A6`) also supports:
  - `claimNative()` — claim testnet BTC
  - `claimToken(address)` — claim testnet ERC20 tokens (USDC/USDT)

**Mainnet** — User must transfer real BTC to the wallet address. There is no faucet on mainnet.

### 3. (Optional) Testnet ERC20 Tokens

| Token | Contract Address | Decimals |
|-------|-----------------|----------|
| USDC | `0x29d1ee93e9ecf6e50f309f498e40a6b42d352fa1` | 6 |
| USDT | `0xdce0af57e8f2ce957b3838cd2a2f3f3677965dd3` | 6 |

## Step 1: Register Agent

Register a new agent with an initial Agent URI. The URI should point to a JSON file describing the agent (see Step 2 for the JSON format).

You can use a placeholder URI first and update it later.

**Using evm-wallet-skill:**
```bash
SKILL_DIR=".claude/skills/evm-wallet-skill"
cd "$SKILL_DIR" && node src/contract.js goat-testnet \
  0x556089008Fc0a60cD09390Eca93477ca254A5522 \
  "register(string)" "<agent_uri>" \
  --yes --json
```

**Using cast (with private key):**
```bash
cast send 0x556089008Fc0a60cD09390Eca93477ca254A5522 \
  "register(string)" "<agent_uri>" \
  --rpc-url https://rpc.testnet3.goat.network \
  --private-key <PRIVATE_KEY>
```

**Returns:** Transaction hash. The `agentId` (uint256) is emitted in the transaction logs as the third topic of the Transfer event.

**Extract agentId from tx receipt:**
```bash
cast receipt <tx_hash> --rpc-url https://rpc.testnet3.goat.network
# Look for topics in logs — the agentId is the last topic of the Transfer event (hex → decimal)
```

**Verify registration:**
```bash
cast call 0x556089008Fc0a60cD09390Eca93477ca254A5522 \
  "getAgentWallet(uint256)(address)" <agentId> \
  --rpc-url https://rpc.testnet3.goat.network
```

## Step 2: Create & Publish Agent URI (Merchant Info)

The Agent URI is the key to making your agent discoverable on-chain. For x402 merchants, it should contain your merchant information so other agents can find and pay you.

### Agent URI JSON Template

```json
{
  "name": "<merchant_display_name>",
  "description": "<what your merchant does>",
  "type": "merchant",
  "version": "1.0.0",
  "erc8004": {
    "agentId": <your_agent_id>,
    "network": "goat-testnet",
    "identityRegistry": "0x556089008Fc0a60cD09390Eca93477ca254A5522",
    "reputationRegistry": "0xd9140951d8aE6E5F625a02F5908535e16e3af964"
  },
  "x402": {
    "merchantId": "<your_merchant_id>",
    "receiveType": "DIRECT",
    "portalApi": "https://x402-api-lx58aabp0r.testnet3.goat.network",
    "gatewayApi": "https://x402-api-lx58aabp0r.testnet3.goat.network/api/v1",
    "supportedPayments": [
      {
        "chainId": 48816,
        "chainName": "GOAT Testnet3",
        "token": "USDC",
        "tokenContract": "0x29d1ee93e9ecf6e50f309f498e40a6b42d352fa1",
        "decimals": 6,
        "receivingAddress": "<your_receiving_address>"
      }
    ]
  },
  "contact": {
    "email": "<your_email>"
  }
}
```

### Hosting the JSON

The Agent URI must be a publicly accessible URL. Options:

1. **GitHub Gist (recommended for testing)**:
   ```bash
   gh gist create --public -f "agent.json" agent.json
   # Use the raw URL: https://gist.githubusercontent.com/<user>/<gist_id>/raw/agent.json
   ```

2. **IPFS** (recommended for production):
   ```
   ipfs://QmXxxxx...
   ```

3. **Your own server**:
   ```
   https://your-domain.com/agent.json
   ```

> **Tip**: Use shorter URLs to save gas when writing on-chain. IPFS hashes or short URLs are more gas-efficient than long GitHub Gist URLs.

## Step 3: Update Agent URI

After creating and hosting the JSON, update the on-chain Agent URI:

**Using evm-wallet-skill:**
```bash
SKILL_DIR=".claude/skills/evm-wallet-skill"
cd "$SKILL_DIR" && node src/contract.js goat-testnet \
  0x556089008Fc0a60cD09390Eca93477ca254A5522 \
  "setAgentURI(uint256,string)" <agentId> "<new_uri>" \
  --yes --json
```

**Using cast:**
```bash
cast send 0x556089008Fc0a60cD09390Eca93477ca254A5522 \
  "setAgentURI(uint256,string)" <agentId> "<new_uri>" \
  --rpc-url https://rpc.testnet3.goat.network \
  --private-key <PRIVATE_KEY>
```

## Step 4: Set Metadata (Optional)

Store additional key-value metadata on-chain for your agent:

```bash
SKILL_DIR=".claude/skills/evm-wallet-skill"
cd "$SKILL_DIR" && node src/contract.js goat-testnet \
  0x556089008Fc0a60cD09390Eca93477ca254A5522 \
  "setMetadata(uint256,string,bytes)" <agentId> "<key>" "<hex_value>" \
  --yes --json
```

Read metadata:
```bash
cast call 0x556089008Fc0a60cD09390Eca93477ca254A5522 \
  "getMetadata(uint256,string)(bytes)" <agentId> "<key>" \
  --rpc-url https://rpc.testnet3.goat.network
```

## Step 5: Reputation Management (Optional)

### Submit Feedback for Another Agent

```bash
SKILL_DIR=".claude/skills/evm-wallet-skill"
cd "$SKILL_DIR" && node src/contract.js goat-testnet \
  0xd9140951d8aE6E5F625a02F5908535e16e3af964 \
  "giveFeedback(uint256,int128,uint8,string,string,string,string,bytes32)" \
  <agentId> <value> <valueDecimals> "<tag1>" "<tag2>" "<endpoint>" "<feedbackURI>" "<feedbackHash>" \
  --yes --json
```

Parameters:
- `value` / `valueDecimals`: Score (e.g., value=87, decimals=0 = score of 87; value=9977, decimals=2 = 99.77%)
- `tag1`, `tag2`: Category tags (e.g., "quality", "speed")
- `feedbackHash`: bytes32 hash of feedback content

### Query Reputation

```bash
cast call 0xd9140951d8aE6E5F625a02F5908535e16e3af964 \
  "getSummary(uint256,address[],string,string)(uint64,int128,uint8)" \
  <agentId> "[]" "<tag1>" "<tag2>" \
  --rpc-url https://rpc.testnet3.goat.network
```

Returns: `(count, summaryValue, summaryValueDecimals)`

### Get Clients Who Gave Feedback

```bash
cast call 0xd9140951d8aE6E5F625a02F5908535e16e3af964 \
  "getClients(uint256)(address[])" <agentId> \
  --rpc-url https://rpc.testnet3.goat.network
```

### Revoke Feedback

```bash
SKILL_DIR=".claude/skills/evm-wallet-skill"
cd "$SKILL_DIR" && node src/contract.js goat-testnet \
  0xd9140951d8aE6E5F625a02F5908535e16e3af964 \
  "revokeFeedback(uint256,uint64)" <agentId> <feedbackIndex> \
  --yes --json
```

## Quick Reference

### Read Operations (No Gas)

| Action | Command |
|--------|---------|
| Get agent wallet | `cast call <identity> "getAgentWallet(uint256)(address)" <agentId>` |
| Get metadata | `cast call <identity> "getMetadata(uint256,string)(bytes)" <agentId> "<key>"` |
| Get reputation | `cast call <reputation> "getSummary(uint256,address[],string,string)(uint64,int128,uint8)" <agentId> "[]" "" ""` |
| Get clients | `cast call <reputation> "getClients(uint256)(address[])" <agentId>` |

### Write Operations (Requires Gas)

| Action | Estimated Gas | Estimated Cost |
|--------|--------------|----------------|
| `register(string)` | ~156,000 | ~0.0004 BTC |
| `setAgentURI(uint256,string)` | ~123,000 | ~0.00025 BTC |
| `setMetadata(uint256,string,bytes)` | ~80,000 | ~0.00016 BTC |
| `giveFeedback(...)` | ~150,000 | ~0.0003 BTC |
| `revokeFeedback(...)` | ~50,000 | ~0.0001 BTC |

> Gas costs vary with URI/data length. Shorter strings = less gas.

## Handling $ARGUMENTS

If the user provides an action argument, jump directly to that step:
- `register` → Step 1: Register Agent
- `uri` → Step 2-3: Create & Update Agent URI
- `metadata` → Step 4: Set Metadata
- `reputation` → Step 5: Reputation Management
- `query` → Quick Reference read operations
- No argument → Start from Step 1 and guide through the full flow

## Learn More

If you have further questions about ERC-8004 agent identity, contract details, or integration with x402 payments, refer to the project source code at https://github.com/GOATNetwork/agentkit for complete documentation, contract ABIs, and examples.

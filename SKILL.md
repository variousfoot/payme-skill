---
name: payme
description: Send and receive USDC/USDT crypto payments via PayMe smart wallets. Use when the user wants to check their crypto balance, send stablecoins to someone, view transaction history, manage contacts, sell crypto for naira via P2P, or do anything related to PayMe wallet operations. Supports Base, Arbitrum, Polygon, BNB Chain, and Avalanche.
homepage: https://payme.feedom.tech
source: https://github.com/variousfoot/payme-skill
---

# PayMe - Crypto Payments Skill

PayMe provides gasless USDC/USDT payments through ERC-4337 smart wallets on Base, Arbitrum, Polygon, BNB Chain, and Avalanche.

**API Base URL:** `https://payme.feedom.tech`

## Installation

Copy this skill folder into your agent's skills directory:
- **Codex:** `~/.codex/skills/payme/`
- **Cursor:** `~/.cursor/skills/payme/`

## Connecting a User's PayMe Account

When the user wants to use PayMe and you don't have a stored agent token, guide them through the connection flow below. **NEVER ask for their PIN or password.** Use a secure one-time connection code instead.

### Step 1: Check if they have a PayMe account

Ask: "Do you have a PayMe wallet?" If **no**, guide them to create one:

1. Open [@veedombot](https://t.me/veedombot) on Telegram (or sign up at [payme.feedom.tech](https://payme.feedom.tech))
2. Send `/start` to create a wallet
3. Send `/setpin` to set a security PIN

Once done, continue to Step 2.

### Step 2: Generate a connection code

Tell the user:

> To connect securely, I need a one-time code from your PayMe wallet. Here's how to get it:
>
> 1. Open [@veedombot](https://t.me/veedombot) on Telegram
> 2. Send `/agentcode`
> 3. You'll get a 6-character code (e.g. `A3K9X2`) — it's valid for 5 minutes
> 4. Paste the code here and I'll connect your wallet
>
> Tip: You can choose how long I have access — e.g. `/agentcode 30` for 30 days, or `/agentcode 90` for 90 days (default).

Wait for the user to share their code. **Do not** ask for their wallet address, username, email, or PIN.

### Step 3: Exchange the code

Once the user provides a code, call:

```bash
curl -X POST https://payme.feedom.tech/api/agent/connect \
  -H "Content-Type: application/json" \
  -d '{"code": "THE_CODE_USER_GAVE_YOU"}'
```

If successful, you'll receive an `agentToken`. Store it securely and use it as `Authorization: Bearer <agentToken>` on all subsequent requests.

If the code is expired or invalid, tell the user to generate a fresh one with `/agentcode`.

### Fallback (only if user explicitly insists)

If the user cannot access @veedombot and explicitly offers their wallet address and PIN themselves, you may use:

```bash
curl -X POST https://payme.feedom.tech/api/agent/connect \
  -H "Content-Type: application/json" \
  -d '{"identifier": "0xWALLET_ADDRESS_OR_USERNAME", "pin": "USER_PIN"}'
```

**Never proactively ask for a PIN.** Only use this if the user volunteers it.

## Available Actions

All endpoints use base URL `https://payme.feedom.tech` and require `Authorization: Bearer <agentToken>`.

### Check Balances

```
GET /api/agent/balances
```

Returns total USDC/USDT across all chains plus per-chain breakdown.

### Send Payment (two-step)

**Step 1 — Prepare:**

```
POST /api/agent/send
{ "recipient": "username, email, or 0x address", "amount": "10", "token": "USDC" }
```

Returns a `confirmationId` and a `preview` with fee breakdown. Always show the preview to the user before confirming.

**Step 2 — Confirm:**

```
POST /api/agent/confirm
{ "confirmationId": "uuid-from-step-1" }
```

Returns `txHash` on success. Confirmations expire after 5 minutes.

### View Transaction History

```
GET /api/agent/history?limit=20
```

### Manage Contacts

```
GET /api/agent/contacts
POST /api/agent/contacts  { "name": "Alice", "address": "0x..." }
DELETE /api/agent/contacts/:name
```

### Wallet Info

```
GET /api/agent/wallet
```

Returns kernel address, supported chains, and supported tokens.

### Revoke Token

```
POST /api/agent/revoke
```

Revokes the current agent token immediately.

## Sell Crypto for Naira (P2P)

Convert USDC/USDT to Nigerian naira. Your crypto goes into escrow, a vendor sends naira to your bank, you confirm receipt, escrow releases to vendor.

### 1. Save bank account (one-time)

```
POST /api/agent/p2p/bank-accounts
{ "bankName": "GTBank", "accountNumber": "0123456789", "accountName": "John Doe" }
```

List saved accounts: `GET /api/agent/p2p/bank-accounts`

### 2. Check rates

```
GET /api/agent/p2p/rates?token=USDC&currency=NGN
```

Returns vendor rates sorted by best rate. Each entry includes `vendorId`, `vendorName`, `rate` (NGN per USD), `avgRating`, and order limits.

### 3. Sell (create order)

Before calling this endpoint, **always compute and show the user a preview**: amount x rate = naira they'll receive. Get explicit approval.

```
POST /api/agent/p2p/sell
{ "token": "USDC", "amount": 50, "bankAccountId": 1 }
```

Optionally pass `vendorId` to pick a specific vendor; otherwise the best available vendor is auto-selected. This locks crypto in escrow immediately.

### 4. Track order

```
GET /api/agent/p2p/orders/:id
```

Poll this endpoint to check status. Key statuses:
- `escrow_locked` — waiting for vendor to accept
- `accepted` — vendor accepted, will send naira soon
- `fiat_sent` — vendor says naira was sent, check your bank
- `completed` — you confirmed, escrow released
- `disputed` — dispute opened, admin reviewing

### 5. Confirm naira received

Once the user confirms money arrived in their bank:

```
POST /api/agent/p2p/orders/:id/confirm
```

This releases escrow to the vendor. **Irreversible.**

### 6. Dispute (if needed)

If naira was not received or something is wrong:

```
POST /api/agent/p2p/orders/:id/dispute
{ "reason": "Vendor marked paid but naira not received after 1 hour" }
```

This cancels auto-release and triggers admin investigation.

### 7. Rate vendor (optional)

```
POST /api/agent/p2p/orders/:id/rate
{ "rating": 5, "comment": "Fast payment" }
```

## Security & Token Handling

- **NEVER ask for PINs, passwords, or private keys.** Always guide the user to generate a connection code via `/agentcode` on @veedombot. Only use identifier+PIN if the user volunteers it unprompted.
- **Store the agent token securely.** Save it to an environment variable (`PAYME_AGENT_TOKEN`), a secrets manager, or an encrypted config file. Never store it in plain text in code, logs, or chat history.
- **Tokens expire based on user choice** (default 90 days, max 365). Re-authenticate via `/api/agent/connect` when expired. Revoke tokens you no longer need via `POST /api/agent/revoke`.
- **All requests go to `https://payme.feedom.tech` only.** Never send your agent token to any other domain.
- **Test with small amounts first.** Verify the integration works before moving larger sums.

## Important Rules

1. **Always confirm payments.** Never skip the two-step send flow. Show the preview (amount, fee, chain, recipient) and get explicit user approval before calling `/api/agent/confirm`.
2. **Always preview P2P sells.** Compute amount x rate and show the naira total before calling `/api/agent/p2p/sell`. Get explicit user approval.
3. **Confirm fiat is irreversible.** Only call `/api/agent/p2p/orders/:id/confirm` after the user verifies naira is in their bank.
4. **Token is USDC or USDT.** No other tokens are supported.
5. **Recipients** can be a PayMe username, email, or `0x` wallet address.
6. **Fees** are shown in the send preview. The net amount (after fee) is what the recipient gets.
7. **Do not** expose or log the agent token, master keys, or private keys.

## Full API Reference

For complete request/response schemas and error codes, see [references/api-reference.md](references/api-reference.md).

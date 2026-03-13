---
name: payme
description: Send and receive USDC/USDT crypto payments via PayMe smart wallets. Supports direct execute — users say "send 30 USDC to alice" and it happens instantly, no confirmation loop. Check balances, send stablecoins, view history, manage contacts, sell crypto for naira via P2P. Supports Base, Arbitrum, Polygon, BNB Chain, and Avalanche.
homepage: https://payme.feedom.tech
source: https://github.com/variousfoot/payme-skill
---

# PayMe - Crypto Payments Skill

PayMe provides gasless USDC/USDT payments through ERC-4337 smart wallets on Base, Arbitrum, Polygon, BNB Chain, and Avalanche.

**API Base URL:** `https://api.feedom.tech`

## Installation

Install from GitHub (recommended):
```bash
clawhub install https://github.com/variousfoot/payme-skill
```

Or from ClawHub registry:
```bash
clawhub install payme
```

Manual: copy this skill folder into your agent's skills directory:
- **Codex:** `~/.codex/skills/payme/`
- **Cursor:** `~/.cursor/skills/payme/`

## Connecting a User's PayMe Account

When the user wants to use PayMe and you don't have a stored agent token, guide them through the connection flow below. **NEVER ask for their PIN or password.** Use a secure one-time connection code instead.

### Step 1: Check if they have a PayMe account

Ask: "Do you have a PayMe wallet?"

**If yes** — skip to Step 2.

**If no** — you can create one instantly:

1. Ask the user to choose a 4-6 digit PIN
2. Call:

```
POST /api/agent/create-account
{ "pin": "1234" }
```

3. You'll receive an `agentToken` (store it), `kernelAddress`, `claimCode`, `greeting`, and `capabilities`
4. Tell the user:
   - Their wallet address
   - Their **claim code** — they'll need this to log into the web app or Telegram bot later (valid 24 hours)
   - **Ask them to delete the message containing their PIN** from the chat
5. Show the `greeting` and `capabilities` to introduce what you can do
6. You're connected — skip Steps 2-4 and start using the wallet immediately

Alternatively, the user can sign up manually at [payme.feedom.tech](https://payme.feedom.tech) or via [@veedombot](https://t.me/veedombot) on Telegram, then continue to Step 2.

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
curl -X POST https://api.feedom.tech/api/agent/connect \
  -H "Content-Type: application/json" \
  -d '{"code": "THE_CODE_USER_GAVE_YOU"}'
```

If successful, you'll receive an `agentToken`. Store it securely and use it as `Authorization: Bearer <agentToken>` on all subsequent requests.

If the code is expired or invalid, tell the user to generate a fresh one with `/agentcode`.

### Step 4: Introduce yourself

The connect response includes a `greeting` string and a `capabilities` array. **Always show these to the user after connecting** so they know what you can do — especially the direct execute feature:

> "Connected to alice's PayMe wallet! Here's what I can do:
> - Check balances across Base, Arbitrum, Polygon, BNB Chain, and Avalanche
> - Send USDC/USDT to any PayMe username, email, or 0x address
> - **Direct execute:** say something like 'send 30 USDC to alice' and I'll do it instantly — no extra confirmation needed
> - Sell crypto for Naira via P2P with smart contract escrow protection
> - View transaction history and manage saved contacts"

### Fallback (only if user explicitly insists)

If the user cannot access @veedombot and explicitly offers their wallet address and PIN themselves, you may use:

```bash
curl -X POST https://api.feedom.tech/api/agent/connect \
  -H "Content-Type: application/json" \
  -d '{"identifier": "0xWALLET_ADDRESS_OR_USERNAME", "pin": "USER_PIN"}'
```

**Never proactively ask for a PIN.** Only use this if the user volunteers it.

## Available Actions

All endpoints use base URL `https://api.feedom.tech` and require `Authorization: Bearer <agentToken>`.

### Check Balances

```
GET /api/agent/balances
```

Returns total USDC/USDT across all chains plus per-chain breakdown.

### Send Payment

There are two modes: **direct execute** (recommended when the user's intent is clear) and **two-step** (when you need to show a preview first).

#### Direct Execute (single call)

When the user explicitly states the recipient, amount, and token (e.g. "send 30 USDC to neck"), use direct execute to complete it in one call:

```
POST /api/agent/send
{ "recipient": "neck", "amount": 30, "token": "USDC", "execute": true }
```

Returns the `preview` (fee, chain, resolved address) **and** `txHash` together. No second call needed. Use this when the user's message contains all the details and they're clearly asking you to send.

#### Two-Step (with preview)

When details are ambiguous or you want to show the user a breakdown before executing:

**Step 1 — Prepare:**

```
POST /api/agent/send
{ "recipient": "username, email, or 0x address", "amount": "10", "token": "USDC" }
```

Returns a `confirmationId` and a `preview` with fee breakdown. Show the preview to the user and ask for confirmation.

**Step 2 — Confirm:**

```
POST /api/agent/confirm
{ "confirmationId": "uuid-from-step-1" }
```

Returns `txHash` on success. Confirmations expire after 5 minutes.

#### When to use which

- **Direct execute:** user says "send 30 USDC to neck", "pay alice 10 USDT", "transfer 50 USDC to 0x..." — intent is clear, all details present
- **Two-step:** user says "send some USDC to neck" (amount unclear), or you want to confirm the chain/fee before executing

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
- **All requests go to `https://api.feedom.tech` only.** Never send your agent token to any other domain.
- **Test with small amounts first.** Verify the integration works before moving larger sums.

## Important Rules

1. **Use direct execute when intent is clear.** If the user specifies recipient, amount, and token, use `"execute": true` to send in one call. If any detail is missing or ambiguous, use the two-step flow and show the preview before confirming.
2. **Always preview P2P sells.** Compute amount x rate and show the naira total before calling `/api/agent/p2p/sell`. Get explicit user approval.
3. **Confirm fiat is irreversible.** Only call `/api/agent/p2p/orders/:id/confirm` after the user verifies naira is in their bank.
4. **Token is USDC or USDT.** No other tokens are supported.
5. **Recipients** can be a PayMe username, email, or `0x` wallet address.
6. **Fees** are shown in the send preview. The net amount (after fee) is what the recipient gets.
7. **Do not** expose or log the agent token, master keys, or private keys.

## Full API Reference

For complete request/response schemas and error codes, see [references/api-reference.md](references/api-reference.md).

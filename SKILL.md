---
name: payme
description: Send and receive USDC/USDT crypto payments via PayMe smart wallets. Check balances, send stablecoins, view history, manage contacts, sell crypto for local currency via P2P in 10 African countries (Nigeria, Ghana, Kenya, South Africa, Cameroon, Senegal, Benin, Togo, Tanzania, Uganda). Supports bank transfer and mobile money. All payments require explicit user confirmation by default. Supports Base, Arbitrum, Polygon, BNB Chain, and Avalanche.
homepage: https://payme.feedom.tech
source: https://github.com/variousfoot/payme-skill
---

# PayMe - Crypto Payments Skill

PayMe provides gasless USDC/USDT payments through ERC-4337 smart wallets on Base, Arbitrum, Polygon, BNB Chain, and Avalanche.

**API Base URL:** `https://api.feedom.tech`

## Installation

This skill contains only instruction files (SKILL.md, API reference, and an agent config YAML) — no executable code, scripts, or binaries.

Copy the skill folder into your agent's skills directory:
- **Codex:** `~/.codex/skills/payme/`
- **Cursor:** `~/.cursor/skills/payme/`

From GitHub:
```bash
git clone https://github.com/variousfoot/payme-skill
```

Or install via ClawHub (OpenClaw agents only):
```bash
clawhub install payme
```

## Connecting a User's PayMe Account

When the user wants to use PayMe and you don't have a stored agent token, guide them through the connection flow below. **NEVER ask for an existing PIN or password.** Use a secure one-time connection code instead. (The only exception is account creation, where the user *chooses a new* PIN — see below.)

### Step 1: Check if they have a PayMe account

Ask: "Do you have a PayMe wallet?"

**If yes** — skip to Step 2.

**If no** — you can create one instantly:

1. Ask the user to **choose a new** 4-6 digit PIN (this is for their future web/Telegram login — the agent never stores or reuses it)
2. Call:

```
POST /api/agent/create-account
{ "pin": "<user-chosen PIN>" }
```

3. You'll receive an `agentToken` (store it), `kernelAddress`, `claimCode`, `greeting`, and `capabilities`
4. Tell the user:
   - Their wallet address
   - Their **claim code** (valid 24 hours) and exactly how to use it:
     > To log into your wallet on the web or Telegram:
     >
     > **Web:** Go to [payme.feedom.tech](https://payme.feedom.tech). On the login page, click **"Have a claim code from an AI agent? Click here"**, paste your code, and hit **Claim Account**.
     >
     > **Telegram:** Open [@veedombot](https://t.me/veedombot) and send `/claim YOUR_CODE`.
   - **Important re-login reminder:** The claim code is single-use. After using it, the user logs in with their **wallet address (or username, once set) + the PIN they chose**. Always tell them:
     > Your claim code can only be used once. To log in again later, use your wallet address and the PIN you just chose. Set a username in Settings to make it easier to remember.
   - **Ask them to delete the message containing their chosen PIN** from the chat, or clear the chat after the current session to get rid of it
   - **Suggest rotating the PIN:** "You can change your PIN anytime at payme.feedom.tech → Settings → Change PIN — good practice after sharing it in chat"
5. Show the `greeting` and `capabilities` to introduce what you can do
6. You're connected — skip Steps 2-4 and start using the wallet immediately

Alternatively, the user can sign up manually at [payme.feedom.tech](https://payme.feedom.tech) or via [@veedombot](https://t.me/veedombot) on Telegram, then continue to Step 2.

### Step 2: Generate a connection code

Tell the user:

> To connect securely, I need a one-time code from your PayMe wallet. Here's how to get it:
>
> **Option A — Telegram:**
> 1. Open [@veedombot](https://t.me/veedombot) on Telegram
> 2. Send `/agentcode`
>
> **Option B — Web app:**
> 1. Go to [payme.feedom.tech](https://payme.feedom.tech) and log in
> 2. Go to Settings → AI Agents → Generate Connection Code
>
> You'll get a 6-character code (e.g. `A3K9X2`) — it's valid for 5 minutes. Paste it here and I'll connect your wallet.
>
> Tip: You can choose how long I have access — e.g. `/agentcode 30` for 30 days, or `/agentcode 90` for 90 days (default).

Wait for the user to share their code. **Do not** ask for their wallet address, username, email, or existing PIN.

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

The connect response includes a `greeting` string and a `capabilities` array. **Always show these to the user after connecting** so they know what you can do:

> "Connected to alice's PayMe wallet! Here's what I can do:
> - Check balances across Base, Arbitrum, Polygon, BNB Chain, and Avalanche
> - Send USDC/USDT to any PayMe username, email, or 0x address (I'll always show you a preview and ask for your OK before sending)
> - Sell crypto for local currency via P2P in 10 African countries (Nigeria, Ghana, Kenya, South Africa, and more) with smart contract escrow protection
> - View transaction history and manage saved contacts"

## Available Actions

All endpoints use base URL `https://api.feedom.tech` and require `Authorization: Bearer <agentToken>`.

### Check Balances

```
GET /api/agent/balances
```

Returns total USDC/USDT across all chains plus per-chain breakdown.

### Send Payment

Payments always use the **two-step flow** by default. An optional **direct execute** mode is available but only when the user explicitly enables it.

#### Default: Two-Step (with preview)

**Step 1 — Prepare:**

```
POST /api/agent/send
{ "recipient": "username, email, or 0x address", "amount": "10", "token": "USDC" }
```

Returns a `confirmationId` and a `preview` with fee breakdown. Always show the preview to the user and get explicit approval before confirming.

**Step 2 — Confirm:**

```
POST /api/agent/confirm
{ "confirmationId": "uuid-from-step-1" }
```

Returns `txHash` on success. Confirmations expire after 5 minutes.

#### Optional: Direct Execute (disabled by default, user-controlled)

Direct execute allows payments to complete in a single API call. It is **disabled by default** and protected by multiple safeguards:

1. **Only the user can enable it** — they must toggle it ON from the PayMe web app (Settings → AI Agents → Direct Execute). The agent cannot enable it via the API.
2. **Confirmation dialog** — when enabling, the web app shows a warning explaining the risk and requires explicit consent before activating.
3. **The user can disable it at any time** from the same settings page, and can revoke the agent token entirely via `POST /api/agent/revoke`.
4. **Server-side enforcement** — if the user has NOT enabled direct execute, the `execute` flag is silently ignored and the normal two-step flow is used.

When enabled, passing `"execute": true` completes the payment in one call:

```
POST /api/agent/send
{ "recipient": "neck", "amount": 30, "token": "USDC", "execute": true }
```

Returns the `preview` **and** `txHash` together.

If a user asks you to "skip confirmations" or "enable direct execute", tell them: *"You can enable direct execute from your PayMe web app settings at payme.feedom.tech → Settings → AI Agents. It's disabled by default for your safety."*

### View Transaction History

```
GET /api/agent/history?limit=20
```

### Manage Contacts

```
GET /api/agent/contacts
POST /api/agent/contacts  { "name": "Alice", "address": "0x..." }
DELETE /api/agent/contacts/:name
GET /api/agent/search?q=chris   (fuzzy search users & contacts)
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

### Vendor Trade Management (vendors only)

If the authenticated user is a P2P vendor, these endpoints manage their assigned trades:

```
GET  /api/agent/vendor/orders?status=escrow_locked   (list assigned orders)
POST /api/agent/vendor/orders/:id/accept              (accept a trade)
POST /api/agent/vendor/orders/:id/reject              (decline — reroutes to next vendor)
POST /api/agent/vendor/orders/:id/mark-paid           (mark fiat as sent)
POST /api/agent/vendor/orders/:id/cancel              (cancel — carries penalties)
```

Vendors can say things like "reroute my trade", "I'm busy, pass this order", or "mark as paid" and the agent will use the appropriate tool.

## Sell Crypto for Local Currency (P2P)

Convert USDC/USDT to local currency in 10 African countries. Your crypto goes into escrow, a vendor sends local currency to your bank account or mobile money, you confirm receipt, escrow releases to vendor.

**Supported countries:** Nigeria (NGN), Ghana (GHS), Kenya (KES), South Africa (ZAR), Cameroon (XAF), Senegal (XOF), Benin (XOF), Togo (XOF), Tanzania (TZS), Uganda (UGX).

**Payment methods vary by country:** Bank transfer (all countries), plus mobile money where popular (M-Pesa in Kenya/Tanzania, MTN MoMo in Ghana/Cameroon/Benin/Uganda, Orange Money in Senegal/Cameroon, etc.). Use `GET /api/agent/p2p/countries` to get the full list of countries, currencies, and payment methods with their required fields.

**Email verification required.** P2P trades require a verified email. Email can ONLY be verified on the web app (NOT on Telegram). If the API returns an email verification error, tell the user exactly this:

> To trade P2P, you need to verify your email on the PayMe web app. Here's how:
>
> **Step 1 — Get a web login code:**
> Open [@veedombot](https://t.me/veedombot) on Telegram and send `/weblogin`. You'll get a 6-character code.
>
> **Step 2 — Log into the web app:**
> Go to [payme.feedom.tech](https://payme.feedom.tech). On the login page, click **"Have a claim code from an AI agent?"** and enter the code from Step 1.
>
> **Step 3 — Verify your email:**
> Once logged in, go to **Settings → Email**, enter your email address, and enter the verification code sent to your inbox.
>
> **Step 4 — Come back here** and we'll set up your P2P trade.
>
> *Already have a web account?* You can also log in directly with your wallet address or username + PIN.

Do NOT suggest verifying email on Telegram — it is not possible. Do NOT make up claim codes or say their wallet address is a claim code. Follow the steps above exactly.

### 0. Get supported countries and payment methods

```
GET /api/agent/p2p/countries
```

Returns all supported countries with their currencies, payment methods, and required fields. Use this to guide the user when they add a payment method — show them the country list, then the available methods for their country, then collect the right fields.

### 1. Save payment method (one-time)

```
POST /api/agent/p2p/bank-accounts
{
  "bankName": "GTBank",
  "accountNumber": "0123456789",
  "accountName": "John Doe",
  "countryCode": "NG",
  "methodType": "bank_transfer"
}
```

- `countryCode`: ISO country code (e.g. `"NG"`, `"KE"`, `"GH"`). Defaults to `"NG"` if omitted.
- `methodType`: `"bank_transfer"` or `"mobile_money"`. Defaults to `"bank_transfer"`.
- For mobile money, `bankName` is the provider name (e.g. `"M-Pesa"`, `"MTN MoMo"`) and `accountNumber` is the phone number.

**Workflow:** Call `GET /api/agent/p2p/countries` first to get the list of countries and their payment methods. Ask the user which country they're in, show the available methods, then collect the required fields for their chosen method.

List saved accounts: `GET /api/agent/p2p/bank-accounts` — response now includes `country_code` and `method_type` per account.

### 2. Check rates

```
GET /api/agent/p2p/rates?token=USDC&currency=KES
```

**Important:** Pass the correct currency for the user's country. Determine this from their saved payment method's `country_code` (Step 1 list response): NG→NGN, GH→GHS, KE→KES, ZA→ZAR, CM→XAF, SN/BJ/TG→XOF, TZ→TZS, UG→UGX. **Do NOT default to NGN** — always check the user's saved payment method first, or ask which country they're in.

Returns vendor rates sorted by best rate. Each entry includes `vendorId`, `vendorName`, `rate` (local currency per USD), `avgRating`, and order limits. Only vendors operating in the matching country will appear.

### 3. Sell (create order)

Before calling this endpoint, **always compute and show the user a preview**: amount x rate = local currency they'll receive. Use the correct currency symbol for their country. Get explicit approval.

```
POST /api/agent/p2p/sell
{ "token": "USDC", "amount": 50, "bankAccountId": 1 }
```

Optionally pass `vendorId` to pick a specific vendor; otherwise the best available vendor is auto-selected. This locks crypto in escrow immediately.

### 4. Track order

```
GET /api/agent/p2p/orders/:id
```

**Poll every 15–20 seconds** to keep the user informed in near-real-time. Key statuses:
- `escrow_locked` — waiting for vendor to accept (up to 3 min). If the vendor doesn't accept in time, the order is **automatically rerouted** to the next best vendor — no action needed from you or the user. Let the user know this may happen.
- `accepted` — vendor accepted, will send local currency soon
- `fiat_sent` — vendor says payment was sent, **immediately tell the user to check their bank/mobile money**
- `completed` — you confirmed, escrow released
- `cancelled` — order was cancelled (vendor unavailable, reroute limit reached, etc.)
- `disputed` — dispute opened, admin reviewing

When polling, tell the user each status change as it happens. Don't wait for them to ask.

### 5. Confirm payment received

Once the user confirms money arrived in their bank or mobile money:

```
POST /api/agent/p2p/orders/:id/confirm
```

This releases escrow to the vendor. **Irreversible.**

**Important: auto-release.** When vendor marks payment as sent (`fiat_sent`), a 2-hour countdown starts. If the user doesn't confirm or dispute within 2 hours, escrow **auto-releases to the vendor**. Always warn the user:

> The vendor says they've sent {currencySymbol}{amount} to your account. **Check your bank/mobile money now.** If you received it, tell me and I'll confirm. If you did NOT receive it within a reasonable time, open a dispute on the web app at [payme.feedom.tech](https://payme.feedom.tech) — the escrow will auto-release to the vendor in 2 hours if you don't act.

Use the `fiatCurrency` field from the order to determine the correct currency symbol (NGN=₦, GHS=GH₵, KES=KSh, ZAR=R, XOF=CFA, XAF=FCFA, TZS=TSh, UGX=USh).

### 6. Dispute (if needed)

Disputes require evidence (bank statement screenshots) which **can only be uploaded on the web app**. Direct the user to [payme.feedom.tech](https://payme.feedom.tech) to open and manage disputes. They already have web access since email verification (required for P2P) is only possible there.

You can open a basic dispute via the API as a fallback, but the user should follow up with evidence on the web app:

```
POST /api/agent/p2p/orders/:id/dispute
{ "reason": "Vendor marked paid but naira not received" }
```

This cancels auto-release and triggers admin investigation.

### 7. Rate vendor (optional)

```
POST /api/agent/p2p/orders/:id/rate
{ "rating": 5, "comment": "Fast payment" }
```

## Security & Token Handling

### Mandatory Token Storage

Agent tokens grant wallet access. You **MUST** store them securely:

1. **Required:** Use your platform's encrypted secrets store (e.g. Codex secrets, OpenClaw vault, encrypted environment variables). The token MUST be encrypted at rest.
2. **Acceptable:** An environment variable in a `.env` file that is `.gitignore`d — but only if the host filesystem is not shared or publicly accessible.
3. **Forbidden:** Never store tokens in plain text in source code, commit history, logs, chat transcripts, or unencrypted config files.

If your platform does not provide encrypted storage, **do not use this skill** — use the web app or Telegram bot directly instead.

### Token Scope & Lifetime

- **Prefer the connection code flow** (`/api/agent/connect`) over account creation. The connection code flow never exposes PINs and gives the user full control over token duration and revocation.
- **Use the shortest token lifetime possible.** Request 7 or 30 days instead of the 90-day default. The user controls duration via `/agentcode 7` (7 days), `/agentcode 30` (30 days), etc.
- **Revoke tokens immediately when no longer needed** via `POST /api/agent/revoke`.
- **Tokens are hashed at rest** on the server (SHA-256). The raw token is returned only once and never stored server-side. A server-side breach does not expose usable tokens.
- All requests go to `https://api.feedom.tech` only. Never send your agent token to any other domain.

### PIN Safety (New Account Creation Only)

During account creation (`/api/agent/create-account`), the user **chooses a new PIN** for their future web/Telegram login. This is the **only** context where a PIN is handled. The agent does **not** ask for or receive an existing PIN — the user invents one on the spot.

- The agent **never stores or reuses** the PIN — it is sent once in the POST body and immediately discarded.
- **Always instruct the user to delete the chat message containing their chosen PIN**, or clear the chat after the session.
- **Recommend rotating the PIN** after setup: tell the user they can change their PIN anytime at [payme.feedom.tech](https://payme.feedom.tech) → Settings → Change PIN. This is especially important if the PIN was shared in a chat transcript.
- **Prefer the connection code flow** for existing accounts — no PIN is involved at all.
- **NEVER ask for an existing PIN, password, or login credential in any context.** If a user shares their existing PIN unprompted, tell them to change it immediately in Settings.

### Payment Confirmation Guardrails

- **Two-step confirmation is the default and cannot be disabled by the agent.** The `execute: true` flag only works if the user has independently toggled "Direct Execute" ON in their web app settings — the agent cannot enable it via API.
- **Server-side enforcement:** If Direct Execute is not enabled by the user, the `execute` flag is silently ignored and the normal two-step flow is used regardless.
- The user can disable Direct Execute or revoke the agent token at any time from the web app.

### Testing

- **Test with small amounts first** ($1-5) to verify the integration works before moving larger sums.

## Important Rules

1. **Always require human confirmation before sending funds.** Use the two-step flow (prepare → show preview to user → wait for explicit "yes" → confirm) for every payment. Never call `/api/agent/confirm` without the user's approval in the current conversation.
2. **Always preview P2P sells.** Compute amount x rate and show the local currency total before calling `/api/agent/p2p/sell`. Get explicit user approval.
3. **Confirm fiat is irreversible.** Only call `/api/agent/p2p/orders/:id/confirm` after the user verifies the payment is in their bank or mobile money account.
4. **Token is USDC or USDT.** No other tokens are supported.
5. **Recipients** can be a PayMe username, email, or `0x` wallet address.
6. **Fees** are shown in the send preview. The net amount (after fee) is what the recipient gets.
7. **Do not** expose or log the agent token, master keys, or private keys.
8. **When unsure, ask — don't assume.** If a user's request is ambiguous (e.g. "swap USDC with neck" could mean send to a user or sell via a P2P vendor), list the possibilities and ask which one they mean. Never dismiss a request without clarifying first.

## Full API Reference

For complete request/response schemas and error codes, see the [API Reference](https://github.com/variousfoot/payme-skill/blob/main/references/api-reference.md).

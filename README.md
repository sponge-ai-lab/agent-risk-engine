# agent-risk-engine

Risk checks that run before an agent payment is held in escrow, on any payment rail.

> **Status: design.** No code yet.

## What it does

- Before any money is locked, the settlement layer (for example [Misa](https://trymisa.com)) asks: **"Is it safe to hold this payment?"**
- The engine answers **allow**, or **deny with a reason** (for example `SELLER_EXPOSURE_LIMIT`).
- It keeps an in-memory view of every open hold, across all agents, sellers, verifiers, assets and rails.
- It never holds, signs or moves funds.

## What we check before every hold

**The request itself**
- The asset is supported, and the amount is positive and within the per-payment cap.
- The order is new: the same order ID cannot be held twice.
- The hold deadline is neither too short to deliver nor too long (funds locked for days).

**The buyer (agent)**
- Total held for this agent stays under its limit, per asset (for example, at most 500 XRP).
- Total held across all assets stays under its limit in USD (for example, at most $1,000).
- Order rate stays normal (for example, at most 20 orders per minute).

**The seller**
- Total held for this seller stays under its limit (for example, at most 10,000 USDC).
- New sellers get lower limits until they have a delivery record.
- Sellers whose deliveries often fail verification get lower limits or are paused.
- A sudden burst of new buyers is flagged (for example, more than 200 per minute), even when each buyer is within its own budget.

**The verifier (arbiter)**
- Total held waiting on one verifier stays under its limit, so an outage cannot freeze too much money.
- A verifier that is slow or missing deadlines stops receiving new holds.

**Across the network**
- No single seller or verifier holds too large a share of all open funds.
- Prices used for USD limits are fresh; a stale price means deny, never a guess.

## What we track after a hold

- **Authorized:** the amount is added to buyer, seller and verifier exposure.
- **Captured:** the seller was paid; exposure is released and the seller's delivery record improves.
- **Reclaimed or voided:** the buyer was refunded; exposure is released and counts against the seller.
- **Expired:** the deadline passed; the hold is queued for reclaim and stops counting toward limits.
- The same event applied twice changes nothing, and replaying the event log rebuilds the same state.

## How it handles different assets and rails

- Each asset is identified by network and asset, as in x402 (`xrpl:0` / `XRP`, `eip155:8453` / USDC, `stripe` / `usd`).
- Amounts are integers in each asset's smallest unit (drops, cents, wei), stored as 128-bit values. Different assets are never added directly.
- USD totals use a price feed with a haircut per asset (for example, XRP counted at 80% of its price).
- Each rail has a small adapter that turns its events (XRPL `EscrowCreate`, an EVM authorization, a Stripe manual-capture payment) into the events above.

## What it does not do

- Hold, sign or move funds.
- Replace wallet budgets: a wallet limits one agent, while this engine sees risk across all of them.
- Score fraud with machine learning; it applies explicit rules, and thresholds are configuration.

## Roadmap

1. **Core (C++20):** exposure per account and asset, limits, expiry queue, `check()` and `apply(event)`, unit tests.
2. **Replay and benchmarks:** rebuild from an event log, rate windows, USD limits, p50/p99 latency per check.
3. **Service:** an HTTP or gRPC interface that Misa calls before each hold.
4. **x402 adapters:** TypeScript hooks (`onBeforePaymentCreation`, `onBeforeVerify`, `onBeforeSettle`).

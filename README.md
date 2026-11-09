# agent-risk-engine

Risk checks that run before an agent payment is held in escrow, on any payment rail.

> **Status: design.** No code yet.

## What it does

- Before any money is locked, the settlement layer (for example [Misa](https://trymisa.com)) asks: **"Is it safe to hold this payment?"**
- The engine answers **allow**, **review**, or **deny with a reason** (for example `SELLER_EXPOSURE_LIMIT`).
- It keeps an in-memory view of every open hold, across all agents, sellers, verifiers, assets and rails.
- It never holds, signs or moves funds.

The checks below borrow from controls that already exist elsewhere: broker pre-trade risk checks, clearing-house risk management, card and marketplace fraud controls, and crypto settlement practice. Each section names where its idea comes from.

## What we check before every hold

**1. The request itself**
*Precedent: erroneous-order checks in broker pre-trade controls; known x402 attacks.*
- The asset and rail are supported, and the amount is positive.
- The amount is within the per-payment cap (a "fat finger" check).
- The price is close to what this seller usually charges for this resource; a sudden jump is flagged.
- The pay-to address is the one registered for this seller, so a swapped address is caught.
- The order is new: the same order ID cannot be held twice.
- The hold deadline is neither too short to deliver nor too long (funds locked for days).

**2. The buyer's authorization**
*Precedent: signed mandates in AP2, Visa Intelligent Commerce and Mastercard Agent Pay.*
- When the buyer supplies a mandate, the payment fits inside it: allowed sellers, maximum amount, total budget and expiry.
- A payment outside the mandate is denied before any funds move.

**3. The buyer (agent)**
*Precedent: credit limits in broker pre-trade controls; velocity rules in card fraud tools.*
- Total held for this agent stays under its limit, per asset (for example, at most 500 XRP).
- Total held across all assets stays under its limit in USD (for example, at most $1,000).
- Order rate and spend rate stay normal over sliding windows (for example, at most 20 orders per minute).
- Many tiny payments that add up to a large total are caught (budget drain).
- A sudden jump in the number of different sellers an agent pays is flagged.

**4. The seller**
*Precedent: reserves and payout delays for new accounts on marketplaces; dispute-ratio monitoring by card networks.*
- Total held for this seller stays under its limit (for example, at most 10,000 USDC).
- New sellers get lower limits until they have a delivery record.
- The share of a seller's deliveries that fail verification is tracked; above one threshold the seller is flagged, above a higher one it is paused.
- A sudden burst of new buyers is flagged (for example, more than 200 per minute), even when each buyer is within its own budget.

**5. The verifier (arbiter)**
*Precedent: concentration and operational limits in clearing houses.*
- Total held waiting on one verifier stays under its limit, so an outage cannot freeze too much money.
- A verifier that is slow or misses deadlines stops receiving new holds.

**6. The asset and the rail**
*Precedent: settlement finality and stablecoin practice.*
- Prices used for USD limits are fresh; a stale price means deny, never a guess.
- Each asset has a haircut (for example, XRP counted at 80% of its price).
- A stablecoin trading away from its peg beyond a threshold pauses new holds in it.
- A hold counts as open only once it is final on its rail (a validated ledger on XRPL, a finalized block on EVM chains); until then it is pending.
- Assets whose issuer can freeze transfers are tracked separately.

**7. Compliance**
*Precedent: sanctions screening at the point of payment.*
- Buyer and seller addresses are checked against a sanctions screening result before a hold. The engine consumes the result from a screening provider; it does not screen on its own.

**8. Across the network**
*Precedent: concentration limits and stress testing in clearing houses.*
- No single seller, verifier or asset holds too large a share of all open funds.
- A stress view reports the loss if the largest sellers or verifiers failed at once.
- A kill switch pauses a seller, agent, verifier, asset or whole rail immediately.

## What we track after a hold

- **Authorized:** the amount is added to buyer, seller and verifier exposure.
- **Captured:** the seller was paid; exposure is released and the seller's delivery record improves.
- **Reclaimed or voided:** the buyer was refunded; exposure is released and counts against the seller.
- **Expired:** the deadline passed; the hold is queued for reclaim and stops counting toward limits.
- The same event applied twice changes nothing, and replaying the event log rebuilds the same state.

## How rules are run

- Every rule is configuration, not code. Thresholds belong to the operator; for Misa they are private product settings.
- Each rule can **deny** (block) or **review** (allow, but flag). New rules start in review and become denies once false positives are understood.
- Every decision is logged with the rule and the values that triggered it, so any decision can be explained later.

## How it handles different assets and rails

- Each asset is identified by network and asset, as in x402 (`xrpl:0` / `XRP`, `eip155:8453` / USDC, `stripe` / `usd`).
- Amounts are integers in each asset's smallest unit (drops, cents, wei), stored as 128-bit values. Different assets are never added directly.
- Each rail has a small adapter that turns its events (XRPL `EscrowCreate`, an EVM authorization, a Stripe manual-capture payment) into the events above.

## What it does not do

- Hold, sign or move funds.
- Replace wallet budgets: a wallet limits one agent, while this engine sees risk across all of them.
- Screen sanctions lists itself, or score fraud with machine learning. It applies explicit rules and consumes signals from other systems.

## Roadmap

1. **Core (C++20):** exposure per account and asset, limits, expiry queue, `check()` and `apply(event)`, unit tests.
2. **Replay and benchmarks:** rebuild from an event log, rate windows, USD limits, p50/p99 latency per check.
3. **Service:** an HTTP or gRPC interface that Misa calls before each hold.
4. **x402 adapters:** TypeScript hooks (`onBeforePaymentCreation`, `onBeforeVerify`, `onBeforeSettle`).

## References

- [SEC Rule 15c3-5, Risk Management Controls for Brokers or Dealers with Market Access](https://www.federalregister.gov/documents/2010/01/26/2010-1269/risk-management-controls-for-brokers-or-dealers-with-market-access) and [FINRA guidance](https://www.finra.org/rules-guidance/key-topics/market-access)
- [CPMI-IOSCO, Resilience of central counterparties: further guidance on the PFMI](https://www.bis.org/cpmi/publ/d163.pdf)
- [Stripe Connect reserves](https://docs.stripe.com/connect/connected-account-reserves) and [Radar rules reference](https://docs.stripe.com/radar/rules/reference)
- [eBay payments for new sellers](https://pages.ebay.com.sg/pay/help/index.html)
- [Halborn, x402 security risks and controls](https://www.halborn.com/blog/post/x402-explained-security-risks-and-controls-for-http-402-micropayments) and [CSA research note on x402 SDK flaws](https://labs.cloudsecurityalliance.org/research/csa-research-note-x402-agent-payment-sdk-flaws-20261025-csa/)
- [Elliptic, sanctions screening in crypto](https://www.elliptic.co/sanctions-guide-gate.html)

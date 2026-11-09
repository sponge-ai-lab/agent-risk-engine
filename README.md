# agent-risk-engine

Settlement-side risk checks for agent payments: exposure, limits and velocity across every open hold, on any payment rail.

> **Status: design.** This repository holds the design only; no code yet.

## What we are building

A small C++ engine that the settlement layer asks one question before it locks any money:

> **"Is it safe to hold this payment?"**

It answers **allow** or **deny, with a reason**, in microseconds, from an in-memory view of every open hold. It never touches funds: custody, signing and settlement stay with the rail.

### Example

An agent asks to buy a dataset for 40 USDC from seller `S`, verified by arbiter `A`. Before the escrow is created, [Misa](https://trymisa.com) calls the engine:

```
check(agent=agent_42, seller=S, verifier=A, asset=USDC on Base, amount=40)
```

The engine looks at what is already held and replies:

```
deny: SELLER_EXPOSURE_LIMIT
  seller S already has 9,980 USDC held across 312 open escrows; limit is 10,000
```

No wallet could have caught this: each of those 312 escrows came from a different agent, and each one was within its own budget.

## Why the settlement layer, not the wallet

An agent wallet enforces its owner's budget: how much *this* agent may spend. It cannot see other agents, sellers or rails. The settlement layer holds the payments, so it is the one place that sees all of them at once.

| | Analogy | Question it answers | Sees |
| --- | --- | --- | --- |
| Agent wallet | A broker's pre-trade checks for one client | "May my agent make this payment?" | One agent |
| **This engine** | A clearing house's risk management | "Is it safe to hold and settle this payment?" | Every agent, seller, verifier and rail |

The engine does not replace wallet budgets. It can enforce simple buyer budgets for agents without a policy-aware wallet, but its main job is the risk no single wallet can see.

## Checks

| Check | Example rule | What it protects against |
| --- | --- | --- |
| **Seller exposure** | At most 10,000 USDC held for one seller | A fraudulent or failing seller taking many buyers down at once |
| **Verifier exposure** | At most $50,000 of holds waiting on one arbiter | An arbiter outage freezing a large share of open payments |
| **Buyer exposure** | At most 500 XRP held for one agent | Runaway agents; budgets for agents without smart wallets |
| **Aggregate exposure** | At most $1,000 per agent across all assets, priced conservatively | Spreading spend across assets to dodge per-asset limits |
| **Velocity** | At most 20 orders per agent per minute; at most 200 new buyers per seller per minute | Loops, scripted abuse, coordinated attacks on one seller |
| **Expiry** | Holds past their deadline leave exposure and are queued for reclaim | Funds staying locked, and limits being consumed by dead holds |

Every rule is configuration, not code. Which rules apply, and their thresholds, belong to the operator (for Misa, private product configuration).

## Model

- **Assets** are identified by network and asset, as in x402: CAIP-2 networks such as `xrpl:0` or `eip155:8453`, or `stripe` for card rails. Each asset has its own decimals.
- **Amounts** are integers in an asset's smallest unit (drops, cents, wei), stored as 128-bit values so 18-decimal assets fit. Amounts of different assets are never added directly.
- **Aggregate limits** convert to a reference currency through a price feed, with a haircut per asset (for example, XRP counted at 80% of its price). A stale price means deny, never a guess.
- **Events** are rail-agnostic and follow the x402 `auth-capture` lifecycle. Each rail has an adapter that translates its own events:

| Event | Exposure | XRPL | EVM | Stripe |
| --- | --- | --- | --- | --- |
| `Authorized` | + amount | `EscrowCreate` | authorize | manual-capture PaymentIntent |
| `Captured` | − amount (paid to seller) | `EscrowFinish` | capture | capture |
| `Reclaimed` / `Voided` | − amount (back to buyer) | `EscrowCancel` | reclaim / void | cancel |
| `Expired` | marked for reclaim | past `CancelAfter` | past deadline | authorization expired |

- **State is derived from events.** Replaying the settlement event log rebuilds the exact same state, so a restart or a second instance needs no separate database. Applying the same event twice has no effect.

## Shape

```
rail adapters (XRPL, EVM, Stripe) -> events -> risk core (C++) -> allow / deny + reason
                                                    ^
                     x402 hooks (TypeScript) -------+
                     onBeforePaymentCreation, onBeforeVerify, onBeforeSettle
```

- **Core:** a C++20 library with no dependency on any rail or protocol. It is exposed as a small service later.
- **Adapters:** thin TypeScript glue that calls the core from x402 hooks and from Misa's settlement engine.

## Non-goals

- Holding, signing or moving funds
- Fraud scoring with machine learning (the engine applies explicit rules; a model can feed it signals later)
- Replacing wallet-side budgets and allowlists

## Roadmap

| Milestone | Delivers | Done when |
| --- | --- | --- |
| **1. Core** | Exposure book per (account, asset), per-asset limits, expiry queue, `check()` and `apply(event)` | Unit tests cover every event type, limit boundaries and duplicate events |
| **2. Replay and benchmarks** | Rebuild from an event log; velocity windows; aggregate limits with a price feed | Replayed state equals live state; p50/p99 latency per `check()` published here |
| **3. Service** | HTTP or gRPC interface; Misa calls it before each hold | Misa denies a payment with a reason from the engine on testnet |
| **4. x402 adapters** | TypeScript hooks for buyers, resource servers and facilitators | An x402 client or server can add the engine with one hook |

## Open questions

- Should a deny be final, or should some reasons allow a smaller amount (partial approval)?
- How should limits behave when a verifier is slow but not down?
- Which price feed, and how stale may a price be before aggregate checks deny?

# agent-risk-engine

Settlement-side risk checks for agent payments: exposure, limits and velocity across every open hold, on any payment rail.

> **Status: design.** This repository holds the design only; no code yet.

## Why

An agent wallet can enforce its owner's budget: how much *this* agent may spend. It cannot see what happens across other agents, sellers and rails. A settlement layer such as [Misa](https://trymisa.com) holds the payments, so it is the one place that sees all of them at once.

This engine is the clearing-house view, not the broker's:

| Layer | Question it answers | Owner |
| --- | --- | --- |
| Agent wallet | "May my agent make this payment?" (owner budgets, allowlists) | The agent's owner |
| **This engine** | "Is it safe to hold and settle this payment?" (open exposure per seller and verifier, cross-agent velocity, expiring holds) | The settlement layer |

The two are complementary: the engine can also enforce buyer budgets for agents without a policy-aware wallet, but its main job is the risk a single wallet cannot see.

## Scope

**In:**
- Exposure: amount currently held per agent, seller and verifier, per asset
- Limits: per asset in native units, and in aggregate in a reference currency with conservative pricing and haircuts
- Velocity: spend and order count over sliding windows
- Expiry: holds past their deadline leave exposure and are queued for reclaim
- Deterministic replay: state rebuilt from the settlement event log

**Out:** custody, signing, and moving funds. The engine only answers allow or deny, with a reason.

## Model

- **Assets** are identified by network and asset, as in x402 (CAIP-2 networks: `xrpl:0`, `eip155:8453`, or `stripe` for card rails). Each asset has its own decimals.
- **Amounts** are integers in an asset's smallest unit, stored as 128-bit values so 18-decimal assets fit. Amounts of different assets are never added directly.
- **Events** are rail-agnostic and follow the x402 `auth-capture` lifecycle: `Authorized`, `Captured`, `Reclaimed`, `Voided`, `Expired`. A rail adapter translates its own events (an XRPL `EscrowCreate`, an EVM authorization, a Stripe manual-capture PaymentIntent) into these.

## Shape

```
rail adapters (XRPL, EVM, Stripe) -> events -> risk core (C++) -> allow / deny
                                                    ^
                     x402 hooks (TypeScript) -------+
                     onBeforePaymentCreation, onBeforeVerify, onBeforeSettle
```

- **Core:** a C++20 library with no dependency on any rail or protocol, exposed later as a small service.
- **Adapters:** thin TypeScript glue that calls the core from x402 hooks and from Misa's settlement engine.

## Roadmap

1. Core library: exposure map, limits, expiry queue, unit tests
2. Event replay and benchmarks (latency per check, p50/p99)
3. Service interface, and a call from Misa before each hold is created
4. x402 hook adapters

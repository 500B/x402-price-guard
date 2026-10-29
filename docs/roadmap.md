# Implementation roadmap

Goal: a realistic, multi-party x402 environment in which any payment guard can be installed into one slot and verified against honest and adversarial behavior, with ground truth taken from the chain. Version 1 contains no guard logic.

## Layers

| Layer | Package | What it is |
|---|---|---|
| 0 | workspace | pnpm workspace, shared types and schemas, config, wallet derivation, tooling |
| 1 | chain | Anvil fork of Base Sepolia with real USDC, Permit2 and the upto proxy; funding, time and mining control, snapshot/revert, chain watcher, balance ledger |
| 2 | facilitator | Real `/supported`, `/verify`, `/settle` built on the official facilitator classes, with lifecycle hooks and a fault-injection endpoint |
| 3 | discovery | Bazaar catalog served by the facilitator, with seeding and poisoning |
| 4 | sellers | Honest seller on the official middleware; adversarial seller driven by behavior profiles; optional reverse proxy |
| 5 | buyer | Unmodified SDK client, instrumented signer, guard slot |
| 6 | agent | Scripted brain; LLM brain adapter |
| 7 | evidence | Event bus, ledger reconciliation, metrics |
| 8 | runner | Scenario schema, orchestration, assertions, differential reports, CLI |
| 9 | tiers | T0 mocks, T1 local fork (default), T2 testnet smoke, T3 containers |

## Phases

| Phase | Layers | Deliverables | Exit criteria |
|---|---|---|---|
| 1. Chain and ledger | 0, 1 | Workspace scaffold; fork boot with pinned block; wallet derivation and funding; time and mining control; snapshot/revert; chain watcher; balance snapshots | A raw EIP-3009 transfer built by the lab settles on the fork; snapshot/revert restores balances exactly; the watcher reports the transfer with its nonce |
| 2. Facilitator and discovery | 2, 3 | Facilitator service with hooks and fault control; discovery endpoints and seeding | `/supported` advertises `exact` and `upto` on the fork; a minimal honest seller round trip settles; each fault mode is observable in events |
| 3. Sellers | 4 | Honest seller with static, dynamic, upto, upfront and free routes and the discovery, payment-identifier and offer-receipt extensions; adversarial seller with the full profile catalog; reverse proxy | Every paid route returns 402 when unpaid; each profile produces its intended ledger delta against an unguarded buyer |
| 4. Buyer, scripted brain, bus | 5, 6 (scripted), 7 (bus), 9 (T0) | Agent runner; instrumented signer; guard slot; client hooks on the bus; scripted plans; T0 mocks | One honest call yields exactly one signature, one settlement and one body; the recorded typed data matches what the facilitator verified |
| 5. Runner, reconciliation, reports | 7, 8 | Scenario schema; orchestration lifecycle; reconciliation rules; metrics; differential report; CLI; honest, price, binding, protocol-edge, infrastructure, discovery and concurrency packs with recorded baselines | Honest pack passes with an empty slot; attack baselines equal the injected deltas; two runs with one seed produce identical evidence apart from timestamps and hashes |
| 6. Realism extensions | 6 (LLM), 9 (T2, T3) | LLM brain adapter and prompt-injection pack; MCP seller and client; Docker Compose with Toxiproxy; testnet smoke | Injection pack runs end to end; the same scenarios pass in T3; testnet smoke matches local results |
| 7. Deferred | — | Offline contract-deployment tier; Solana tier; Python and Go sellers | — |

## How a guard is verified once the lab exists

1. Run every pack with the empty slot; the report records the loss each attack causes against the unmodified SDK.
2. Run the same packs with the guard installed in the slot.
3. Pass criteria: zero loss and zero unexpected signatures on adversarial packs; zero false denials on the honest pack; every refusal explained by a guard decision event.

## Status

Design complete. Implementation not started. Rendered architecture and transaction-flow diagrams: https://claude.ai/artifact/67QzhgBtpyU3kezkqPJCEb

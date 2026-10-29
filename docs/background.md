# Theoretical background

This lab exists to verify payment guards for AI agents that pay over x402. The theory below is the minimum needed to understand why the lab is built the way it is.

## 1. The protocol in one paragraph

x402 revives HTTP 402. A resource server answers an unpaid request with `402` and a `PAYMENT-REQUIRED` header: a base64 JSON object naming the resource and listing acceptable payments (`accepts[]`, each with `scheme`, `network` in CAIP-2 form, `amount` in atomic token units, `asset`, `payTo`, `maxTimeoutSeconds`). The buyer signs one of them and retries with `PAYMENT-SIGNATURE`. The server asks a facilitator to `/verify` the signature, serves the resource, asks the facilitator to `/settle` on chain, and returns `PAYMENT-RESPONSE` with the settlement result. For the `exact` scheme on EVM chains the signature is an EIP-3009 `transferWithAuthorization(from, to, value, validAfter, validBefore, nonce)`; `upto` signs a ceiling through Permit2 and the server settles any amount up to it; `batch-settlement` deposits escrow and signs cumulative vouchers. Settlement happens inside the HTTP round trip.

## 2. The trust model and its gap

Five parties take part: the buyer (agent, SDK, wallet), the seller (resource server and its middleware), the facilitator, the chain, and discovery. The 402 is produced entirely by the seller and is unauthenticated. The official SDK parses it as raw JSON and signs the quoted `amount` for the quoted `payTo`; its only built-in control is a per-payment USD ceiling (default `$1`), which is a cap, not a price expectation. The facilitator checks that the signature matches the seller's requirements, not the buyer's intent. A produced signature is a bearer authorization: after it leaves the buyer, the outcome is decided by the seller, the facilitator and the chain. This is the gap a price guard closes, and it was demonstrated against the current SDK in x402-foundation/x402 issue #3866 (October 2026).

## 3. Known attack classes

- From issue #3866: price inflation on a known resource, gradual drift that defeats "N× last price" heuristics, recipient substitution, concurrent first calls racing past an empty price memory, and decimal or numeric amounts that slip past integer checks.
- From "Five Attacks on x402" (arXiv 2605.11781): grant-before-finality with later reversal, settlement preemption by anyone who sees the signed header, replay across the HTTP–chain boundary, proxy and cache confusion, and server-selection attacks through discovery.
- From "Free-Riding the Agentic Web" (arXiv 2605.30998): five invariants a sound deployment must hold — payment integrity, value consistency, context binding, authorization uniqueness, execution conservation — and four flaw classes: cross-resource substitution, duplicate-settlement races, `upto` allowance overdraft, and denial of settlement.

## 4. What follows for a test setup

1. **The signature is the point of no return.** The primary metric of any run is the number of signature requests the buyer's signer received, and the guard checkpoint that matters is the one that runs before signing.
2. **The chain is the only ground truth.** `Transfer` and `AuthorizationUsed` logs and wallet balances decide what happened; facilitator responses, `PAYMENT-RESPONSE` headers and server logs are claims to reconcile against them.
3. **Every claim must be correlatable.** A lab request id links buyer and seller events; the EIP-3009 nonce links the signature, the facilitator's verification and the chain log; the transaction hash links settlement to the receipt.
4. **Realism is not optional.** The attacks exploit properties of the real pipeline (raw parsing, hook order, object identity, facilitator behavior, token semantics), so the lab runs the unmodified SDK, the real middleware, real facilitator code and the real token contracts on a forked chain that the lab controls (time, mining, reorgs).
5. **Verification is differential and fail-closed.** Every scenario runs once with an empty guard slot to measure the loss, then with the guard; a guard passes when adversarial runs show no loss and no unexpected signature, and honest runs show no false denial. "Not observed" is never "proven failed".

## References

- x402 specification v2 and scheme specs: https://github.com/x402-foundation/x402/tree/main/specs
- x402 documentation: https://docs.x402.org
- Issue #3866, overcharge against the SDK client: https://github.com/x402-foundation/x402/issues/3866
- Li, Wang, Wang, "Five Attacks on x402 Agentic Payment Protocol": https://arxiv.org/abs/2605.11781
- Ling et al., "Free-Riding the Agentic Web: A Systematic Security Analysis of x402 Payments": https://arxiv.org/abs/2605.30998
- EIP-3009, Transfer With Authorization: https://eips.ethereum.org/EIPS/eip-3009

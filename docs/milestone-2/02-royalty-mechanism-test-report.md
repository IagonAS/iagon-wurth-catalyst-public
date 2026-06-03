# Milestone 2 – Part 2: Royalty Mechanism Test Report

## 1 Purpose and Scope

### 1.1 Purpose of this report

This document is one of the four output reports for Milestone 2 of the Iagon & Würth Catalyst Fund 13 project. It describes the royalty mechanism that the orchestrator implements off-chain and how that mechanism interacts with the on-chain primitives documented in Part 1 (Smart Contract Prototype Report).

In particular, this report addresses the Milestone 2 *Royalty Mechanism* output and its associated acceptance criteria, which call for:

- the **automated royalty calculation and payout process** to be implemented in code and tested,
- the logic to be **validated against the business rules** agreed with Würth,
- the calculation to be exercised with worked examples and samples or logs that reviewers can inspect.

The report is intentionally written at the same level of abstraction as the Milestone 1 deliverables — sufficient to validate scope, behaviour, and intent without disclosing internal implementation details that fall under the closed-source enterprise pilot agreement.

### 1.2 Scope boundaries

In scope for this document:

- the per-order **fee split** between vendor royalties, the maintainer fee, and the operator fee;
- the **tiered maintainer fee** structure and its calculation rule, including current tier values;
- the **fund-lock** mechanism that runs from order acceptance to the first on-chain royalty distribution;
- the **batching** strategies that aggregate eligible royalties into on-chain transactions, including the testing and production strategies and our choice for Milestone 2;
- **payout** semantics from a business perspective — what a payout does, whether partial payouts are supported, and what downstream processing options exist for the released stablecoin;
- the **edge cases and validation rules** that protect the system from inconsistent or unfair outcomes.

Out of scope for this document:

- the on-chain primitives and their datums (covered in Part 1),
- the orchestrator's HTTP API surface (covered in Part 4),
- detailed implementation specifics such as table schemas, service-class internals, and validator code paths (covered in the closed-source repository),
- production-environment operational concerns such as monitoring thresholds and runbook procedures.

### 1.3 Relationship to other Milestone 2 deliverables

- **Part 1 — Smart Contract Prototype Report** describes the on-chain primitives (IOU mint, vault use, bucket withdrawal) that this royalty mechanism ultimately drives.
- **Part 3 — Off-chain API Prototype Report** describes the services and the ORSY integration that exercise the mechanism described here end-to-end.
- **Part 4 — API Endpoint Specification** documents the orchestrator HTTP endpoints through which orders, settlement state, and payouts are observed and controlled.

## 2 Royalty Calculation

### 2.1 Fee split

Every order accepted by the orchestrator is split three ways between the parties that have an economic claim against it:

- **Vendor royalties** — fixed, per-unit amounts owed to the intellectual-property owner for each licensed item. Computed deterministically from the licence's configured royalty value and the order line item's quantity.
- **Maintainer fee** — a percentage charged for operating the royalty platform itself. Tiered by the per-unit royalty value of each line item, summed across the order.
- **Operator fee** — the remainder of the order value after the vendor royalties and the maintainer fee have been deducted. The operator (Würth) is the residual claimant on each order.

In short:

```
Total Order Value = Vendor Royalties + Maintainer Fee + Operator Fee
```

The first two terms are computed deterministically from the order's line items and the configured fee tiers; the operator fee is whatever is left.

### 2.2 Maintainer fee tiers

The maintainer fee is a percentage of the vendor royalty itself. It is tiered by the per-unit royalty value of each line item, with a higher-value royalty attracting a slightly higher percentage. The tier table at the time of this report is shown below; tier rows are stored in a configuration table and seeded during database migration so that they can be adjusted without code changes.

| Royalty per unit                  | Fee percentage |
| --------------------------------- | -------------- |
| Below $1.00                       | 3.00 %         |
| $1.00 to below $2.00              | 3.30 %         |
| $2.00 to below $3.00              | 3.40 %         |
| $3.00 to below $4.00              | 3.60 %         |
| $4.00 to below $5.00              | 3.70 %         |
| $5.00 to below $6.00              | 3.80 %         |
| $6.00 and above                   | 4.00 %         |

Amounts are denominated in micro-units (millionths) of the underlying stablecoin to support sub-cent precision; the rationale is given in Section 2.3. Tier selection finds the lowest-numbered tier whose upper bound is greater than or equal to the line item's per-unit royalty.

The tier values shown above are **contractually defined between Iagon and Würth**. They may be adjusted before production launch as the commercial terms of the platform are finalised, and may be changed thereafter by joint agreement between both parties — for example to revise percentages in response to operating-cost data, to introduce additional tiers, or to add tier overrides for specific vendor classes.

Crucially, the tier table is **not part of the smart contract**. It is held in an orchestrator-side configuration table that is seeded during database migration and adjusted operationally as the agreement requires. At settlement time, the orchestrator computes the maintainer fee from the active tier table and instructs the chain to mint the resulting IOU amount; the on-chain validators are agnostic to how that amount was derived and check only that the value being minted is consistent. This keeps the on-chain footprint minimal and makes future tier adjustments a configuration change rather than a contract migration with its own audit cycle.

### 2.3 Per-item calculation

For each line item in an order:

```
itemRoyalty = royaltyAmountMicro × quantity
itemMaintainerFee = floor(royaltyAmountMicro × quantity × tierFeePercent ÷ 100)
```

**Worked example.** A line item whose licence is configured with a per-unit royalty of $2.50 (2,500,000 micro), ordered with a quantity of 2:

- `itemRoyalty = 2,500,000 × 2 = 5,000,000 micro` — $5.00 owed to the vendor.
- The $2.50 per-unit royalty falls in the *$2.00 to below $3.00* tier at 3.40 %.
- `itemMaintainerFee = floor(2,500,000 × 2 × 3.40 ÷ 100) = floor(170,000) = 170,000 micro` — $0.17, which is 3.40 % of the $5.00 vendor royalty.

The order-level totals are the sums of the per-item values across all line items. The order-level operator fee is then:

```
operatorFee = totalOrderValue − totalVendorRoyalties − totalMaintainerFee
```

`floor()` is applied to every monetary computation so that no fractional micro-unit is ever produced. This guarantees deterministic, audit-friendly amounts at every stage of settlement and avoids the rounding drift that would otherwise accumulate over many orders.

The use of integer arithmetic with `floor()` truncation is exactly why the platform denominates monetary values in **micro-units** rather than cents or whole currency units. Milestone 1 established that the platform must support **sub-cent royalty precision** so that low-cost parts can carry economically meaningful royalties without being rounded down to zero. By representing all monetary values as integers measured in millionths of a currency unit — six decimal places of precision — the orchestrator and the on-chain primitives can compute, accumulate, and settle royalty amounts down to a single micro-unit without ever invoking floating-point arithmetic, and without losing precision at any step of the calculation. Six decimal places of integer precision is also the canonical representation used by most Cardano-native stablecoins, so the orchestrator's internal accounting is in lock-step with the on-chain assets it ultimately moves.

### 2.4 When the calculation is applied

Fees are calculated at two distinct points in the order lifecycle:

1. **At order submission**, as part of the response returned to the caller. This calculation is informational and is also used to enforce the validation rules in Section 2.5. It is not authoritative for settlement.
2. **At settlement time**, when the orchestrator instructs the chain to mint IOU tokens to the operator and maintainer system buckets. This calculation is the authoritative on-chain value. Recalculating at this point means that any subsequent change to a tier table value before settlement is reflected in the settlement, while past settlements remain untouched.

### 2.5 Validation rules

The orchestrator enforces a small set of validation rules at order submission, divided into *hard* rules — orders failing them are rejected outright — and one *soft* rule that logs a warning while still letting the order proceed.

Hard validation (order rejected):

- the total order amount must be strictly positive,
- each referenced licence must exist on the orchestrator side and be marked active,
- each line-item quantity must be strictly positive,
- the sum of total vendor royalties and total maintainer fee must not exceed the total order value, so that the operator fee cannot go negative.

Soft validation (warning logged, order proceeds):

- if the combined vendor royalties and maintainer fee exceed a configurable share of the total order value (default 30 %), a warning is recorded for operational review.

### 2.6 Edge cases and borderline conditions

The mechanism is exercised by a unit-test suite that covers each tier in isolation, multi-tier orders, the boundary between adjacent tiers, and degenerate inputs. The following cases are explicitly tested and form the basis of the worked examples cited in Section 6.

**Tier coverage**

- A line item with a per-unit royalty of $0.50 falls in the *below $1.00* tier and is charged 3.00 %. For one unit, the fee floors to 15,000 micro.
- A line item with a per-unit royalty of $1.50 falls in the *$1.00 to below $2.00* tier and is charged 3.30 %. For two units, the fee floors to 99,000 micro.
- A line item with a per-unit royalty of $8.00 falls in the *$6.00 and above* tier and is charged 4.00 %. For one unit, the fee floors to 320,000 micro.

**Multi-tier orders**

- An order combining $0.50 × 1 ($0.50 royalty), $3.50 × 2 ($7.00 royalty), and $8.00 × 1 ($8.00 royalty) produces three independent tier lookups whose fees sum to the total maintainer fee. Tiers are evaluated per line item and never blended.

**Tier boundary**

- The maximum value of a tier is *inclusive*. A line item with a per-unit royalty of exactly $0.999999 (one micro below the next-tier threshold) still falls in the *below $1.00* tier at 3.00 %.
- A line item one micro higher, at exactly $1.00, switches into the *$1.00 to below $2.00* tier at 3.30 %. The pair confirms the transition happens at exactly the configured threshold.

**Floor rounding**

- Per-item fee calculations that produce a fractional micro-unit are floored, never rounded up. A $0.777777 × 1 line item at 3.00 % yields a fee of exactly 23,333 micro (truncated from 23,333.31).

**Degenerate inputs**

- An order with a zero per-unit royalty produces a zero maintainer fee, regardless of quantity.
- An order with no line items produces a zero maintainer fee.

**Operator-fee-negative orders**

- The fee structure document gives an explicit borderline case. An order with $20.00 total value that already commits $20.00 in vendor royalties (for example: licence A at $0.50 × 10 plus licence B at $3.00 × 5) leaves no room for the maintainer fee of $0.69 — the operator fee would go negative — and is therefore rejected at validation.
- The same order is accepted if the total order value is raised to $25.00, leaving an operator fee of $4.31 after $20.00 in vendor royalties and $0.69 in maintainer fees.

These cases collectively demonstrate that the royalty calculation is **deterministic** (same inputs always produce the same outputs), **conservative under rounding** (truncation rather than rounding up), and **safe under adversarial inputs** (overpriced royalties or zero-value orders cannot put the operator into a negative position).

## 3 Fund Locks

### 3.1 Purpose of the lock window

An order accepted by the orchestrator does not immediately trigger an on-chain royalty distribution. Instead the order enters a **fund-lock window** during which the order is committed to the orchestrator's database and is visible in the dashboards, but its royalty value is not yet eligible for on-chain settlement. The lock window has three purposes:

- it accommodates **order cancellations** initiated via ORSY by the customer, or by a Würth administrator within a documented grace period without triggering wasted on-chain activity;
- it gives the operator a single, well-defined moment at which the royalty becomes economically final;
- it makes downstream settlement **batchable** by definition — eligible orders are always those whose lock has expired, which gives the batching strategies a clean queue to work from.

The lock window is configurable via an environment variable (`ORDER_LOCK_DURATION_MS`, expressed in milliseconds) and defaults to 24 hours (`86_400_000` ms) for development environments. The Milestone 2 demonstration uses `30000` (30 s) so that lock expiry is observable live in the walkthrough video. The production value will be agreed jointly with Würth before mainnet rollout.

### 3.2 Order lifecycle

Every order moves through the following statuses during its lifetime in the orchestrator:

| Status             | Meaning                                                                                                  |
| ------------------ | -------------------------------------------------------------------------------------------------------- |
| PENDING            | Initial status. The order is locked. Royalties are computed but not yet eligible for on-chain settlement. |
| BATCHED            | All of the order's line items have been assigned to one or more settlement batches.                       |
| AWAITING_SETTLEMENT | All vendor royalty batches have been confirmed on-chain. The order is waiting for the operator and maintainer fee mint. |
| PROCESSED          | The settlement transaction has been confirmed. The order is fully complete.                              |
| CANCELLED          | The order was cancelled before processing began.                                                         |

A cancellation during PENDING is a no-op on the chain: the lock simply expires unobserved and the order moves to CANCELLED. A cancellation after BATCHED is not currently supported and is rejected by the orchestrator with an explanatory error.

### 3.3 What "locking funds" means in practice

The lock at this milestone is **logical**, not custodial: the orchestrator does not move money or reserve any on-chain asset during the lock window. The lock is expressed as a `locked_until` timestamp on the order and a status of PENDING; the batching subsystem filters those orders out of its queue. This makes the lock cheap, reversible, and observably correct, at the cost of relying on the off-chain database for short-term integrity. The economic effect is the same as a custodial reservation: no royalty token is minted and no stablecoin moves until the lock window has passed.

### 3.4 Calibrating the window over time

The length of the lock window is an operational lever, calibrated to throughput. In the initial rollout we expect lower order volume, and a longer window gives each batch enough orders for the on-chain cost of settling it to be worth paying. As order volume grows and the system's day-to-day correctness has been established in production, the window can be shortened: batches will still reach a reasonable size in less elapsed time, and the gap between an order being placed and its royalty becoming visibly settled on chain — the time to on-chain transparency — drops accordingly.

## 4 Batching Strategies

### 4.1 Why batching exists

On Cardano, every transaction incurs a fee that depends on the size of the transaction and the execution units consumed by any scripts it invokes. Submitting one transaction per order per vendor/bucket or per line item is the simplest possible model but is expensive at scale. Batching multiple orders' royalty distributions into a single transaction amortises the fixed cost across many entries, but introduces complexity around how to group, when to send, and how to recover from partial failures.

The orchestrator therefore exposes batching as a swappable **strategy** behind a single interface. The active strategy is selected at startup via an environment variable, and new strategies can be added without changing the orchestrator's order, batch-processing, or settlement code. This is the same pattern used elsewhere in the platform for swappable backends; for batching it lets us run a low-latency strategy in development and a cost-efficient strategy in production without forking either code path.

### 4.2 Strategies shipped at Milestone 2

Two strategies are implemented at this milestone:

**Every order, by vendor.** Each order is batched immediately on becoming eligible, with one batch per vendor per order — no accumulation window. Each batch is sealed to *ready* status as soon as it is created. This strategy produces the highest number of on-chain transactions and the lowest vendor latency. It is the **testing / development strategy** and is the orchestrator's current default for the chain-simulator-backed walkthrough demonstrations: it makes test runs predictable, each tx easy to trace to a specific order, and exposes any failure on the first attempt rather than after a window has elapsed.

**Time-window batch.** All eligible items are accumulated into a single open batch regardless of vendor or order. The batch is sealed to *ready* when either the time window expires (default 24 hours) or the batch reaches a configurable maximum item count (default 20, chosen to stay well within Cardano's transaction size limit). Vendor royalties from multiple vendors and multiple orders are then released in a single on-chain transaction. This is the **production strategy** and the one we expect to operate under in mainnet conditions: it produces a lower tx volume and a high tx utilisation, at the cost of up to one window's worth of vendor-visible latency.

A condensed comparison:

| Strategy             | Tx volume | Vendor latency      | Tx utilisation | Operational complexity |
| -------------------- | --------- | ------------------- | -------------- | ---------------------- |
| Every-order-by-vendor | Highest   | Lowest (immediate)  | Low            | Trivial                |
| Time-window batch     | Low       | Up to one window    | High           | Low                    |

### 4.3 What we ship with for Milestone 2

The Milestone 2 evidence package uses **every-order-by-vendor** for the recorded walkthrough and for the deterministic test outputs in Section 6 because that strategy makes each on-chain transaction map one-to-one to a specific order. Reviewers can therefore follow the same order through the orchestrator user interface, the chain-simulator, and the relevant on-chain artifacts without having to disentangle multi-order batches.

The **time-window** strategy is fully implemented, tested in isolation, and switchable at startup via the active-strategy environment variable. It is intended to become the default ahead of the production rollout planned for Milestone 5, after a separately documented load and cost-modelling exercise has chosen the appropriate window length and maximum batch size for Würth's expected order volumes.

### 4.4 Safety guarantees

Both strategies share three guarantees that are enforced by the surrounding services rather than by the strategy itself:

- **Lock-awareness.** Only orders whose lock has expired are visible to the batching strategy. An order in PENDING is never seen as eligible.
- **No double-batching.** Each line item can appear in at most one non-failed batch. This is enforced both by a database-level anti-join and by the order-status gate (`BATCHED` orders are excluded from future cycles).
- **Royalty snapshot.** The royalty value used at settlement is the value as it was *at batching time*. A subsequent change to a licence's royalty does not retroactively alter already-batched items. This gives downstream reconciliation a stable source of truth.

## 5 Payouts

### 5.1 What a payout is, from a business perspective

A *payout* is the moment when accumulated royalty credits in a vendor's (or system) bucket are converted into stablecoin and released to a Cardano address. Up to that point, royalty credits accumulate as IOU tokens inside the participant's bucket. The payout transaction burns those IOU tokens and simultaneously spends a vault UTxO to release an equal amount of stablecoin to the receiving address. The on-chain mechanics are documented in Part 1; this section focuses on what that means for the parties involved.

A payout in the orchestrator is initiated by an explicit action — there is no automatic payout schedule at this milestone. Two payout actions exist:

- A **vendor payout** drains an individual vendor's bucket. Triggered from the vendor detail view of the orchestrator user interface.
- An **operator payout** drains the operator's system bucket. Triggered from the operator dashboard.

The maintainer's accumulated balance is held in the maintainer system bucket and is paid out through the same mechanism, but not through the current demo dashboard.

### 5.2 Partial payouts

**Partial payouts are not possible at the contract level.** The bucket withdrawal validator deliberately requires that the burn amount equals the *full* IOU balance of the bucket being withdrawn from, in the same transaction that releases the matching stablecoin from the vault. There is no validator path that accepts a partial burn or a partial vault release. This is a property of the on-chain design, not a limitation of the orchestrator: even a hand-crafted transaction submitted directly to the chain would be rejected by the script if it tried to leave any IOU balance behind.

This is deliberate. A whole-bucket withdrawal is the simplest possible invariant to audit — the bucket's IOU balance flips to zero in lock-step with a vault release that exactly matches the burned amount. Allowing partial drains would introduce additional state to validate (residual balance, identity of the residual), additional rounding and accounting edge cases, and additional validator paths that an external auditor would have to reason about independently. The conservative choice for the initial design was to make withdrawals atomic and total.

As a consequence, the orchestrator's payout endpoints (vendor and operator) build transactions that drain the full IOU balance of the selected bucket. There is no "withdraw $X of $Y" surface at any layer of the stack at Milestone 2.

### 5.3 Downstream processing options

Where the stablecoin lands after a payout is the **bucket keepers' choice**, not the orchestrator's. The orchestrator's payout flow builds a transaction whose output goes to an address controlled by the bucket's keepers; from there, downstream processing depends entirely on how those keepers are organised. We expect at least three patterns to be in use across the lifecycle of the platform:

- **Self-custody on Cardano.** The bucket keepers hold their own signing keys and the stablecoin lands at a vendor- or operator-controlled wallet directly on chain. This is the lightest-weight option and the default model used during the Milestone 2 demonstrations, and likely executed by the maintainer (Iagon).
- **Custodial management.** The bucket keepers are an enterprise custody service (for example, Fireblocks or a comparable solution operated on Würth's or a vendor's behalf), and the stablecoin lands in an address managed by that custodian. The payout transaction itself is unchanged; only the address it pays to belongs to the custodian. A Fireblocks-style admin signing experience is already prototyped in the project repository.
- **Automated off-chain settlement.** A downstream service operated by or on behalf of the bucket's keepers can drive payouts on a schedule, on a balance threshold, or on any other rule the vendor or operator chooses to encode. When the trigger fires, the service initiates the bucket-emptying transaction described in §5.1 and selects the destination address for the released stablecoin — either preconfigured in the automation's own setup, or chosen dynamically per cycle (for example, a fresh address each time for accounting or compliance reasons). The destination is not a property of the bucket itself; the bucket simply releases its full IOU balance to whichever output the transaction is built with, and the automation is the party deciding what that output points to. Once the stablecoin lands at that address, further off-chain steps can take over — a bridge from stablecoin to fiat, a bank-wire workflow, or an internal accounting reconciliation against a vendor's general ledger.

Because the choice of downstream is encoded entirely in the bucket's keeper configuration, individual vendors could adopt different models without changing the orchestrator. A single deployment can mix self-custody and custodial vendors freely.

## 6 Test Outputs and Demonstration Evidence

This section anchors the report against the worked examples and run logs that reviewers can inspect.

### 6.1 Unit test transcripts

The royalty calculation is exercised by three unit-test suites, all running under `vitest` against in-memory implementations of their dependencies so that the calculation logic is verified in isolation from the database and the surrounding services. All cases pass at the time of writing.

- **`orderService.spec.ts`** — verifies that vendor royalty totals are summed correctly across line items, including duplicate line items for the same licence, and that the resulting `totalRoyaltiesMicro` is fed into the fee calculation step.
- **`feeCalculationService.spec.ts`** — verifies the tiered maintainer-fee calculation, the operator-fee-as-remainder rule, and the edge cases described in Section 2.6.
- **`batchProcessingService.spec.ts`** — verifies that when a batch contains line items from multiple vendors, the resulting on-chain mint preserves each vendor's royalty total separately and routes it to that vendor's own bucket.

Together the three suites cover the three stages of the calculation: the order service computes the vendor royalty total from line items; the fee service applies the tier table to derive the maintainer and operator shares; and the batch processing service splits the vendor royalty back out per vendor at settlement time so that each vendor's bucket receives only its own share.

#### Royalty summing — `orderService.spec.ts`

These tests use a mock licence repository (`createMockLicense`) and a mock fee calculation service so that the orchestrator's own royalty-summing logic can be asserted independently of the tier table.

```typescript
test('totalRoyaltiesMicro includes all duplicate occurrences', async () => {
  const deps = createMockDeps()
  const license = createMockLicense({
    id: 1,
    partNumber: 'WIDGET',
    royaltyAmountMicro: 1000000,
  })
  deps.licenses.set('WIDGET', license)

  const result = await createOrderService(deps).submitOrder({
    items: [
      { partNumber: 'WIDGET', quantity: 10 },
      { partNumber: 'WIDGET', quantity: 5 },
    ],
    totalOrderAmountMicro: 50000000,
  })

  // royalty = 1000000 * (10 + 5) = 15000000
  expect(result.feeBreakdown.totalRoyaltiesMicro).toBe(15000000)
})
```

```typescript
test('submitOrder returns fees from feeCalculationService', async () => {
  const deps = createMockDeps()
  const license = createMockLicense({
    id: 1,
    partNumber: 'WIDGET',
    royaltyAmountMicro: 1000000,
  })
  deps.licenses.set('WIDGET', license)

  // Make feeCalculationService return specific fees
  deps.feeCalculationService.calculateSettlementFees = async ({
    totalOrderValueMicro,
    totalRoyaltiesMicro,
  }) => {
    return {
      maintainerFeeMicro: 500000,
      operatorFeeMicro: totalOrderValueMicro - totalRoyaltiesMicro - 500000,
    }
  }

  const result = await createOrderService(deps).submitOrder({
    items: [{ partNumber: 'WIDGET', quantity: 2 }],
    totalOrderAmountMicro: 10000000,
  })

  expect(result.feeBreakdown.totalRoyaltiesMicro).toBe(2000000)
  expect(result.feeBreakdown.maintainerFeeMicro).toBe(500000)
  expect(result.feeBreakdown.operatorFeeMicro).toBe(7500000)
})
```

#### Maintainer-fee tiers — `feeCalculationService.spec.ts`

The test fixture seeds the same tier table shown in Section 2.2:

```typescript
const DEFAULT_TIERS: FeeTier[] = [
  { id: 1, maxRoyaltyMicro: 999999, feePercent: 3.0 },
  { id: 2, maxRoyaltyMicro: 1999999, feePercent: 3.3 },
  { id: 3, maxRoyaltyMicro: 2999999, feePercent: 3.4 },
  { id: 4, maxRoyaltyMicro: 3999999, feePercent: 3.6 },
  { id: 5, maxRoyaltyMicro: 4999999, feePercent: 3.7 },
  { id: 6, maxRoyaltyMicro: 5999999, feePercent: 3.8 },
  { id: 7, maxRoyaltyMicro: Number.MAX_SAFE_INTEGER, feePercent: 4.0 },
]
```

#### Tier coverage — `calculateMaintainerFee`

```typescript
test('applies 3.00% tier for sub-dollar royalty amounts', async () => {
  // 500000 micro = $0.50, falls in < $1 tier (3.00%)
  const fee = await service.calculateMaintainerFee([
    { royaltyAmountMicro: 500000, quantity: 1 },
  ])

  // floor(500000 * 1 * 3.00 / 100) = floor(15000) = 15000
  expect(fee).toBe(15000)
})
```

```typescript
test('applies 3.30% tier for $1-$1.99 royalty amounts', async () => {
  // 1500000 micro = $1.50, falls in $1-$1.99 tier (3.30%)
  const fee = await service.calculateMaintainerFee([
    { royaltyAmountMicro: 1500000, quantity: 2 },
  ])

  // floor(1500000 * 2 * 3.30 / 100) = 99000
  expect(fee).toBe(99000)
})
```

```typescript
test('applies 4% tier for $6+ royalty amounts', async () => {
  // 8000000 micro = $8.00, falls in $6+ tier (4.00%)
  const fee = await service.calculateMaintainerFee([
    { royaltyAmountMicro: 8000000, quantity: 1 },
  ])

  // floor(8000000 * 1 * 4 / 100) = 320000
  expect(fee).toBe(320000)
})
```

#### Multi-tier order

```typescript
test('sums fees across multiple items with different tiers', async () => {
  const fee = await service.calculateMaintainerFee([
    { royaltyAmountMicro: 500000, quantity: 1 },  // < $1 tier 3.00% → floor(15000) = 15000
    { royaltyAmountMicro: 3500000, quantity: 2 }, // $3-$3.99 tier 3.60% → floor(252000) = 252000
    { royaltyAmountMicro: 8000000, quantity: 1 }, // $6+ tier 4.00% → floor(320000) = 320000
  ])

  expect(fee).toBe(15000 + 252000 + 320000)
})
```

#### Tier boundary

These two cases bracket the transition between tier 1 (3.00 %) and tier 2 (3.30 %) from both sides: $0.999999 stays in tier 1, and $1.000000 — one micro above — switches into tier 2. Together they confirm the switch happens at exactly the configured threshold.

```typescript
test('uses boundary value correctly (exact tier max)', async () => {
  // 999999 micro is the max of tier 1 (3.00%) — should match that tier
  const fee = await service.calculateMaintainerFee([
    { royaltyAmountMicro: 999999, quantity: 1 },
  ])

  // floor(999999 * 1 * 3.00 / 100) = floor(29999.97) = 29999
  expect(fee).toBe(29999)
})
```

```typescript
test('uses boundary value correctly (exact tier min)', async () => {
  // 1000000 micro = $1.00, one micro above tier 1's upper bound — should
  // switch into the $1.00 to below $2.00 tier (3.30%)
  const fee = await service.calculateMaintainerFee([
    { royaltyAmountMicro: 1000000, quantity: 1 },
  ])

  // floor(1000000 * 1 * 3.30 / 100) = floor(33000) = 33000
  expect(fee).toBe(33000)
})
```

#### Floor rounding

```typescript
test('floors fractional fee amounts', async () => {
  // 777777 * 1 * 3.00 / 100 = 23333.31 → floor to 23333
  const fee = await service.calculateMaintainerFee([
    { royaltyAmountMicro: 777777, quantity: 1 },
  ])

  expect(fee).toBe(23333)
})
```

#### Degenerate inputs

```typescript
test('returns 0 for zero royalty amount', async () => {
  const fee = await service.calculateMaintainerFee([
    { royaltyAmountMicro: 0, quantity: 5 },
  ])

  expect(fee).toBe(0)
})

test('returns 0 for empty items', async () => {
  const fee = await service.calculateMaintainerFee([])

  expect(fee).toBe(0)
})
```

#### Settlement fee composition — `calculateSettlementFees`

These two cases verify that the operator fee is the deterministic remainder of total order value minus vendor royalties minus the tiered maintainer fee.

```typescript
test('calculates maintainer and operator fees correctly', async () => {
  // 2500000 micro = $2.50 each, qty 2 → $2-$2.99 tier (3.40%)
  const result = await service.calculateSettlementFees({
    totalOrderValueMicro: 10000000,
    totalRoyaltiesMicro: 5000000,
    items: [{ royaltyAmountMicro: 2500000, quantity: 2 }],
  })

  // maintainer = floor(2500000 * 2 * 3.40 / 100) = floor(170000) = 170000
  // operator   = 10000000 - 5000000 - 170000 = 4830000
  expect(result.maintainerFeeMicro).toBe(170000)
  expect(result.operatorFeeMicro).toBe(4830000)
})
```

```typescript
test('operator fee is remainder after royalties and maintainer fee', async () => {
  // 500000 micro = $0.50, falls in < $1 tier (3.00%)
  const result = await service.calculateSettlementFees({
    totalOrderValueMicro: 2000000,
    totalRoyaltiesMicro: 500000,
    items: [{ royaltyAmountMicro: 500000, quantity: 1 }],
  })

  // maintainer = floor(500000 * 1 * 3.00 / 100) = floor(15000) = 15000
  // operator   = 2000000 - 500000 - 15000 = 1485000
  expect(result.maintainerFeeMicro).toBe(15000)
  expect(result.operatorFeeMicro).toBe(1485000)
})
```

#### Vendor-separation at settlement — `batchProcessingService.spec.ts`

When a batch contains line items from multiple vendors, the orchestrator must produce an on-chain mint that credits each vendor's *own* bucket with that vendor's *own* royalty total — never blending them and never sending a vendor's royalty to the wrong bucket. The test below feeds the batch processor a batch whose per-vendor royalty totals are $3.00 for vendor 10 and $5.00 for vendor 20, with each vendor mapped to a distinct on-chain bucket, and asserts that the resulting `fillAndSubmitBucket` call carries exactly those two amounts directed at exactly those two buckets:

```typescript
test('multi-vendor batch groups items by vendor into multiple fill orders', async () => {
  const batch = makeBatch({ id: 9, totalRoyaltyMicro: 8000000 })
  const vendorA = makeVendor({ id: 10, bucketTokenName: 'bucket_a' })
  const vendorB = makeVendor({ id: 20, bucketTokenName: 'bucket_b' })
  const filler = createMockBatchFiller()

  const batchRepo = createMockBatchRepo({
    findOldestReadyBatch: async () => batch,
    findVendorRoyaltiesForBatch: async () => [
      { vendorId: 10, totalRoyaltyMicro: 3000000 },
      { vendorId: 20, totalRoyaltyMicro: 5000000 },
    ],
  })

  const service = createBatchProcessingService({
    batchRepository: batchRepo,
    vendorRepository: createMockVendorRepo({
      findById: async (id: number) => {
        if (id === 10) return vendorA
        if (id === 20) return vendorB
        return null
      },
    }),
    batchFiller: filler,
    // tracked-transaction repo, timeouts, and re-attempt config elided
  })

  await service.processNextBatch()

  expect(filler.calls[0].orders).toEqual([
    { bucketId: 'bucket_a', increase: 3000000 },
    { bucketId: 'bucket_b', increase: 5000000 },
  ])
})
```

The two crucial assertions are encoded in that final `toEqual`: each vendor's bucket appears exactly once in the order list, and each `increase` value matches its vendor's royalty total. Cross-routing (vendor A's royalty into bucket B) or merging (a single combined `increase` of 8,000,000) would both fail this assertion.

### 6.2 End-to-end demonstration on the chain-simulator

The end-to-end exercise of the royalty mechanism described in this report is the **walkthrough video that accompanies Part 3 — Off-chain API Prototype Report**. The recording drives the orchestrator against the chain-simulator stack and shows, in order, vendor and licence registration, order submission, the fund-lock window expiring, IOU minting into the vendor and operator buckets, the vault state behind the payout, and the resulting vendor and operator payouts. Each stage in the video maps directly onto a section of this report:

| Stage in the walkthrough video | Corresponding section of this report |
|---|---|
| Order submission and the returned fee breakdown | §2 Royalty Calculation |
| Order sitting in PENDING with a future `locked_until` | §3 Fund Locks |
| Lock advance, batching job, and IOU mint into vendor + operator buckets | §4 Batching Strategies |
| Vendor payout (and operator payout) draining the buckets through the vault | §5 Payouts |

Refer to Part 3 for the step-by-step narration of the demonstration, the sequence diagrams that document each stage, and the access details for the video itself.


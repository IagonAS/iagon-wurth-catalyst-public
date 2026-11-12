# Milestone 4 – Refinement Plan: Conclusions and Next Steps

This is the refinement planning file for Milestone 4 (evidence 1.2). It draws conclusions from the tester feedback and the testnet results and sets out the next steps (success criterion 1.3).

## 1 Conclusions

- **The prototype works end to end on testnet.** 25 unique testers from Würth Canada, Würth IT India and Iagon ran the eight test scenarios. All 101 tests run passed, and the average ease rating was 4.96 out of 5 ([`01-user-portal-beta-report.md`](01-user-portal-beta-report.md)).
- **Royalties settle on chain as designed.** All 31 orders that were not cancelled settled on Cardano preprod 24 hours after they were placed, in 26 transactions, and a test order cancelled inside its window was never settled. A first payout redeemed 15.00 tUSD of IOU from the treasury. The dashboard matches the chain for every balance ([`04-smart-contract-test-logs.md`](04-smart-contract-test-logs.md)).
- **Würth's systems are integrated.** ORSY Connect sends orders to the orchestrator, and Würth has integrated DIS with Iagon storage through the storage connector ([`03-storage-connector-report.md`](03-storage-connector-report.md)).
- **Feedback was positive and the issues raised are closed.** Testers asked for no changes to the portal. The two issues raised were resolved during the test round (§2). In the review call, Würth reported no outstanding items for the dashboard.

## 2 Feedback incorporated

| Issue raised | Raised by | What was done | Status |
|---|---|---|---|
| I1: Explain how total royalties and the maintainer fee are calculated | Würth IT India tester (R01) | Iagon explained the rules by email, with a worked example from the tester's own rejected order, and again in a call. The rules are documented in the partner dashboard under API docs, Getting started guide, Fees. | Resolved; the tester confirmed |
| I2: Define what ORSY should validate before submitting an order, to avoid failures and 400 errors | Würth IT India tester (R01) | Iagon gave ORSY the acceptance rule below, which ORSY can check before sending an order, or ORSY can keep every royalty clearly below the selling price. | Resolved; the tester confirmed |

**The rules.** Amounts are in micro-units (1.00 = 1,000,000).

- Royalties *R* = the sum over the order lines of the licence's royalty per piece × quantity.
- Maintainer fee *M* = the sum over the order lines of royalty per piece × quantity × *p* / 100, rounded down, where *p* depends on the royalty per piece:

| Royalty per piece (tUSD) | Maintainer fee *p* |
|---|---|
| up to 0.999999 | 3.00% |
| 1.00 – 1.999999 | 3.30% |
| 2.00 – 2.999999 | 3.40% |
| 3.00 – 3.999999 | 3.60% |
| 4.00 – 4.999999 | 3.70% |
| 5.00 – 5.999999 | 3.80% |
| 6.00 and above | 4.00% |

- An order with total *T* is accepted when *R* + *M* ≤ *T*. The rest, *T* − *R* − *M*, is the operator fee.

All 32 orders in the test round follow these rules exactly, and the 31 that were not cancelled were settled on chain with these amounts ([`04-smart-contract-test-logs.md`](04-smart-contract-test-logs.md) §4.2).

## 3 Changes made during the test round

Besides answering the two issues, Iagon shipped these changes for Würth during the test round:

- **Self-service API keys.** Keys created on the partner dashboard's API tokens page work on the orchestrator beta. A new key is active within about 10 seconds, and revoking one takes effect just as fast, so Würth manages its own keys.
- **Download links tied to orders.** A link to a print file is requested with its order (`orderId` or `orderNumber`, optionally `partNumber` and `customerId`). It is issued only for an existing, non-cancelled order that contains the part. `GET /orders/{id}/links` lists the links for an order and how often each was downloaded.
- **Clearer error answers.** The ORSY API now answers an incomplete request or invalid JSON with 400, or 413 for a body over 250 KB, and a message naming the problem, instead of 500.
- **Owner-only file access in the dashboard.** Only account owners can download files or create share links; members can still upload and organise files.

## 4 Next steps

| # | Step | Why | When |
|---|---|---|---|
| 1 | Support additional printers for print jobs | Agreed with Würth in the review call as the next development step | Next development round |
| 2 | Deploy the contracts to mainnet and report the live addresses and transaction hashes | Milestone 5 output | Milestone 5 |
| 3 | Make the storage connector production-ready, with rollout sign-off by Iagon and Würth for the DIS integration and print queue | Milestone 5 output | Milestone 5 |
| 4 | Commission an external smart contract audit and share its findings and remediations | Milestone 5 output | Milestone 5 |
| 5 | Write the solution risk mitigation plan for Würth's internal compliance and governance | Milestone 5 output | Milestone 5 |

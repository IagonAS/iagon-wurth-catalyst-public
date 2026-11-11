# Milestone 4 – Part 4: Smart Contract Test Logs

## 1 Purpose and scope

This report covers Output 4 of Milestone 4: the logs, walkthroughs and validation results from running the royalty smart contracts on the Cardano preprod testnet. Every transaction in it is public and can be checked on any preprod explorer.

| Evidence | Where it is |
|---|---|
| Evidence 4.1: logs, user interaction data and validation results for tokenization, licence execution and royalty distribution | §3 to §5 and [`onchain-evidence/`](onchain-evidence/) |
| Evidence 4.2: video walkthrough and annotated screenshots of IP upload, licence tracking, history logs and royalty distribution | [`05-walkthrough.mp4`](05-walkthrough.mp4), [`screenshots/`](screenshots/) |
| Evidence 4.3: the full set of logs, videos and validation results as a distinct set of evidence | This report together with [`onchain-evidence/`](onchain-evidence/) |

The log covers every royalty transaction from the first settlement on 29 October 2026 to the first payout on 5 November 2026: 26 settlements and 1 payout.

## 2 Contracts and addresses on preprod

| Item | Value |
|---|---|
| Royalty script (IOU minting policy and treasury spending validator) | `b6a73a46f799c90029baf4e0e623779ada9d98b614df432db3a08eec` |
| Reference script UTxO (read by every transaction) | `1956453abfbad065a5fc11e7e05f89d3242ead60800feffa34e0b691e6e7e569#3` |
| State UTxO (read by every settlement) | At script address `addr_test1wz70scn0lnf29t72ve5j03dmkgxxt9zvk9pl66xz0hhn0zgf6vlu9`; re-created as the system's state changes, so settlements read different UTxOs over time |
| IOU token | Policy `b6a73a46…3a08eec`, asset name `IOU`; 1 IOU = 1 tUSD |
| Treasury (vault) address | `addr_test1wzm2wwjx77vujqpfht6wpe3rw7dd48vckc2d7sedkwsgamqfrmf2n` |
| Test stablecoin (tUSD) | Policy `c39baeddfa567267ddde2c9c8034c924b35d8bb6f2b91e9db637e115`, asset name `tUSD` |
| Maintainer fee address | `addr_test1vpttyvsmdt3rvqdf7ruj6l6j25juvneu87928w7rv4s2xdqa0h5l9` |
| Operator wallet (pays settlement fees) | `addr_test1vpmyudpe83jffrt8cwlmkf7lc2qcz5lazkxzz3jqjtwgsasz6cz54` |

Amounts on chain are in micro-units (1.00 tUSD = 1,000,000). The IOU token has no decimals registered, so explorers show 6.00 IOU as "6.00M". The treasury held 1,000,000 tUSD until the payout on 5 November 2026, and 999,985 tUSD after it. Every IOU is a claim on that treasury, redeemable one to one.

## 3 What each transaction does

**Royalty settlement.** Once an order's 24-hour cancellation window ends, the operator submits a settlement. The settlement:

- spends and re-creates the treasury UTxO under the spending validator;
- mints IOU tokens under the minting policy, equal to the order's royalties plus the maintainer fee;
- sends each vendor's royalty to that vendor's address, and the fee to the maintainer address.

Orders whose windows end close together are settled in one transaction.

**Payout.** When an owner pays out a vendor in the partner dashboard, the transaction burns the vendor's IOU tokens and releases the same amount of tUSD from the treasury to the vendor's address.

In both, the minting policy and the spending validator run together. This is the tokenization step of the system: a licence purchase becomes a royalty token on chain, and that token is later redeemed for stablecoin.

## 4 Validation results

### 4.1 Execution

| Check | Result |
|---|---|
| Transactions confirmed on preprod | 27 of 27 (26 settlements, 1 payout) |
| Scripts run and valid | 27 of 27, each running the minting policy and the spending validator |
| Script cost per transaction | 344k–899k memory units and 127M–320M CPU steps: at most 5.1% and 3.2% of the per-transaction limits (17.5M units, 10B steps) |
| Transaction fee | 0.28–0.35 tADA |
| IOU minted = IOU sent to vendors + maintainer | 26 of 26 settlements |
| Payout: IOU burned = tUSD released = treasury decrease | 15.00 = 15.00 = 15.00 tUSD |

### 4.2 Business rules, order by order

All 32 orders were checked against the chain, using the dashboard's data feed. The feed records every order line with the royalty per piece at the time of the order, and the order's times to the second. [`onchain-evidence/orders-reconciliation.csv`](onchain-evidence/orders-reconciliation.csv) has one row per order with each check.

| Rule | Result |
|---|---|
| Every order that is not cancelled is settled on chain | 31 of 31, in 26 transactions; every settlement transaction belongs to a dashboard order |
| Royalties = pieces × royalty per piece, summed over the order lines | 32 of 32 |
| Maintainer fee = each line's royalty × the tier rate for its royalty per piece, rounded down | 32 of 32 |
| The order is accepted only if royalties + maintainer fee ≤ order total | 32 of 32 |
| Settlement waits for the 24-hour cancellation window | 31 of 31, settled 4 to 236 seconds after the window closed |
| On chain, each vendor and the maintainer receive exactly what their orders' lines add up to | 26 of 26 settlements |
| A cancelled order is never settled | Test order `M4_cancel_test` (1 × GOLF MOUNTING BRACKET, 10.00 tUSD) was placed through ORSY Connect at 22:54:05 UTC on 5 November 2026 and cancelled by an owner in the partner dashboard within four minutes. Its status is cancelled, it has no settlement transaction, no IOU was minted for it, and the vendor's balance is unchanged |

The cancellation check completes the order-cancellation test that the Milestone 3 integration test report scoped to Milestone 4. The fee tiers and the acceptance rule are set out in [`06-refinement-plan.md`](06-refinement-plan.md) §2.

One licence now lists a different royalty from the one its order used. Order `23kk25` (2 × MINI HAND VALVE 2W 6MM) records 1.00 tUSD per piece on its order line and was settled at that rate. The licence was later changed to 98.00 tUSD per piece. Settled orders keep the royalty they were placed with.

### 4.3 Ledger and backing

| Check | Result |
|---|---|
| Dashboard IOU balance = on-chain balance, per holder | 22 of 22 (21 vendors and the maintainer), exact to the micro-unit; see [`onchain-evidence/vendor-balances.csv`](onchain-evidence/vendor-balances.csv) |
| Total IOU issued | 78.2345 tUSD: 75.75 royalties and 2.4845 maintainer fee |
| IOU outstanding after the payout | 63.2345 tUSD: 60.75 owed to vendors and 2.4845 to the maintainer |
| Treasury after the payout | 999,985 tUSD, which covers all outstanding IOU |

## 5 Worked example: order Test_10-1_30

This is the order followed in the walkthrough video.

| Step | Data | Source |
|---|---|---|
| Licence | GOLF MOUNTING BRACKET (IGN-DEMO-P4), Iagon Golf Vendor, 2.00 tUSD per piece | [02](screenshots/02-licence-chain-id.png), [07](screenshots/07-dash-licence.png) |
| Order | WAG BODY SHOP buys 3 pieces for 30.00 tUSD; the orchestrator records it at 14:44:34 UTC on 3 November 2026 (15:44 CET in the dashboard) | [06](screenshots/06-vendor-order.png), [08](screenshots/08-dash-order.png) |
| Royalties | 3 × 2.00 = 6.00 tUSD | |
| Maintainer fee | 2.00 per piece is in the 3.40% tier: 6.00 × 3.40% = 0.204 tUSD | |
| Operator fee | 30.00 − 6.00 − 0.204 = 23.796 tUSD, kept in the ledger | [06](screenshots/06-vendor-order.png) shows 6,000,000 / 204,000 / 23,796,000 micro-units |
| Cancellation window | Open until 14:44:34 UTC on 4 November 2026 | [08](screenshots/08-dash-order.png) |
| Settlement | Transaction `e5d354535feb326e3a440b7453cb76392f79834518178d80155d299b26df7216`, block 5,372,906, 14:44:49 UTC on 4 November 2026: 15 seconds after the window closed | [`05-walkthrough.mp4`](05-walkthrough.mp4), from 6:56 |
| On chain | 6.204 IOU minted: 6.00 to the vendor's address, 0.204 to the maintainer | [11](screenshots/11-explorer-settlement.png) |
| Dashboard | Royalties paid; the vendor is owed 6.00 tUSD as IOU | [09](screenshots/09-dash-royalties-paid.png), [10](screenshots/10-dash-vendor-iou.png) |

The payout completes the cycle for another vendor. Iagon Alpha Vendor collected 15.00 tUSD of IOU from four settlements (3.00 + 3.00 + 1.50 + 7.50). It was paid out on 5 November 2026 in transaction `530a8b602da76e4ee8a3f9c25eac868ee54bf6666497f5ed346adfba2e41db46` (block 5,377,784): 15.00 IOU burned and 15.00 tUSD released from the treasury to the vendor ([13](screenshots/13-dash-payout.png)).

## 6 Evidence files

| File | Content |
|---|---|
| [`onchain-evidence/settlements.csv`](onchain-evidence/settlements.csv) | The 26 settlements: hash, block, time, orders settled, amounts, effective fee rate, vendor outputs, transaction fee, script cost and validity, explorer link |
| [`onchain-evidence/payouts.csv`](onchain-evidence/payouts.csv) | The payout: amounts, treasury before and after, transaction fee, script cost and validity, explorer link |
| [`onchain-evidence/orders-reconciliation.csv`](onchain-evidence/orders-reconciliation.csv) | The 32 orders (31 settled, 1 cancelled): lines, times, amounts, settlement, and each rule check |
| [`onchain-evidence/vendor-balances.csv`](onchain-evidence/vendor-balances.csv) | IOU received, burned and outstanding per holder, against the dashboard |
| [`admin-dashboard-export/`](admin-dashboard-export/) | The partner dashboard's data feed and tables as exported on 5 November 2026 |

To check any transaction independently, open `https://preprod.cexplorer.io/tx/<hash>`, or query a public preprod API such as Koios (`https://preprod.koios.rest/api/v1/tx_info`). Koios `asset_txs` lists the IOU token's full history for policy `b6a73a46f799c90029baf4e0e623779ada9d98b614df432db3a08eec` and asset name `494f55` (`IOU`).

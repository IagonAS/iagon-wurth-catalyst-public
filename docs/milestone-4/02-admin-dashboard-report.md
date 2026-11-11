# Milestone 4 – Part 2: Admin Dashboard Prototype

## 1 Purpose and scope

This report covers Output 2 of Milestone 4, the **Admin Dashboard Prototype**. It is the work summary for evidence 2.1 and indexes the interaction logs and walkthroughs for evidence 2.2.

| Criterion | Where it is answered |
|---|---|
| Success criterion 2.1: admin dashboard deployed on testnet, demonstrating IP uploads, licence execution tracking, transaction history and royalty distribution logs | §2 and §3 |
| Success criterion 2.2: confirmed through interaction logs, video walkthroughs and annotated screenshots | §4 and §6 |
| Success criterion 2.3: feedback from Würth and Iagon teams acting as IP holders, documented in a shared summary | §5 |

## 2 What the admin side consists of

Administration happens in two places, both connected to the same orchestrator on the Cardano preprod testnet.

- **ORSY Connect admin (3D Print Management)**, used by Würth: registering IP holders (Create 3D Vendor, Link 3D Vendor), adding parts and licences (3D Vendor Articles, Licenses), uploading print files (3D File Upload), creating print jobs (3D Print Job) and tracking orders (3D Vendor Order).
- **Iagon partner dashboard**, the ledger view for partners: Overview, Vendors, Licenses, Orders, Payouts and Files.

| Required function | How the prototype shows it | Screenshots |
|---|---|---|
| IP uploads | ORSY 3D File Upload stores the print file in Iagon storage; the dashboard's Files page lists the stored folders | [03](screenshots/03-file-in-storage.png) |
| Licence execution tracking | Licenses lists each licence with its royalty; the vendor view shows orders, pieces and royalties earned per licence | [02](screenshots/02-licence-chain-id.png), [07](screenshots/07-dash-licence.png), [10](screenshots/10-dash-vendor-iou.png) |
| Transaction history | Orders lists every order with its status: Open (with the time left to cancel) until the 24-hour window ends, then Royalties paid with the settlement transaction, or Cancelled if an owner cancelled it in time | [08](screenshots/08-dash-order.png), [09](screenshots/09-dash-royalties-paid.png) |
| Royalty distribution logs | Each vendor's IOU balance, the maintainer fee, the treasury that backs every IOU, and every payout with its transaction | [10](screenshots/10-dash-vendor-iou.png), [12](screenshots/12-dash-treasury.png), [13](screenshots/13-dash-payout.png) |

The ORSY 3D Vendor Order view also shows each order's royalty split as recorded by the orchestrator, for example 6,000,000 micro-units of royalties, 204,000 of maintainer fee and 23,796,000 of operator fee on the walkthrough order ([06](screenshots/06-vendor-order.png)).

## 3 State of the testnet deployment

Figures as exported from the partner dashboard on 5 November 2026, after the test round:

| Measure | Value |
|---|---|
| Vendors (IP holders) registered | 21 |
| Licences in the catalogue | 24 |
| Orders | 32: 31 settled, worth 634.36 tUSD, and 1 cancelled test order (10.00 tUSD) |
| Orders with royalties paid on chain | 31 of the 31 not cancelled |
| Settlement transactions | 26 (some settle several orders together) |
| Royalties issued as IOU | 75.75 tUSD |
| Maintainer fee issued as IOU | 2.4845 tUSD (the dashboard rounds it to 2.48) |
| Operator fee (kept in the ledger, not paid on chain) | 556.1255 tUSD |
| Payouts | 1: 15.00 tUSD to Iagon Alpha Vendor |
| Treasury | 1,000,000 tUSD before the payout, 999,985 tUSD after |

## 4 Interaction logs

The dashboard's data was exported as it stood on 5 November 2026. It is in [`admin-dashboard-export/`](admin-dashboard-export/): the data feed behind the dashboard pages as returned to the browser (`feed/*.json`), and the same data as tables:

| File | Rows | Content |
|---|---|---|
| `orders.csv` | 32 | Order, created and cancel-until times (UTC, to the second), status, total, royalties, maintainer fee, every order line with its royalty at order time, and the settlement transaction hash |
| `licences.csv` | 24 | Part, part numbers, vendor, current royalty per piece, active, created |
| `vendors.csv` | 21 | Vendor, key hash, whether the platform holds the key, IOU owed, payout address, registered |
| `payouts.csv` | 1 | Vendor, amount, status, time requested and transaction hash |
| `files.csv` | 2 | The storage folders (`ORSY_UAT`, `Test-Data`) |

Every order and balance was checked against the chain ([`04-smart-contract-test-logs.md`](04-smart-contract-test-logs.md) §4). All 31 settled orders map to one of the 26 settlement transactions, with the exact amounts; the cancelled order has none. All 22 IOU balances on the dashboard (21 vendors and the maintainer) equal the balances held on chain, to the micro-unit.

One licence now lists a different royalty from the one its order used. MINI HAND VALVE 2W 6MM (vendor "Wurth ca") lists 98.00 tUSD per piece. Its only order, `23kk25` (2 pieces, total 98.36 tUSD), records 1.00 tUSD per piece on its order line and was settled at that rate. The licence was changed after the order; settled orders keep the royalty they were placed with.

## 5 Feedback from Würth and Iagon

**From the testers with admin access.** Ten testers ran the admin tests (2 Admin and 8 Both accounts): two from Würth IT India, one from Würth Canada with an Admin account and one with a Both account, and six from Iagon. All admin tests they ran passed (§5 of [`01-user-portal-beta-report.md`](01-user-portal-beta-report.md)). Their comments on the Iagon dashboard:

- The ORSY vendor appeared on the dashboard, and the order passed the cancellation window as expected (Würth IT India).
- "All looks good at dashboard" (Würth IT India).
- "Easy to follow": View as vendor showed the royalties earned, and the next day the IOU balance had grown by that amount (six testers: Würth Canada and Iagon).
- "Clear": the vendor was listed with its licence (two admin testers).

The only issues raised concerned how royalties and the maintainer fee are calculated and what ORSY should check before submitting an order. Both were resolved during the test round, and the tester who raised them confirmed this ([`06-refinement-plan.md`](06-refinement-plan.md) §2).

**From the review with Würth.** Würth and Iagon reviewed the results in a call. Würth said they were happy with the dashboard and with how quickly the issues were resolved. No outstanding items remain for the dashboard. The next step agreed with Würth is support for additional printers.

## 6 Walkthroughs and screenshots

| File | Content |
|---|---|
| [`05-walkthrough.mp4`](05-walkthrough.mp4), 0:00–6:56 | Architecture, then the seven steps from registering the IP holder to the order in the partner dashboard |
| [`05-walkthrough.mp4`](05-walkthrough.mp4), 6:56–8:06 | The same order after its 24-hour window, the settlement transaction on the explorer, and the treasury |

Screenshots 01–12 are stills from the walkthrough, with the step caption in the image. Screenshot 13 is a separate capture of the Payouts page. ORSY Connect UAT shows times in Eastern time; the partner dashboard shows CET.

| Screenshot | Caption in the image | What it shows |
|---|---|---|
| [01-vendor-linked.png](screenshots/01-vendor-linked.png) | 1 · Register the IP holder · ORSY admin | The vendor linked to the orchestrator, with its UUID and public key hash |
| [02-licence-chain-id.png](screenshots/02-licence-chain-id.png) | 2 · License a part · ORSY admin | The licence listed with its Chain ID |
| [03-file-in-storage.png](screenshots/03-file-in-storage.png) | 3 · Upload the print file to Iagon storage · ORSY admin | The print file stored |
| [04-print-queue.png](screenshots/04-print-queue.png) | 4 · Print queue · ORSY admin | The print job queued |
| [05-order-submitted.png](screenshots/05-order-submitted.png) | 5 · Buy a licence to print · ORSY customer (body shop) | Confirmation that the purchase order was sent to the orchestrator |
| [06-vendor-order.png](screenshots/06-vendor-order.png) | 6 · Order tracking · ORSY admin | The order with its royalty, maintainer fee and operator fee |
| [07-dash-licence.png](screenshots/07-dash-licence.png) | 7 · Iagon partner dashboard | The licence in the dashboard |
| [08-dash-order.png](screenshots/08-dash-order.png) | 7 · Iagon partner dashboard | Orders inside the 24-hour window, with the time left to cancel |
| [09-dash-royalties-paid.png](screenshots/09-dash-royalties-paid.png) | 8 · Royalty settled · Iagon partner dashboard | Orders with royalties paid and their transactions |
| [10-dash-vendor-iou.png](screenshots/10-dash-vendor-iou.png) | 8 · Royalty settled · Iagon partner dashboard | The vendor view with the IOU owed |
| [11-explorer-settlement.png](screenshots/11-explorer-settlement.png) | 9 · Settlement transaction · Cardano preprod explorer (Cexplorer) | The settlement's value flow: IOU to the vendor and the maintainer |
| [12-dash-treasury.png](screenshots/12-dash-treasury.png) | 10 · Treasury · Iagon partner dashboard | The treasury and the maintainer fee |
| [13-dash-payout.png](screenshots/13-dash-payout.png) | (none) | The Payouts page after the first payout, 5 November 2026 |

# Milestone 4 – Complete Initial Development Prototype on Testnet

Iagon & Würth, Project Catalyst Fund 13, project 1300111.

The prototype runs on the Cardano preprod testnet. It connects Würth's ORSY Connect storefront to the Iagon royalty orchestrator, the Iagon partner dashboard, the Iagon storage connector and the royalty smart contracts. Testing ran from 28 October to 5 November 2026. 25 testers ran 101 tests and all passed. All 31 orders that were not cancelled settled on chain, a test order cancelled inside its 24-hour window was never settled, and the first payout was redeemed from the treasury.

## Evidence map

| Evidence item (statement of milestone) | Where it is |
|---|---|
| 1.1 Feedback summary spreadsheet from 25 unique testers of the user portal | [`tester-feedback/tester-feedback-summary.xlsx`](tester-feedback/tester-feedback-summary.xlsx), with the responses in [`tester-feedback/responses/`](tester-feedback/responses/) and the report [`01-user-portal-beta-report.md`](01-user-portal-beta-report.md) |
| 1.2 Refinement planning file with next steps and conclusions | [`06-refinement-plan.md`](06-refinement-plan.md) |
| 2.1 Work summary with feedback from Würth and Iagon on the admin dashboard | [`02-admin-dashboard-report.md`](02-admin-dashboard-report.md), with the feedback in §5 |
| 2.2 Interaction logs and video walkthroughs from the testnet admin dashboard | [`admin-dashboard-export/`](admin-dashboard-export/), [`05-walkthrough.mp4`](05-walkthrough.mp4), [`screenshots/`](screenshots/) |
| 3.1 Component testing report for the Iagon storage connector | [`03-storage-connector-report.md`](03-storage-connector-report.md) §3 |
| 3.2 Test scenarios, outcomes and validation of the integration with DIS in a demo environment | [`03-storage-connector-report.md`](03-storage-connector-report.md) §4 |
| 4.1 Logs, user interaction data and validation results for the smart contracts on testnet | [`04-smart-contract-test-logs.md`](04-smart-contract-test-logs.md), [`onchain-evidence/`](onchain-evidence/) |
| 4.2 Video walkthrough and annotated screenshots of IP upload, licence tracking, history logs and royalty distributions | [`05-walkthrough.mp4`](05-walkthrough.mp4), [`screenshots/`](screenshots/) |
| 4.3 Full set of logs, walkthrough videos and validation results as a distinct set of evidence | [`04-smart-contract-test-logs.md`](04-smart-contract-test-logs.md) with [`onchain-evidence/`](onchain-evidence/) |

Würth Canada's signed partner acceptance statement for Milestone 4 is provided with the proof of achievement.

## Folder layout

| Path | Content |
|---|---|
| `01-user-portal-beta-report.md` | Part 1: user portal beta, test method, testers, results and feedback |
| `02-admin-dashboard-report.md` | Part 2: admin dashboard, interaction logs, feedback from Würth and Iagon |
| `03-storage-connector-report.md` | Part 3: storage connector, component tests and DIS integration |
| `04-smart-contract-test-logs.md` | Part 4: contracts, transactions and validation results |
| `05-walkthrough.mp4` | Walkthrough, about 8 minutes: from IP holder registration to the order in the partner dashboard (0:00–6:56), then the royalty settled on chain after the 24-hour window, and the treasury (6:56–8:06) |
| `06-refinement-plan.md` | Conclusions, feedback incorporated, changes made and next steps |
| `tester-feedback/` | Feedback summary spreadsheet and the 25 tester responses |
| `admin-dashboard-export/` | Partner dashboard data feed and tables as exported on 5 November 2026 |
| `onchain-evidence/` | Settlements, payout, order reconciliation and vendor balances, with explorer links |
| `screenshots/` | 12 annotated stills of the walkthrough steps, and a capture of the Payouts page |

## Notes for reviewers

- **Tester responses.** The response PDFs are Iagon's transcriptions of the forms testers returned by email, in one layout, with names and email addresses redacted; only email domains are shown. Iagon holds the originals.
- **Videos and screenshots.** ORSY Connect UAT is Würth's non-public test system. Everything shown from it that the walkthrough did not create is blurred: personal data, and also other vendors, parts, files, print jobs, orders, customers and dropdown lists. The walkthrough's own records (Iagon Golf Vendor, IGN-DEMO-P4, its licence, print file, print job and order Test_10-1_30) stay visible, and so does all on-chain data.
- **Closed-source pilot.** As in earlier milestones, implementation details are generalised where they touch security or partner-confidential information.
- **Independent checks.** Every transaction hash in this folder can be opened on a public preprod explorer, for example `https://preprod.cexplorer.io/tx/<hash>`.

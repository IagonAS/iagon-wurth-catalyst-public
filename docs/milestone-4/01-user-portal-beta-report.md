# Milestone 4 – Part 1: User Portal Beta and Tester Feedback

## 1 Purpose and scope

This report covers Output 1 of Milestone 4, the **User Portal Beta**. It shows that the beta lets users browse IP, buy a print licence and submit print jobs on the Cardano preprod testnet, with royalties settled by the smart contracts. It answers success criteria 1.1 to 1.3:

| Criterion | Where it is answered |
|---|---|
| Success criterion 1.1: functional beta user portal meets defined test scenarios | §3 and §5 of this report |
| Success criterion 1.2: feedback from at least 25 unique testers is collected and incorporated into refinement planning | §4, §6, and [`06-refinement-plan.md`](06-refinement-plan.md) |
| Success criterion 1.3: a summary document presents conclusions and outlines next steps | [`06-refinement-plan.md`](06-refinement-plan.md) |

The feedback summary spreadsheet (evidence 1.1) is [`tester-feedback/tester-feedback-summary.xlsx`](tester-feedback/tester-feedback-summary.xlsx). The 25 individual responses are in [`tester-feedback/responses/`](tester-feedback/responses/).

## 2 The user portal beta

The buyer-facing portal is Würth's own storefront, **ORSY Connect**, as decided in Milestone 3. No separate storefront was built. In the beta, ORSY Connect (UAT environment) is connected to the Iagon royalty orchestrator, which settles royalties on the Cardano preprod testnet.

| Function | Where it happens | Screenshot |
|---|---|---|
| IP browsing | ORSY Connect: a body shop picks a 3D-print supplier and browses its printable parts | Walkthrough video, about 4:20–4:40 |
| Licence purchase | ORSY Connect: the purchase order is sent to the orchestrator, which records the order and its royalty | [05](screenshots/05-order-submitted.png) (order sent), [06](screenshots/06-vendor-order.png) (order recorded) |
| Print-queue submission | ORSY admin: the print file is uploaded to Iagon storage and a print job is created | [03](screenshots/03-file-in-storage.png), [04](screenshots/04-print-queue.png) |
| Smart contract execution | After the 24-hour cancellation window, the royalty is minted on chain as IOU tokens to the vendor | [09](screenshots/09-dash-royalties-paid.png), [11](screenshots/11-explorer-settlement.png) |

The walkthrough video ([`05-walkthrough.mp4`](05-walkthrough.mp4), 0:00–6:56) shows these steps end to end. The on-chain side is documented in [`04-smart-contract-test-logs.md`](04-smart-contract-test-logs.md).

## 3 Test method

Each tester received a login to ORSY Connect UAT and a two-page test form with eight scenarios. They ran the tests their account allowed, marked each one Pass, Fail or N/A, rated how easy it was, and returned the form by email. The forms are dated 29 October to 3 November 2026, as written by the testers. Test 8 is checked on the working day after the order, so a form's date is usually the day its tester started.

| # | Who | Test | Steps | Expected result |
|---|---|---|---|---|
| 1 | Shop | Browse 3D parts | Purchase Order → Add Purchase Order, pick a 3D-print supplier, open Browse Parts | The supplier's 3D-printable parts are listed |
| 2 | Shop | Buy a print licence | Add a part, enter a quantity and click Submit | ORSY confirms the order was sent to the orchestrator |
| 3 | Admin | Register a vendor | Create 3D Vendor, then link it in Link 3D Vendor | The vendor is listed with a UUID and public key hash |
| 4 | Admin | License a part | Add the part in 3D Vendor Articles, then create its licence with a royalty per piece | Licenses lists it with a Chain ID |
| 5 | Admin | Upload the print file | 3D File Upload: pick the supplier, add a .3mf or .gcode file, Upload to Storage | The file is listed |
| 6 | Admin | Send to print queue | Add Print File Metadata (product, printer, material), then Create Print Job | 3D Print Job lists the job with a print status |
| 7 | Admin | Track the order | Iagon partner dashboard: View as vendor for the supplier of test 2 | The order is listed; royalties earned = pieces × royalty per piece |
| 8 | Admin | Royalty distribution | The next working day, open the same vendor again | The order has left the cancel window; the IOU balance grew by its royalty |

Test 8 confirms that the royalty was settled on the Cardano testnet. The Chain ID in test 4 is the orchestrator's identifier for the licence; royalties are tokenized on chain when an order settles ([`04-smart-contract-test-logs.md`](04-smart-contract-test-logs.md)). Shop accounts (an ORSY Connect buyer) run tests 1–2, Admin accounts run tests 3–8, and Both run all eight.

**Privacy.** The published response PDFs are Iagon's transcriptions of the returned forms into one layout, with tester names and email addresses redacted; only the email domain is shown. The originals are held by Iagon.

## 4 Testers

25 unique testers took part.

| Organisation | Testers |
|---|---|
| Würth Canada | 13 |
| Würth IT India (ORSY Connect team) | 2 |
| Iagon | 10 |
| **Total** | **25** |

| Account | Tests it covers | Testers |
|---|---|---|
| Shop | 1–2 | 15 |
| Admin | 3–8 | 2 |
| Both | 1–8 | 8 |

## 5 Results

All 101 tests that were run passed. No test failed. N/A means the tester did not run the test: shop accounts have no admin access (tests 3–8), and the two Admin-only testers had no order of their own to track (tests 7–8).

| # | Test | Pass | Fail | N/A |
|---|---|---|---|---|
| 1 | Browse 3D parts | 23 | 0 | 2 |
| 2 | Buy a print licence | 23 | 0 | 2 |
| 3 | Register a vendor | 9 | 0 | 16 |
| 4 | License a part | 10 | 0 | 15 |
| 5 | Upload the print file | 10 | 0 | 15 |
| 6 | Send to print queue | 10 | 0 | 15 |
| 7 | Track the order | 8 | 0 | 17 |
| 8 | Royalty distribution | 8 | 0 | 17 |
| | **Total** | **101** | **0** | **99** |

One admin tester marked test 3 N/A because they added their part to an existing vendor instead of creating a new one.

**Ease of use.** Asked how easy the tests were to complete (1 = hard, 5 = easy), 24 testers answered 5 and one answered 4. The average is 4.96.

## 6 Feedback

Each response's written feedback was coded into themes. The coding is in the Themes column of the spreadsheet's Responses sheet.

| Code | Theme | Testers |
|---|---|---|
| P1 | Parts were easy to find under their supplier in ORSY | 21 |
| P2 | Ordering was clear: ORSY confirmed the order was sent to Iagon | 23 |
| P3 | Vendor registration worked at once (UUID and key hash issued) | 9 |
| P4 | Licence, print-file upload and print job worked first time | 10 |
| P5 | The dashboard tracked the order and its royalty through the end of the lock period | 8 |
| P6 | The dashboard listed the vendor and its licence | 4 |
| I1 | Explain how total royalties and the maintainer fee are calculated | 1 |
| I2 | Define what ORSY should validate before submitting an order, to avoid order failures and 400 Bad Request errors | 1 |

The two issues (I1, I2) came from one tester on the Würth IT India team, who had seen an order rejected with a 400 error. Both were resolved during the test round. Iagon explained the calculation by email, using an order of the tester's that had been rejected as a worked example, and again in a call. The rules are documented in the partner dashboard (API docs, Getting started guide, Fees). The tester confirmed that this answered both points. The fee rules are set out in [`06-refinement-plan.md`](06-refinement-plan.md) §2.

No tester asked for a change to the portal: all 25 answered "What should we change first?" with "Nothing for now" or "NA".

## 7 Conclusion

The user portal beta meets the defined test scenarios: every test that was run passed, for testers from Würth Canada, Würth IT India and Iagon. The feedback is incorporated into the refinement plan, [`06-refinement-plan.md`](06-refinement-plan.md), which also sets out the next steps.

## 8 Evidence in this folder

| File | What it holds |
|---|---|
| [`tester-feedback/tester-feedback-summary.xlsx`](tester-feedback/tester-feedback-summary.xlsx) | Summary (counts, results by test, ratings, themes), one row per tester, and the test scenarios |
| [`tester-feedback/responses/`](tester-feedback/responses/) | The 25 responses (`response1.pdf` … `response25.pdf`, redacted) |
| [`05-walkthrough.mp4`](05-walkthrough.mp4) | Walkthrough, about 8 minutes: from IP holder registration to the royalty settled on chain |
| [`screenshots/`](screenshots/) | Annotated screenshots 01–06 of the ORSY Connect steps |

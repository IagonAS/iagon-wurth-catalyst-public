# Milestone 4 – Part 3: Iagon Storage Connector, Initial Implementation

## 1 Purpose and scope

This report covers Output 3 of Milestone 4: a basic implementation of the Iagon storage connector, tested as a component and deployed in a demo environment. It is the component testing report (evidence 3.1) and records the test scenarios and outcomes of the integration with Würth's Digital Inventory Services, DIS (evidence 3.2).

| Criterion | Where it is answered |
|---|---|
| Success criterion 3.1: a test version of the storage connector is available in a demo environment | §2 |
| Success criterion 3.2: integrated component testing is completed and documented | §3 and §4 |

The connector follows the design from Milestone 3, [`../milestone-3/04-storage-connector-design.md`](../milestone-3/04-storage-connector-design.md). Implementation details, including parts of the interface, have evolved since then and will keep changing until the production-ready connector, a Milestone 5 output.

## 2 The implementation and its demo environment

The connector implements the compatibility gateway designed in Milestone 3. It presents an established enterprise object-storage interface to Würth's systems and translates each request into Iagon storage operations. Würth's systems use standard storage tooling, and the files live in Iagon's decentralised storage.

In the demo environment, the connector serves Würth's ORSY Connect UAT and DIS:

- ORSY admin's **3D File Upload** stores print files (.3mf, .gcode) in Iagon storage through the connector, under the partner's `ORSY_UAT` folder.
- **3D Print Job** creates a print job for a stored file, with its printer and material.
- Würth integrated **DIS** with Iagon storage through the connector. Würth confirmed in the review call that file upload and download work end to end.

During the test round, access to the stored files was tightened. These rules are enforced by the orchestrator's ORSY API and the partner dashboard, not by the connector itself:

- Download links for a print file are now tied to an order (`orderId` or `orderNumber`, optionally with `partNumber` and `customerId`). A link is issued only for an existing, non-cancelled order that contains the part. `GET /orders/{id}/links` lists every link requested for an order and how often each was downloaded.
- In the partner dashboard, only account owners can download files or create share links. Members can still upload and organise files.

## 3 Component testing

The connector's automated test suite was run on 5 November 2026 against the head of the connector repository's main branch, commit `ce5c01d` (6 August 2026). Component and test names are generalised here, as in Milestone 3; the full per-test log is held by Iagon and available to reviewers on request.

| Component | What the tests check | Tests | Result |
|---|---|---|---|
| Request authentication | Signed requests: missing or malformed credentials, wrong account, expired timestamp, bad signature, valid signature, query parameters in the signed resource | 7 | 7 passed |
| Request routing | HEAD reads properties only, GET downloads | 2 | 2 passed |
| Error mapping | Iagon errors become the matching storage-interface errors: duplicate path → 409 conflict, missing → 404, others → 500 with the upstream detail | 8 | 8 passed |
| Iagon client | The overwrite flag is sent only when requested | 2 | 2 passed |
| Path resolution and cache | Object paths resolve to Iagon entries; listing by prefix and delimiter; containers are top-level folders; cache reuse and invalidation | 13 | 13 passed |
| Upload semantics | An unconditional upload overwrites; a create-only upload refuses to overwrite and returns 409 | 3 | 3 passed |
| **Total** | | **35** | **35 passed, 0 failed** |

The suite also has 12 integration tests that drive a running gateway with the storage interface's official client SDK. They create, list and delete containers, and upload, read, overwrite, list and delete files. They need a running gateway and its storage credentials, so they were not part of this offline run. The live equivalent is the DIS integration in §4, which exercised upload and retrieval through the deployed connector.

## 4 Integration with DIS: scenarios and outcomes

| Date (2026) | Scenario | Outcome |
|---|---|---|
| 25 October | Iagon end-to-end run on ORSY Connect UAT: three vendors, each with a part, a licence and a print file uploaded to storage | Uploads: 3 of 3 stored under `/ORSY_UAT/PrintFiles`. Print job creation failed because the product list was empty; fixed before the tester round |
| 29 October | Würth IT India testers (ORSY Connect team) upload their own print files (`DIPMilwaukees.3mf`, `DIPMetabo_29102026.3mf`) and send them to the print queue | Both files stored in Iagon storage and queued |
| 29 October – 3 November | Tester round: test 5 (upload the print file) and test 6 (send to print queue), see [`01-user-portal-beta-report.md`](01-user-portal-beta-report.md) | Test 5: 10 of 10 passed. Test 6: 10 of 10 passed |
| 3 November | Walkthrough: `iagon-golf-bracket.3mf` uploaded for Iagon Golf Vendor, then a print job created (Bambu Lab X1C, PETG) | File stored, job queued ([03](screenshots/03-file-in-storage.png), [04](screenshots/04-print-queue.png), [`05-walkthrough.mp4`](05-walkthrough.mp4)) |
| Review call | Würth confirms DIS is integrated with Iagon storage through the connector | Upload and download working |

The test round exercised upload through the connector directly (tests 5 and 6). Download is confirmed by Würth's DIS integration, which retrieves the stored print files.

## 5 Conclusion

A working connector is deployed in the demo environment, its component tests pass, and Würth has integrated DIS with it. Hardening the connector for production is part of Milestone 5.

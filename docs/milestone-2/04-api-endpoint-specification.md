# Milestone 2 – Part 4: API Endpoint Specification

## 1 Purpose and Scope

### 1.1 Purpose of this document

This document is the public-facing summary of the orchestrator's HTTP API surface as it stands at the end of Milestone 2 of the Iagon & Würth Catalyst Fund 13 project.

The orchestrator is the off-chain coordinator responsible for royalty processing and the associated business logic — vendor and licence registration, order intake, settlement state, and payout execution. It does not handle storage of IP-protected build files; those concerns are addressed by separate components that are not exposed through the orchestrator API.

It addresses the Milestone 2 *API Endpoint Specification Document* output, which calls for:

- an internal document listing the agreed API endpoints related to **license execution**, **royalty distribution**, and **IP management**,
- validated jointly by Iagon and Würth,
- shared with Catalyst reviewers to confirm the current state of endpoint design and integration logic.

The document is intentionally written at the same level of abstraction as the Milestone 1 deliverables — sufficient to validate scope, responsibilities, and theme coverage without disclosing internal implementation details that fall under the closed-source enterprise pilot agreement.

### 1.2 Scope boundaries

In scope for this document:

- the orchestrator's HTTP API, which is the only API surface that is publicly referenced for Milestone 2,
- a thematic map from each endpoint to one of the three Milestone 2 outputs (IP management, license execution, royalty distribution),
- the supporting endpoints needed to operate and integrate the orchestrator (health, configuration, ORSY enterprise adapter),
- the location of the rendered API definition and the access model under which Catalyst reviewers can browse it.

Out of scope for this document:

- the Transaction Builder, Indexer, and chain-simulator HTTP APIs (internal services consumed by the orchestrator; not part of the public Milestone 2 surface),
- request and response payload structures, error catalogues, and field-level validation rules (available in the rendered API definition referenced in Section 2),
- authentication and authorisation internals beyond what is needed to access the rendered definition itself (covered by the closed-source security materials).

### 1.3 Relationship to other Milestone 2 deliverables

- **Part 1 — Smart Contract Prototype Report** describes the on-chain primitives that the endpoints listed here ultimately drive.
- **Part 2 — Royalty Mechanism Test Report** demonstrates how the order, dashboard, and payout endpoints behave end-to-end against the chain-simulator and against Cardano preprod.
- **Part 3 — Off-chain API Prototype Report** describes the services that host these endpoints and the ORSY integration that exercises them.

## 2 Rendered API Definition

The orchestrator's API is described by an OpenAPI 3.x specification (`orchestrator.yaml`), maintained alongside the orchestrator service and serving as the single source of truth for the API. A static, single-page rendering of that specification is published for joint Iagon × Würth review and for Catalyst reviewers. The rendered page is a read-only specification view; it does not allow requests to be issued against any environment.

The site is hosted at <https://orchestrator-beta.iagon.com/api-docs> and gated by HTTP Basic Auth, independent of any other session. Credentials are shared out-of-band as part of the milestone evidence package and should be treated as sensitive.

## 3 Endpoint Catalogue by Consumer

The orchestrator exposes two clearly distinguished groups of endpoints. The first group is shaped by the needs of the orchestrator user interface — the operator and vendor dashboard application — and is what Catalyst reviewers will see exercised in the milestone walkthrough video. The second group is the integration surface used by Würth's enterprise systems (ORSY) and by other programmatic callers, and is what drives the royalty flow end-to-end in production conditions.

A third, smaller group covers operations and access discovery and is documented separately for completeness.

In every table, the **M2 theme** column maps each endpoint to one of the three Catalyst Milestone 2 outputs — *IP management*, *license execution*, and *royalty distribution* — and marks supporting operational endpoints as *Operations*.

### 3.1 Orchestrator user interface

These endpoints are consumed by the orchestrator user interface (the operator and vendor dashboard application). The shapes of their responses are driven by the dashboard's display needs. The dashboard itself is used by Würth, Würth Additive Group, and Iagon personnel; Catalyst reviewers do not interact with it directly but see it exercised in the milestone walkthrough video.

| Method | Path | Operation | Purpose | M2 theme |
|---|---|---|---|---|
| `GET` | `/auth/config` | `getAuthConfig` | Discovery endpoint returning the authentication configuration the SPA needs to initialise its sign-in flow. | Operations |
| `GET` | `/vendors` | `listVendors` | List the registered vendors and their status; backs the vendor list view. | IP management |
| `GET` | `/licenses` | `listLicenses` | List the registered licences; backs the licence catalogue view. | IP management |
| `GET` | `/orders/{id}` | `getOrder` | Retrieve an order by its internal identifier; backs the order detail view. | License execution |
| `GET` | `/orders/by-on-chain-id/{onChainId}` | `getOrderByOnChainId` | Retrieve an order by its on-chain identifier; backs lookups from settlement-side references. | License execution |
| `GET` | `/dashboard/operator` | `getOperatorDashboard` | Operator-side aggregated view of orders, settlement state, and payout positions. | Royalty distribution |
| `GET` | `/dashboard/vendor/{id}` | `getVendorDashboard` | Vendor-side view of an individual vendor's royalty position and history. | Royalty distribution |
| `POST` | `/payouts/vendor/{id}` | `payoutVendor` | Drain a vendor's bucket: burn accumulated royalty credits and release the corresponding stablecoin to the vendor. Triggered from a vendor detail action. | Royalty distribution |
| `POST` | `/payouts/operator` | `payoutOperator` | Drain the operator's system bucket. Triggered from the operator dashboard action. | Royalty distribution |

### 3.2 External integrations and royalty flows

These endpoints form the integration surface that lets external systems drive the orchestrator end-to-end. They divide into four sub-groups: one enterprise adapter for Würth's **ORSY** product and order system, and three sets of direct resource APIs for vendors, licences, and orders.

The direct resource APIs in Sections 3.2.2 through 3.2.4 run in parallel with the equivalent ORSY-adapter endpoints in Section 3.2.1; both surfaces exist on purpose so that ORSY-originated events and non-ORSY callers (administrators, automation scripts, the chain-simulator-backed demo flow) can drive the same downstream state through the same business rules.

#### 3.2.1 ORSY enterprise adapter — Würth

These endpoints accept payloads in the shapes produced by Würth's ORSY system and return responses that are likewise **ORSY-compatible in both style and content**. Field naming, identifier formats, and payload structure follow ORSY's conventions on the request and response sides alike, so ORSY can consume the orchestrator's responses without translation on its end.

Internally, each ORSY-adapter endpoint delegates to the **same business service** that backs the corresponding direct endpoint in Sections 3.2.2 through 3.2.4. The adapter is a translation layer over the orchestrator's vendor, licence, and order services, not a separate implementation — so ORSY-originated calls and direct calls produce identical downstream state and apply identical business rules.

| Method | Path | Operation | Purpose | Direct equivalent | M2 theme |
|---|---|---|---|---|---|
| `POST` | `/adapters/orsy/vendors` | `createVendorFromOrsy` | Create a vendor from an ORSY-shaped payload. | `POST /vendors` | IP management |
| `POST` | `/adapters/orsy/licenses` | `createLicenseFromOrsy` | Create a licence from an ORSY-shaped payload. | `POST /licenses` | IP management |
| `POST` | `/adapters/orsy/orders` | `submitOrderFromOrsy` | Submit an order from an ORSY-shaped payload. | `POST /orders` | License execution |

#### 3.2.2 Direct vendor management

| Method | Path | Operation | Purpose | M2 theme |
|---|---|---|---|---|
| `POST` | `/vendors` | `createVendor` | Register a new vendor (IP owner) and provision the supporting on-chain bucket. | IP management |
| `PATCH` | `/vendors/{id}` | `updateVendor` | Update vendor metadata. | IP management |

#### 3.2.3 Direct licence management

| Method | Path | Operation | Purpose | M2 theme |
|---|---|---|---|---|
| `POST` | `/licenses` | `createLicense` | Register a new print licence and associate it with a vendor. | IP management |
| `PATCH` | `/licenses/{id}` | `updateLicense` | Update licence metadata. | IP management |

#### 3.2.4 Direct order management

| Method | Path | Operation | Purpose | M2 theme |
|---|---|---|---|---|
| `POST` | `/orders` | `submitOrder` | Submit a new order through the orchestrator's direct API. | License execution |
| `DELETE` | `/orders/{id}` | `cancelOrder` | Cancel an order within the configured cancellation window. | License execution |

### 3.3 Operations and access

These endpoints support deployment, monitoring, and runtime health checks. They are not part of any Catalyst Milestone 2 theme but are listed here for completeness of the public surface.

| Method | Path | Operation | Purpose |
|---|---|---|---|
| `GET` | `/health` | `getHealth` | Detailed health check, including downstream dependency status. |
| `GET` | `/health/live` | `getLiveness` | Liveness probe for the container runtime. |
| `GET` | `/health/ready` | `getReadiness` | Readiness probe; reports whether the orchestrator is prepared to accept traffic. |

## 4 Summary

The orchestrator's HTTP API at Milestone 2 is the only API surface published for external review. It is structured around two clearly separated consumer audiences:

- the **orchestrator user interface** (Section 3.1), which is what reviewers will see exercised during the milestone walkthrough; and
- the **external integrations and royalty flows** (Section 3.2), which is the surface that lets Würth's ORSY system and other programmatic callers drive the royalty mechanism end-to-end.

Across both groups the API covers each of the three Milestone 2 themes named in the Catalyst output language: *IP management*, *license execution*, and *royalty distribution*. The operations endpoints in Section 3.3 are listed for completeness only.

The complete, field-level specification is available at <https://orchestrator-beta.iagon.com/api-docs> under the access model described in Section 2. Reviewers should treat the catalogue above as the navigation map and the rendered definition as the authoritative specification.

# Milestone 1 – Part 3: Blockchain Solution Selection

## 1 Purpose and Scope

### 1.1 Purpose of this document

Define and justify the blockchain infrastructure selection for the Würth pilot by:

* evaluating Cardano ecosystem execution options that affect delivery timeline, operational scope, and production readiness
* documenting constraints and trade-offs that influence smart contract design assumptions (cost, throughput, payload size, and operational complexity)
* providing a defensible baseline decision that can be validated internally by Iagon and Würth

### 1.2 Scope boundaries

In scope:

* Cardano L1 as the primary execution and settlement layer
* analysis of scaling paths (Hydra, emerging L2 solutions, Partner Chains) as future options
* operational considerations that affect delivery risk and ongoing maintenance burden

Out of scope (Milestone 1):

* detailed implementation planning for any non-L1 deployment (Hydra Head setup, Partner Chain governance, or emerging L2 integration)
* performance benchmarking or load testing of selected infrastructure

## 2 Executive Summary

This document presents an analysis of available blockchain infrastructure options within the Cardano ecosystem such as Hydra, emerging L2 solutions (such as Midgard), and Partner Chains, and recommends Cardano L1 as the primary execution and settlement layer.

The recommendation is grounded in three factors: (1) Cardano offers the most mature and operationally straightforward path to production, (2) alternative scaling solutions, whether Hydra, emerging L2s, or Partner Chains, introduce complexity that is not justified by current requirements, and (3) existing infrastructure is purpose-built for Cardano. These alternatives should be treated as optional future scaling lanes to be adopted only when concrete requirements demand them.

## 3 Options Evaluated

### 3.1 Cardano Blockchain

**Capabilities**

* A mature, production-hardened environment
* Straightforward operational model with no additional networks, committees, bridges, or L2 lifecycle management
* Native compatibility with existing wallets, explorers, and infrastructure that users already rely on

**Tradeoffs**

* L1 throughput and transaction size limits remain real constraints; highly interactive workloads may become impractical or cost-prohibitive at extreme scale

**Assessment**

Cardano represents the **lowest-risk path to production** and should be the default choice unless specific requirements clearly justify additional infrastructure.

### 3.2 Hydra Head

**Capabilities**

* Provides a clear scaling model: execute high-frequency interactions off-chain within a Hydra Head and settle results back to L1
* Positioned as a production-ready scaling direction in official ecosystem communications
  *Reference: [Scaling Cardano Applications with Hydra](https://cardano.org/news/2025-10-27-scaling-cardano-applications-with-hydra/)*

**Operational Considerations**

* Hydra is not a simple contract deployment. It introduces a protocol lifecycle (head open/close, UTxO commit/decommit, participant coordination) and requires dedicated infrastructure (Hydra nodes per participant)
* Official documentation includes explicit guidance to review known issues and limitations before operating with real funds
  *Reference: [Hydra Head Protocol Documentation](https://hydra.family/head-protocol/docs)*
  *Reference: [Known Issues](https://hydra.family/head-protocol/docs/known-issues)*

**Assessment**

Hydra is a valid scaling solution for use cases requiring high-frequency, low-latency interactions that L1 cannot reasonably support. However, for current requirements, Hydra introduces operational complexity without proportional benefit. It remains a viable option for the future if usage patterns evolve to require it.

### 3.3 Partner Chains Toolkit

**Capabilities**

* A framework for building sovereign chains that can leverage Cardano stake pool operators for shared security and validation
* Active development with documentation available for operators
  *Reference: [Partner Chains Repository](https://github.com/input-output-hk/partner-chains)*

**Maturity Considerations**

* Initial release communications positioned the toolkit as a milestone and feedback phase, explicitly noting it was not intended for live production networks at that stage
  *Reference: [First Release of Partner Chains Toolkit](https://cardano.org/news/2024-08-01-first-release-of-the-partner-chains-toolkit/)*
* Adopting Partner Chains expands scope into operating a separate network: governance and committee design, monitoring and alerting, upgrade coordination, incident response, bridging mechanics, and supporting infrastructure (indexers, explorers, etc.)

**Case Study: Midnight, The Only Public Live Partner Chain**

Midnight is currently the sole public partner chain built using the Partner Chains toolkit, providing a concrete reference point for evaluating this approach:

* **Current Status:** Midnight launched in December 2025 and is now in the Kūkolu phase, with federated mainnet and complete decentralization phases scheduled throughout 2026. The network is actively working toward its genesis block and broader stake pool operator participation.
  *Reference: [Midnight Network](https://midnight.network/)*
  *Reference: [Midnight Partner Chain Overview](https://cexplorer.io/article/midnight-is-a-partner-chain)*

* **Development Timeline:** Despite significant resources from Input Output and partnerships with entities including Google Cloud and Brave, Midnight has required multi-year development to reach its current pre-mainnet state. Full interoperability and hybrid dApp capabilities are projected for late 2026 or beyond.

* **Architectural Note:** Midnight operates with its own consensus and ledger, built from the ground up. As Midnight's leadership has clarified, a partner chain is not an L2 solution; it is an independent network that shares security properties with Cardano through stake pool operator participation.

* **Implication** If the most well-funded partner chain project in the ecosystem is still progressing through mainnet phases in 2026, this underscores that partner chains require substantial investment in network operations, governance, and ecosystem tooling. For projects without Midnight's scale of resources, this path carries significant execution risk.

**Assessment**

Partner Chains are strategically compelling when a project genuinely requires a sovereign execution environment, such as custom virtual machines, specialized fee structures, or domain-specific throughput requirements. However, Midnight's trajectory demonstrates that even well-resourced partner chain efforts require extended development timelines. For current objectives, Partner Chains represent an R&D investment rather than a production default. The expansion of the operational scope is not justified without a concrete requirement.

### 3.4 Emerging L2 Solutions (Midgard and Others)

Beyond Hydra, additional Layer 2 scaling solutions are under active development within the Cardano ecosystem.

**Midgard (Anastasia Labs / FluidTokens)**

Midgard is an L2 scaling solution designed to address network congestion during high-volume activities such as NFT launches and DeFi transactions.

* **Technical Approach:** Midgard employs an isomorphic design, allowing existing dApps to redeploy without code modification. It targets 3-second block times (with further optimization planned), larger block sizes, and expanded script execution units. Security is maintained through periodic checkpoints published to L1.
  *Reference: [Midgard - Project Catalyst](https://projectcatalyst.io/funds/12/cardano-use-cases-product/anastasia-labs-midgard-cardano-layer-2)*

* **Current Status:** As of early 2026, Midgard has completed 2 of 6 development milestones (technical architecture specification and protocol smart contract specs). State management contracts are currently in progress, with full delivery still ahead.

* **Ecosystem Validation:** FluidTokens has committed to utilizing Midgard, indicating ecosystem confidence, though production readiness remains a future milestone.

**Assessment**

Emerging L2 solutions like Midgard represent promising scaling options with potentially simpler integration paths than Hydra (due to isomorphic compatibility). However, these projects are still in active development. For the current project timeline, they do not represent production-ready alternatives. They warrant monitoring as the ecosystem matures.

## 4 Recommendation

**Primary Recommendation:** Build on Cardano L1 as the production backbone, with scaling solutions positioned as optional expansion paths to adopt when concrete requirements demand them.

**Approach:** Build on L1 now because it offers the most stable and well-understood delivery path, while architecting Smart Contracts and off-chain code for optional scaling so that Hydra, Midgard, or Partner Chain integration remains feasible if future requirements justify the investment. This avoids premature complexity while preserving routes to future scaling.

**Supporting Rationale:**

|                            | Cardano L1 | Hydra | Emerging L2s (Midgard) | Partner Chains |
|----------------------------|------------|-------|------------------------|----------------|
| **Production readiness**   | Mature | Evolving (known issues documented) | In development (milestone 2/6) | Pre-mainnet (Midnight in phased rollout) |
| **Operational complexity** | Minimal | Moderate (new infra, lifecycle management) | TBD (promising isomorphic design) | High (full network operations) |
| **Time to delivery**       | Fastest | Additional integration work | Not yet production-ready | Significant scope expansion |
| **Best for**               | Default choice; proven, low-risk path | High-frequency, low-latency interactions | Future option once production-ready | Sovereign execution with custom requirements |


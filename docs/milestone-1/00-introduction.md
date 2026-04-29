# Milestone 1 – Research, Design, and Requirements Validation

## 1 Executive Summary

This document presents the outcomes of Milestone 1: Research, Design, and Requirements Validation for the Iagon & Würth Catalyst Fund 13 project. The purpose of this milestone is to establish a validated technical and architectural foundation for a blockchain-enabled system that supports secure licensing of 3D-printable intellectual property (IP) and automated royalty settlement, while remaining compatible with Würth’s existing enterprise systems and operational constraints.

The project originated from Würth’s need to modernize and automate royalty handling for digitally manufactured parts distributed through its ORSY Connect ecosystem. While Würth already operates a trusted, production-grade solution for managing digital inventory and print execution, royalty settlement and transparency remain largely manual and off-chain, requiring significant operational effort and vendor trust. The collaboration with Iagon explores how blockchain-based primitives can be selectively introduced to improve auditability, settlement reliability, and long-term scalability—without disrupting established workflows or exposing sensitive commercial or IP data.

Milestone 1 focuses on design validation rather than implementation. Through a series of technical workshops, architecture reviews, and iterative design discussions between Iagon and Würth stakeholders, the teams evaluated multiple approaches to licensing, royalty triggering, data visibility, and blockchain integration. These discussions led to a set of confirmed architectural principles, including:
- Treating print licenses (not individual print events) as the economic unit for royalty settlement.
- Representing royalties as per-license, fixed-amount values at settlement time, even when the underlying commercial agreement is percentage-based, in order to support sub-cent precision and deterministic settlement for low-cost parts.
- Keeping on-chain data minimal and non-identifying, with all readable business metadata maintained off-chain in Würth-controlled systems.
- Introducing a settlement delay window to accommodate order cancellations before royalty obligations become final and funds are released to vendors.
- Adopting an off-chain orchestration model that coordinates enterprise systems with blockchain transactions, rather than representing orders as stateful on-chain objects.

This milestone also evaluates the suitability of different blockchain deployment options within the Cardano ecosystem, including Layer 1, Partner Chains, and Layer 2 approaches, in light of cost, throughput, privacy, and enterprise governance requirements. No irreversible technical commitments are made at this stage; instead, the milestone documents the rationale behind the shortlisted approaches and the criteria used to assess them.

The deliverables of Milestone 1 are intentionally structured as four independent but complementary design artifacts:
1. A technical overview of the proposed system architecture and data flows.
2. A design outline for royalty and transaction tracking based on digital licenses.
3. An analysis of blockchain solution options within the Cardano ecosystem.
4. A data management plan addressing storage, encryption, and access control for IP-protected files.

Together, these documents form a validated design baseline that has been reviewed internally by both Iagon and Würth. They provide the foundation for subsequent milestones, where the agreed concepts will be translated into smart contracts, APIs, and testnet deployments. As this is a closed-source, enterprise-grade pilot, validation and approval are conducted through documented internal reviews rather than public releases, in accordance with Würth’s security and compliance requirements.


## 2 Project Context and Goals

### 2.1 Problem Statement

Würth operates a mature digital inventory and additive manufacturing ecosystem that enables customers to access and produce 3D-printable parts through enterprise systems such as ORSY Connect. The technical execution of digital inventory distribution and print authorization is well established and trusted by customers.

However, the handling of royalties for intellectual property owners remains largely manual and off-chain. Royalty settlement requires operational reconciliation, delayed payouts, and a high degree of trust from vendors, with limited native transparency into license issuance and settlement timing. As the number of licensed parts and participating vendors increases, this approach introduces scalability, auditability, and cost challenges.

Any solution addressing these issues must operate within strict enterprise constraints. Sensitive IP and commercial data cannot be exposed publicly, existing production workflows must remain unchanged, and new components must integrate with Würth’s current systems rather than replace them.

### 2.2 Project Goals

The goal of this project is to design a system that enables secure and auditable royalty settlement for digital manufacturing licenses while remaining compatible with Würth’s enterprise environment and operational practices.

The project aims to:
- Establish a license-based economic model where royalties are triggered by the issuance of a digital right to print.
- Support deterministic royalty settlement with sub-cent precision for low-cost parts.
- Improve auditability and settlement transparency without exposing sensitive business or IP data.
- Use blockchain technology selectively, limiting on-chain data to non-identifying, verifiable records.
- Define an architecture that can scale across vendors, regions, and future integrations without fundamental redesign.

### 2.3 Success Criteria for Milestone 1

Milestone 1 is focused on research, design, and validation rather than implementation. It is considered successful when there is a shared and documented understanding of the proposed system design.

Success criteria include:
- Agreement on a high-level technical architecture and system boundaries.
- Validation of the conceptual licensing, royalty calculation, and settlement model.
- Documented evaluation of suitable blockchain deployment options.
- A defined data management approach covering storage, encryption, and access control.
- Internal review and approval of the design artifacts by both Iagon and Würth stakeholders.

No production systems, smart contracts, or live integrations are delivered as part of this milestone. All outputs serve as the design baseline for subsequent implementation-focused milestones.

## 3 Stakeholders and Roles

### 3.1 Würth / Würth Additive Group

Würth and Würth Additive Group act as the primary enterprise stakeholders and operators of the existing digital inventory and additive manufacturing ecosystem. They provide the commercial, operational, and technical context in which the proposed solution must function.

Within this project, their role is to define business requirements, validate architectural decisions, and ensure alignment with internal systems such as ORSY Connect and related backend services. They also set constraints related to security, compliance, data handling, and operational continuity. Würth is responsible for internal review and approval of the design artifacts produced during Milestone 1.

### 3.2 Iagon

Iagon is responsible for the research, architectural design, and technical validation of the proposed blockchain-enabled solution.

During Milestone 1, Iagon’s role is limited to design and analysis activities. This includes proposing system architectures, evaluating blockchain deployment options, defining orchestration and data flow models, and documenting trade-offs. Iagon incorporates feedback from Würth stakeholders and iterates on the design but does not deploy production systems or process live data at this stage.

### 3.3 Vendors / IP Owners

Vendors and IP owners are the parties that supply 3D-printable intellectual property and are entitled to royalties derived from licensed print rights.

Their role in Milestone 1 is to validate that the proposed licensing and royalty model reflects commercial realities, supports precise settlement including sub-cent values, and provides sufficient transparency and auditability. Vendors do not interact directly with blockchain infrastructure in this milestone, but their requirements influence the design decisions documented.

### 3.4 Customers

Customers are the end users who purchase digital licenses to print parts through Würth’s existing channels.

In Milestone 1, customers are represented indirectly through Würth’s requirements and operational constraints. The design assumes that customer-facing workflows remain unchanged, with blockchain-related functionality operating transparently in the background. Customer experience, interfaces, and print execution are explicitly out of scope for this milestone.


## 4 Milestone 1 Scope

### 4.1 In Scope

Milestone 1 focuses on research, architectural design, and requirements validation activities required to establish a shared technical foundation for the project.

The following items are in scope:
- Definition of a high-level system architecture and component responsibilities.
- Design of a license-based model for royalty calculation and settlement.
- Evaluation of blockchain deployment options within the Cardano ecosystem.
- Design of data storage, encryption, and access control approaches for IP-protected assets.
- Identification of integration boundaries with existing Würth systems.
- Documentation of assumptions, constraints, and key design decisions.
- Internal review and validation of design artifacts by Iagon and Würth stakeholders.

All in-scope activities result in documentation, diagrams, and decision records rather than executable systems.

### 4.2 Out of Scope

Milestone 1 explicitly excludes implementation, deployment, and operational activities.

The following items are out of scope:
- Development or deployment of smart contracts.
- Implementation of APIs, services, or user interfaces.
- Integration with production ORSY Connect, DIS, or printer systems.
- Processing of live customer orders or royalty settlements.
- Performance testing, load testing, or operational monitoring.
- External audits or third-party validations.

These items are intentionally deferred to subsequent milestones.

### 4.3 Assumptions and Constraints

The work performed in this milestone is based on agreed assumptions and enterprise constraints.

Key assumptions include:
- Existing commercial royalty agreements may be percentage-based, while settlement values must be representable as fixed amounts with sub-cent precision.
- Customer-facing workflows and ordering processes remain unchanged during early project phases.
- Würth continues to act as the primary operational interface for customers and vendors.

Key constraints include:
- Sensitive IP and commercial data must not be exposed on-chain.
- Existing Würth production systems must not be disrupted or replaced.
- The project operates as a closed-source, enterprise-grade pilot with internal validation and approval processes.

Several core design decisions, including the use of license issuance as the economic trigger for royalties, emerged during the course of Milestone 1 discussions and are documented as validated outcomes rather than initial assumptions.

## 5 Milestone 1 Outputs Overview

Milestone 1 is structured into four distinct but related outputs. Each output is documented separately to allow focused review and independent validation while maintaining a coherent overall design.

### 5.1 Part 1 – Technical Overview

This output documents the proposed high-level system architecture and technical design. It describes the core components, actors, and responsibilities, as well as conceptual data flows and security boundaries.

The Technical Overview establishes how enterprise systems, off-chain services, decentralized storage, and blockchain components interact at a design level. It serves as the architectural foundation for subsequent milestones that introduce smart contracts, APIs, and testnet deployments.

### 5.2 Part 2 – Royalty and Transaction Tracking

This output defines the conceptual model for tracking digital licenses, associated transactions, and royalty settlement. It documents how royalties are calculated, represented with sub-cent precision, and settled over time, independent of specific blockchain or implementation details.

The focus of this section is on economic logic and lifecycle modeling rather than execution. It provides the basis for implementing royalty logic in code in later milestones.

### 5.3 Part 3 – Blockchain Solution Selection

This output evaluates potential blockchain deployment options within the Cardano ecosystem. It compares alternatives such as Layer 1, Partner Chains, and Layer 2 approaches against criteria including cost, scalability, privacy, and enterprise governance.

The outcome of this section is a documented rationale for the selected approach, informed by the technical and business requirements identified during Milestone 1.

### 5.4 Part 4 – Data Management Plan

This output describes the approach for storing, encrypting, and controlling access to IP-protected assets and related metadata. It addresses the separation of off-chain data, encryption responsibilities, and access workflows.

The Data Management Plan ensures that the proposed architecture meets enterprise security, confidentiality, and compliance requirements and can be reviewed independently of blockchain and royalty design decisions.

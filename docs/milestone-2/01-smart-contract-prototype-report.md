# Milestone 2 – Part 1: Smart Contract Prototype Report

## 1 Purpose and Scope

### 1.1 Purpose of this report

This document is one of the four output reports for Milestone 2 of the Iagon & Würth Catalyst Fund 13 project. It describes the smart contract prototype delivered for this milestone, which translates the licensing and royalty design validated in Milestone 1 into a working set of on-chain contracts.

In particular, this report addresses the Milestone 2 *Smart Contract Prototype* output and its associated acceptance criteria, which call for:

- a **functional mock contract** simulating **token issuance**, **access control logic**, and **royalty distribution**,
- behaviour that can serve as the foundation for testnet deployment in Milestone 4 and beyond,
- simulation of the key transaction logic outlined in the original proposal.

The report describes the prototype at the level of detail used in the Milestone 1 deliverables — sufficient to validate scope, responsibilities, and behaviour without disclosing internal implementation specifics that fall under the closed-source enterprise pilot agreement.

### 1.2 Scope boundaries

In scope for this document:

- the four contracts that make up the prototype, their responsibilities, and how they relate to one another,
- how the prototype simulates token issuance, access control, and royalty distribution,
- the security properties relied on by the off-chain orchestration layer,
- a recorded happy-path output of the prototype on Cardano preprod, intended as evidence that the prototype is functionally complete and ready to be exercised by subsequent milestones,

Out of scope for this document:

- complete validator-level source listings or datum/redeemer field layouts (kept within the closed-source repository for the partner pilot),
- specialised administrative flows such as bucket removal, vault removal, or backup recovery, which are operational concerns documented separately in internal materials,
- off-chain API behaviour, royalty calculation logic, and end-to-end ordering flows, all of which are covered in the companion Milestone 2 reports.

### 1.3 Relationship to other Milestone 2 deliverables

This report is the on-chain anchor for the other Milestone 2 outputs:

- **Part 2 — Royalty Mechanism Test Report** describes how royalty amounts are calculated, batched, and settled off-chain, and how those off-chain decisions drive the on-chain primitives described here.
- **Part 3 — Off-chain API Prototype Report** describes the services and ORSY integration that submit transactions against this prototype.
- **Part 4 — API Endpoint Specification Document** documents the testnet-facing endpoints (including the chain-simulator endpoints) that exercise the prototype.

## 2 System Overview

### 2.1 What the prototype delivers

The prototype consists of four interrelated Cardano smart contracts, written in Aiken (Plutus V3), that together implement a royalty distribution system using a stablecoin. The system separates three concerns:

1. **System identity and configuration** — a one-shot initialisation flow that uniquely identifies the deployed system and a single, authoritative configuration record that all other contracts read from.
2. **Custody of funds** — a sharded treasury that holds stablecoin reserves and releases them only under verifiable conditions.
3. **Per-participant royalty accounting** — per-participant containers that accumulate royalty credits as on-chain tokens and convert them to stablecoin payouts on withdrawal.

The same prototype is used both for the deterministic mock demonstrations described in the Off-chain API Prototype Report (Part 3) and for live execution on Cardano preprod. The on-chain redeemers, datum shapes, and validation rules are identical in both cases; only the execution environment differs.

### 2.2 Control model

The prototype distinguishes two on-chain roles:

- **Keepers** — a group of administrators with multisig authority over system-level decisions (initialisation, configuration updates, treasury changes). Per-container custody (per-bucket keepers) may differ from system keepers, enabling flexible custody arrangements between operator, maintainer, and vendor participants.
- **Operator** — a single key used for day-to-day royalty operations, with strictly limited authority. The operator can issue royalty credits to participants in batches, but cannot change configuration, move treasury funds outside of the documented payout path, or take funds from participants.

This separation is intentional: it minimises the exposure of administrative keys to day-to-day operations and reduces the blast radius of a compromised operator key.

### 2.3 Reference datum pattern

A central design choice is the use of a single on-chain **reference datum** that holds the entire system's configuration. The other contracts read this datum to determine which keepers, operator, contract hashes, and stablecoin parameters are authoritative.

This pattern, internally referred to as the *paratrooper solution*, has each contract validate the one in front of it during a multi-contract transaction. It resolves circular dependencies between contracts that would otherwise need to know each other's addresses at compile time, and it provides a single source of truth for upgrades and audits.

## 3 Contract Responsibilities

The four contracts are summarised below.

### 3.1 Genesis contract

The Genesis contract is a one-shot minting policy that bootstraps the system. It is parameterised by a specific transaction output that must be spent in the same transaction, which mathematically guarantees that the genesis token can be minted only once.

When executed, the Genesis contract:

- mints exactly one identifying token whose name is derived from the parameterised output,
- sends that token to the Reference contract together with a properly structured configuration datum,
- requires a valid keeper multisig.

The Genesis contract has no on-chain datum of its own. Its sole purpose is to produce a unique, verifiable system identity that the rest of the system anchors against.

### 3.2 Reference contract

The Reference contract holds a single UTxO containing the system's configuration datum — including the keeper set and threshold, the operator key, the contract hashes for the bucket and vault contracts, the operator and maintainer bucket identifiers, and the stablecoin asset identity.

Two actions are supported:

| Action | Authority required | Notes |
|---|---|---|
| Update configuration datum | Old keepers AND new keepers (multisig) | The reference script hash cannot change (self-referential integrity); the genesis token must remain at the reference contract address |
| Backup recovery | Keepers (multisig) | Recovery path if the configuration UTxO becomes malformed |

The Reference contract is the trust root for all on-chain configuration decisions.

### 3.3 Vault contract

The Vault contract is the treasury that holds the stablecoin reserves out of which royalties are paid. The contract supports **natural sharding**: multiple vault UTxOs may exist simultaneously, each identified by a unique NFT minted by the vault contract itself. This allows the treasury to scale beyond the throughput of a single UTxO without coordination overhead.

The actions exposed by the Vault contract are:

| Action | Purpose | Authority required |
|---|---|---|
| Mint Vault Token | Create a new vault shard with a unique NFT identifier | Keepers (multisig) |
| Add Funds | Deposit additional stablecoin into an existing vault shard | Keepers (multisig) |
| Use (Payout) | Release stablecoin to a withdrawing bucket | Triggered alongside a bucket withdrawal, in the same transaction |
| Remove Vault | Retire a vault shard | Keepers (multisig) |
| Backup Recovery | Recover funds from a malformed vault UTxO | Keepers (multisig) |

The **Use** action is the only path by which stablecoin leaves the treasury, and it can be exercised only as part of a transaction that simultaneously burns the corresponding IOU tokens from a participant's bucket.

### 3.4 Bucket contract

The Bucket contract provides per-participant containers in which royalty credits accumulate. Each bucket UTxO is identified by a unique NFT (minted by the bucket contract) and is governed by its own keepers — which may be different from the system keepers, enabling separate custody for operator, maintainer, and individual vendor buckets.

The actions exposed by the Bucket contract are:

| Action | Purpose | Authority required |
|---|---|---|
| Mint Bucket Token | Create a new bucket for a participant | Bucket keepers (multisig) |
| Mint IOU Tokens | Distribute royalty credits ("IOU tokens") to one or more buckets in a single transaction | Operator (single signature) |
| Withdraw | Burn accumulated IOU tokens and trigger a vault payout for the corresponding stablecoin amount | Bucket keepers (multisig) |
| Remove Bucket | Retire an empty bucket | Bucket keepers (multisig) |
| Backup Recovery | Recover funds from a malformed bucket UTxO | System keepers (multisig) |

The **Mint IOU Tokens** action is intentionally batch-oriented: the operator can credit many buckets in a single transaction by passing a list of bucket-and-amount pairs.

## 4 Mapping to Milestone 2 Themes

The Milestone 2 Smart Contract Prototype output names three themes that the prototype must simulate: **token issuance**, **access control logic**, and **royalty distribution**. This section maps each theme onto the contracts described above.

### 4.1 Token issuance

The prototype performs three distinct kinds of token issuance, each with a different policy and lifecycle:

- **System identity issuance.** The Genesis contract mints a one-of-a-kind identifying token at system initialisation. This token never circulates and serves as a permanent on-chain anchor for the system's identity.
- **Container identity issuance.** Both the Vault and Bucket contracts mint unique NFTs as identifiers for their UTxOs. These tokens enable natural sharding (for vaults) and per-participant containers (for buckets) without requiring an external registry.
- **Royalty credit issuance.** The Bucket contract mints **IOU tokens** representing accrued royalties. These tokens are the on-chain representation of a participant's claim against the treasury and are the unit of account for royalty distribution.

The IOU token lifecycle is the most operationally significant of the three: tokens are minted in batch by the operator, accumulate in participant buckets, and are burned when the participant withdraws, releasing an equal amount of stablecoin from the treasury.

### 4.2 Access control logic

Access control in the prototype is enforced on-chain through three mechanisms:

- **Keeper multisig.** All system-level actions (configuration updates, vault creation, vault funding, vault removal, backup recovery) require a verifiable multisig from the keeper set defined in the reference datum, meeting or exceeding the configured threshold.
- **Per-container custody.** Each bucket has its own keeper set and threshold. Withdrawals from a bucket are gated by that bucket's keepers, not the system keepers. This allows operator, maintainer, and vendor buckets to be governed independently.
- **Operator authority scoping.** The operator key can mint IOU tokens to buckets (the day-to-day royalty operation) but has no authority over configuration, treasury composition, or withdrawals. This narrowly scopes the most-frequently-used key to the least-sensitive action.

In addition, the prototype includes a **graceful degradation** path on every contract so that malformed UTxOs cannot permanently lock funds (see §5).

### 4.3 Royalty distribution

Royalty distribution in the prototype follows a two-phase model that matches the off-chain orchestration described in the Royalty Mechanism Test Report:

**Phase 1 — Issuance (operator-driven, batched).** The operator constructs a single transaction that mints IOU tokens to a list of buckets, with one bucket-and-amount pair per recipient. The on-chain validation ensures that:

- the operator signature is present,
- each receiving bucket appears both as an input and as an output of the transaction,
- the bucket's identity is preserved across the transaction (the bucket's identifier and its keeper set are unchanged on the output side) and the output value differs from the input value by exactly the declared IOU amount,
- the amounts minted match the declared distribution list.

This batched mint is what makes per-order or per-batch settlement economically viable on Cardano, and it is the primitive that the off-chain batching strategies target.

**Phase 2 — Payout (bucket-keeper-driven, atomic).** When a participant chooses to withdraw, their bucket's keepers sign a single transaction that burns the bucket's IOU tokens and spends a vault UTxO, releasing an equal amount of stablecoin to the participant's chosen address. Burn and release are bound to one another on-chain — see §5 property 5 — eliminating an entire class of reconciliation failures that would otherwise have to be handled off-chain.

## 5 Security Properties

The prototype is built around six security properties that the off-chain layer relies on:

1. **One-shot initialisation.** The genesis token can be minted only once, because the parameterising UTxO can be spent only once. This guarantees that the system has a single, unforgeable identity.
2. **Self-referential integrity.** The Reference contract's own script hash is stored in its configuration datum and cannot be changed. Any attempt to migrate the reference UTxO to a different script address fails on-chain.
3. **Multisig administration.** All administrative actions require threshold signatures from keepers. No single key can change configuration, move treasury funds outside the documented payout path, or recover funds.
4. **Operator separation.** Day-to-day IOU distribution requires only the operator key, which has strictly limited authority. Compromise of the operator key cannot result in unauthorised payouts, configuration changes, or treasury withdrawals.
5. **Atomic withdrawal binding.** Stablecoin can leave the treasury only in a transaction that simultaneously burns the corresponding IOU tokens. This is enforced on-chain.
6. **Graceful degradation.** All contracts implement a backup-recovery path so that malformed UTxOs cannot permanently lock funds. Recovery requires the appropriate keeper multisig.

Together these properties give the off-chain orchestration layer a small, well-defined trust boundary: the orchestrator and transaction builder do not need to hold or guard treasury funds, and the operator key is intentionally limited so that its compromise cannot cause economic loss beyond the IOU credits already in circulation.

## 6 Test Coverage and Benchmarks

The prototype is validated through three complementary forms of testing:

- **Aiken unit tests**, written alongside the contracts and run inside the Aiken toolchain (not on a Cardano node), that exercise the core validation logic — IOU mint and use semantics, list utilities used in batch distribution, and signature verification. The tests construct synthetic transactions and assert that each validator accepts or rejects them according to the rule under test. They accompany the contracts in the closed-source repository.
- **Execution-unit benchmarks** of the list-traversal helper that the IOU-mint validator uses internally (`find_entry_tail` in `lib/tests/optimal_list.ak`), comparing on-chain memory and CPU consumption at varying list sizes under three step-skip strategies. The benchmark outputs (`one_step_benchmark.out`, `two_step_benchmark.out`, `four_step_benchmark.out`) accompany the contract source.
- **Happy-path execution on Cardano preprod** (Section 7), driven by a documented sequence of administrative and operational transactions, which serves as the end-to-end evidence that the prototype behaves as designed in a real-network environment.

The chain-simulator stand-in introduced in §2.1 is documented separately in the Off-chain API Prototype Report and is used for deterministic, chain-free demonstration of the same prototype.

## 7 Happy-Path Demonstration on Cardano Preprod

This section captures a single, end-to-end execution of the prototype's main flows on Cardano preprod. It is intentionally restricted to the operational happy path.

The sequence below mirrors the six-step happy path agreed with the smart contract engineering team:

1. **Genesis** — initialise the system and produce the reference datum.
2. **Vault Creation** — create the initial treasury shard.
3. **Vault Fill** — fund the treasury shard with stablecoin.
4. **Bucket Creation** — create a participant bucket.
5. **IOU Mint** — issue royalty credits to the participant's bucket (batch-capable, demonstrated with at least one recipient).
6. **Bucket Payout** — burn the participant's IOU tokens and release the corresponding stablecoin from the treasury.

For each step the table below records the on-chain transaction reference and a brief description of the observable effect. The console output for each step is included as an appendix to this report (Section 9).

### 7.1 Transaction summary

| # | Step | Preprod Transaction Hash | Observable effect |
|---|---|---|---|
| 1 | Genesis | `ae03acf520f30de662f2d432ec791e83f397f1dea8b638c0dc4693267000c90a` + `6c9ee99a7c590ef29ab5fa200519cebd4594dad59749a02d5f6f6a616c55f3be` | System identity token minted; reference UTxO at reference contract address; keepers, operator, contract hashes, stablecoin identity recorded in the configuration datum. |
| 2 | Vault Creation | `7ced769d952a77fd0087ec751fcc8902fd672951005115e13eaf479a9253dcca` | New vault UTxO with unique identifier NFT created at the vault contract address; empty stablecoin balance. |
| 3 | Vault Fill | `3c11d5e40e84a7cba6c95fb07f6523f63170a9fb12876694f8461c429924dd0f` | Stablecoin deposited into the vault UTxO; identifier NFT preserved; datum unchanged. |
| 4 | Bucket Creation | `8d9046a9ce992595455b2a29fe657162495a8343c50dad0793d4e7d0389d116e` | New bucket UTxO with unique identifier NFT created at the bucket contract address; bucket-specific keepers and threshold recorded in the bucket datum; IOU balance zero. |
| 5 | IOU Mint | `7553d538a0a00d1c6362083d1c2d262e78252993cec88631794273141c267c94` | Operator-signed transaction credits one or more bucket(s) with IOU tokens equal to the royalty amount(s) due. Bucket datum unchanged; only the IOU token balance increases. |
| 6 | Bucket Payout | `3877ed7d4384f18810d87e81d00abd0d2e7ed9b8b52ad14243f1752a8208b574` | Bucket keepers sign a transaction that burns the bucket's IOU tokens and spends a vault UTxO; an equal amount of stablecoin is released from the treasury to the recipient address. |

### 7.2 What this demonstrates against the Milestone 2 acceptance criteria

The six-step happy path collectively demonstrates each of the themes named in the Milestone 2 *Smart Contract Prototype* acceptance criterion:

- **Token issuance** is exercised at step 1 (system identity), step 2 (vault identifier), step 4 (bucket identifier), and step 5 (royalty credits).
- **Access control logic** is exercised at step 1 (keeper multisig at initialisation), step 2 (keeper multisig for treasury creation), step 5 (operator signature scope for IOU mint), and step 6 (per-bucket keeper multisig for withdrawal).
- **Royalty distribution** is exercised end-to-end across steps 5 and 6 (issuance and atomic payout), which together exhibit the two-phase distribution model described in Section 4.3.

## 8 Appendix — Happy-Path Console Output

### 8.1 Step 1 — Genesis

```
> ./01b_witnessGenesisMintTx.sh 
 Operator Witness 
 Collat Witness 
 Genesis Witness 
 Assembling 
 Submitting 
Transaction successfully submitted. Transaction hash is:
{"txhash":"ae03acf520f30de662f2d432ec791e83f397f1dea8b638c0dc4693267000c90a"}

> ./02b_witnessUpdateReferenceDatumTx.sh 
 Operator Witness 
 Vendor Witness 
 Collat Witness 
 Assembling 
 Submitting 
Transaction successfully submitted. Transaction hash is:
{"txhash":"6c9ee99a7c590ef29ab5fa200519cebd4594dad59749a02d5f6f6a616c55f3be"}
```

### 8.2 Step 2 — Vault Creation

```
> ./01_createVault.sh 
 Gathering Script UTxO Information  
Data UTxO: 6c9ee99a7c590ef29ab5fa200519cebd4594dad59749a02d5f6f6a616c55f3be#0
 Gathering Operator UTxO Information  
Operator UTxO: 1c0f090849cbbeb40fbd90dda56544265b3922e99c6bd546428d6f4935a40323#0

Token Name: 1 3a55eee4fc0d482e4ba3a2f00c8a299a8ed6c71221a80108765f2f54.001c0f090849cbbeb40fbd90dda56544265b3922e99c6bd546428d6f4935a403 
 Gathering Collateral UTxO Information  
 Building Tx 
Estimated transaction fee: 341814 Lovelace 
 Signing 
 Submitting 
Transaction successfully submitted. Transaction hash is:
{"txhash":"7ced769d952a77fd0087ec751fcc8902fd672951005115e13eaf479a9253dcca"}
```

### 8.3 Step 3 — Vault Fill

```
> ./03_addToVault.sh 
 Gathering Script UTxO Information  
Data UTxO: 6c9ee99a7c590ef29ab5fa200519cebd4594dad59749a02d5f6f6a616c55f3be#0
 Gathering Script UTxO Information  
Vault UTxO: 7ced769d952a77fd0087ec751fcc8902fd672951005115e13eaf479a9253dcca#0
 Gathering Operator UTxO Information  
Operator UTxO: 7ced769d952a77fd0087ec751fcc8902fd672951005115e13eaf479a9253dcca#1
Total Stable: 38253 c65cb5e0a28be0fc30cef5c53f55bc665740062e1e24f65b7d310d21.74537461626c65
 Gathering Collateral UTxO Information  
 Building Tx 
Estimated transaction fee: 343402 Lovelace 
 Signing 
 Submitting 
Transaction successfully submitted. Transaction hash is:
{"txhash":"3c11d5e40e84a7cba6c95fb07f6523f63170a9fb12876694f8461c429924dd0f"}
```

### 8.4 Step 4 — Bucket Creation

```
> ./01_createBucket.sh 
 Gathering Reference UTxO Information  
Reference UTxO: 6c9ee99a7c590ef29ab5fa200519cebd4594dad59749a02d5f6f6a616c55f3be#0
 Gathering Vendor UTxO Information  
Vendor UTxO: 6c9ee99a7c590ef29ab5fa200519cebd4594dad59749a02d5f6f6a616c55f3be#1

Token Name: 1 d3a7447ce5831ec873bc1c8d9e61200c6a1b3e2cc1e05c544ea027df.016c9ee99a7c590ef29ab5fa200519cebd4594dad59749a02d5f6f6a616c55f3 
 Gathering Collateral UTxO Information  
 Building Tx 
Estimated transaction fee: 402204 Lovelace 
 Signing 
 Submitting 
Transaction successfully submitted. Transaction hash is:
{"txhash":"8d9046a9ce992595455b2a29fe657162495a8343c50dad0793d4e7d0389d116e"}
```

### 8.5 Step 5 — IOU Mint

```
> ./03_addToBucket.sh 
 Gathering Script UTxO Information  
Data UTxO: 6c9ee99a7c590ef29ab5fa200519cebd4594dad59749a02d5f6f6a616c55f3be#0
 Gathering Script UTxO Information  
Bucket UTxO: 8d9046a9ce992595455b2a29fe657162495a8343c50dad0793d4e7d0389d116e#0
 Gathering Operator UTxO Information  
Operator UTxO: 3c11d5e40e84a7cba6c95fb07f6523f63170a9fb12876694f8461c429924dd0f#1
Total IOU: 123 d3a7447ce5831ec873bc1c8d9e61200c6a1b3e2cc1e05c544ea027df.494f55
 Gathering Collateral UTxO Information  
Bucket sorted input offset: 1
 Building Tx 
Estimated transaction fee: 434557 Lovelace 
 Signing 
 Submitting 
Transaction successfully submitted. Transaction hash is:
{"txhash":"7553d538a0a00d1c6362083d1c2d262e78252993cec88631794273141c267c94"}
```

### 8.6 Step 6 — Bucket Payout

```
> ./04_withdrawBucket.sh 
 Gathering Script UTxO Information  
Data UTxO: 6c9ee99a7c590ef29ab5fa200519cebd4594dad59749a02d5f6f6a616c55f3be#0
 Gathering Script UTxO Information  
Vault UTxO: 3c11d5e40e84a7cba6c95fb07f6523f63170a9fb12876694f8461c429924dd0f#0
 Gathering Script UTxO Information  
Bucket UTxO: 7553d538a0a00d1c6362083d1c2d262e78252993cec88631794273141c267c94#1
 Gathering Operator UTxO Information  
Operator UTxO: 44d354d9be4da7b3887bbd91e36f34079563ecaab93dcd1a69a751d77161e652#1
Total Stable: 38130 c65cb5e0a28be0fc30cef5c53f55bc665740062e1e24f65b7d310d21.74537461626c65
 Gathering Collateral UTxO Information  
 Building Tx 
Estimated transaction fee: 560806 Lovelace 
 Signing 
 Submitting 
Transaction successfully submitted. Transaction hash is:
{"txhash":"3877ed7d4384f18810d87e81d00abd0d2e7ed9b8b52ad14243f1752a8208b574"}
```

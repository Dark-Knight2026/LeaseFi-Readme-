# LeaseFi

> **Decentralized rental property management on Casper Network.**
> A smart-contract protocol for residential and commercial leases — payments, escrow, disputes, and property management — settled on-chain with a universal 2% protocol fee.

[![Network](https://img.shields.io/badge/network-Casper%202.2-blue)](https://casper.network)
[![Status](https://img.shields.io/badge/status-active%20development-brightgreen)]()
[![Audit](https://img.shields.io/badge/audit-pending-yellow)]()
[![Phase 1 Mainnet](https://img.shields.io/badge/Phase%201%20Mainnet-Q4%202026-informational)]()
[![License](https://img.shields.io/badge/license-TBD-lightgrey)]()

---

## Table of Contents

- [Overview](#overview)
- [Why Casper](#why-casper)
- [Technical Architecture](#technical-architecture)
- [Casper Integration](#casper-integration)
- [Tokenomics — BIG](#tokenomics--big)
- [Roadmap](#roadmap)
- [Repository Structure](#repository-structure)
- [Documentation](#documentation)
- [Development](#development)
- [Security](#security)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

LeaseFi replaces the fragmented stack of paper leases, manual rent collection, opaque escrow, and discretionary dispute resolution that defines traditional rental property management with a single on-chain protocol. Every meaningful event in the rental lifecycle — lease creation, rent payment, deposit funding, deduction filing, dispute resolution, deposit release, lease termination — is automatic, auditable transaction on Casper Network.

The protocol serves five user roles in Phase 1:

- **Tenants** — pay rent in CSPR, BIG, USDC, or PYUSD; fund deposits to on-chain escrow; file disputes
- **Landlords / Property Owners** — create leases, collect rent, manage deposits, delegate to PMs
- **Property Managers** — bonded registry, fee-share authorization, multi-property management
- **Compliance Officer / General Counsel** — registry oversight, dispute arbitration designation, OFAC/KYC enforcement
- **Auditors / Regulators** — read-only access to a public, immutable lifecycle audit trail

Additional roles activate in Phases 2-5 (Vendors, Tax Preparers, DST/1031 facilitators, and AI agents).

**Universal 2% fee.** Every value-transferring entry point pays the same 2% protocol fee — no carve-outs, no exempt paths, no special cases. Fees accrue to the FeeDistributor for BIG buyback (Phase 2+) and DAO Treasury operations.

**Regulatory posture.** BIG is offered initially via Regulation S (non-U.S.) and Regulation D 506(c) (U.S. accredited investor) tranches, structured against the CLARITY Act mature-blockchain test. KYC tiers, OFAC screening, and velocity limits are enforced at the protocol boundary. Phase 1 contracts are **immutable post-audit** — no admin keys, no pause function, no upgrade path — to satisfy the mature-blockchain test.

---

## Why Casper

LeaseFi is **Casper-native**, not Casper-deployed. The protocol design depends on Casper primitives that aren't directly available on EVM chains:

| Casper feature | Why LeaseFi needs it |
|---|---|
| **Account-based model with associated keys** | Treasury multi-sig and PM key-rotation use Casper's native multi-key account model — no Gnosis Safe-style external multisig contract required |
| **Predictable finality (~32s)** | Rent payments and deposit releases need atomic, irreversible settlement; finality without reorg risk simplifies indexer state management |
| **Native gas accounting in CSPR** | Users can pay protocol fees and gas in the same native asset; no wrapped-ETH dance |
| **`casper-event-standard`** | Structured event emission with deserialization guarantees — off-chain indexer correctness is provable rather than best-effort |
| **Block time in ms-since-epoch** | `runtime::get_block_time()` returns milliseconds, enabling sub-second-precision lease term arithmetic without block-count drift (Casper 2.2 block time is ~16s; block-counting drifts measurably over multi-year leases) |
| **Lower validator capture surface** | Casper's Proof-of-Stake with stake-weighted finality is well-suited to a regulated, jurisdiction-aware protocol |

LeaseFi also depends on two Casper ecosystem services:

- **[CSPR.click](https://csprclick.io/)** — wallet abstraction for non-crypto-native users (most landlords and tenants are not Web3-native)
- **[CSPR.name](https://cspr.name/)** — human-readable address resolution (`alice.cspr` instead of `account-hash-…`)

---

## Technical Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                         USER LAYER                              │
│   Web dApp · Mobile dApp · CSPR.click wallet · CSPR.name        │
└──────────────────────────┬─────────────────────────────────────┘
                           │ EIP-712 typed-data signatures
                           │ (used as a signing schema standard,
                           │  verified against secp256k1/Ed25519
                           │  via Casper account public keys)
┌──────────────────────────▼─────────────────────────────────────┐
│              SMART CONTRACTS (Casper 2.2 mainnet)               │
│                                                                 │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐        │
│   │ LeaseFactory │──▶│    Lease     │──▶│ DisputeModule│        │
│   │ (deployer)   │   │ (per-lease)  │   │ (arbitration)│        │
│   └──────────────┘   └──────┬───────┘   └──────────────┘        │
│                             │                                   │
│                             ▼                                   │
│   ┌──────────────┐   ┌──────────────────────┐   ┌────────────┐  │
│   │  PMRegistry  │──▶│    PaymentRouter     │──▶│    BIG     │  │
│   │ (bonded PMs) │   │  universal 2% fee    │   │  (CEP-18)  │  │
│   └──────────────┘   │  velocity-limited    │   └────────────┘  │
│                      └──────────┬───────────┘                   │
│                                 │                               │
│                                 ▼                               │
│                       ┌──────────────────┐                      │
│                       │  FeeDistributor  │                      │
│                       │  + Treasury      │                      │
│                       │  (multi-sig)     │                      │
│                       └──────────────────┘                      │
└──────────────────────────┬─────────────────────────────────────┘
                           │ casper-event-standard emit
┌──────────────────────────▼─────────────────────────────────────┐
│                   OFF-CHAIN INFRASTRUCTURE                      │
│   Event Indexer · GraphQL API · Notification Service            │
│   Postgres · Redis · KYC/AML provider · Fraud Detection        │
└────────────────────────────────────────────────────────────────┘
```

### Tech Stack

**On-chain:**
- **Language:** Rust → WASM via `casper-contract` SDK
- **Casper crates:** `casper-types`, `casper-contract`, `casper-event-standard`
- **Shared types:** internal `leasefi-types` workspace crate (errors, events, EIP-712 schemas, constants, `FeeSource` enum, `CallerType` enum, `compute_fee` helper)
- **Currencies:** CSPR (native, 9 decimals), BIG (CEP-18, 18 decimals), USDC (6 decimals), PYUSD (6 decimals)

**Off-chain:**
- **Indexer:** TypeScript / Node.js consuming `casper-event-standard` serialized events
- **API:** GraphQL (Apollo Server)
- **Database:** Postgres for indexed state, Redis for pub/sub and caching
- **Notifications:** Email + push via service workers
- **Type sharing:** `@leasefi/types` npm package codegenerated from the Rust `leasefi-types` crate via `syn` parsing — guarantees TS wire format matches Rust byte-for-byte

**Identity & Compliance:**
- **KYC:** tiered verification by a third-party identity provider; tier unlocks per the product requirements
- **OFAC:** all addresses screened against sanctions lists with risk-based enhanced review at compliance-defined thresholds
- **Velocity limits:** enforced per payer per `FeeSource`; thresholds configurable by governance
- **Signing:** EIP-712 typed-data schemas verified against Casper account public keys (the EIP-712 standard is used as a structured signing format — the actual signature math is secp256k1 or Ed25519, native to Casper)

---

## Casper Integration

This section is for engineers and auditors familiar with Casper.

### Contract Storage

Each contract uses Casper's **named keys** for top-level state references and **dictionary-based storage** for collections. Storage key naming follows the v1.4.4 ms-suffix convention: all timestamp-bearing keys use `_time_ms` or `_ms` suffixes (e.g., `lease_start_time_ms`, `resolution_deadline_ms`, `rent_paid_through_time_ms`). Block-counting is forbidden in business logic — all duration math operates on `runtime::get_block_time()` which returns milliseconds since Unix epoch.

### Cross-Contract Calls

Most cross-contract calls are **same-package contract-to-contract calls** using `runtime::call_contract` with the destination contract's package hash. The `PaymentRouter` registers authorized callers via `CallerType` enum (LeaseFactory, Lease, PMRegistry, DisputeModule, etc.) — every fee-bearing entry point routes through PaymentRouter, which validates the caller's registration before transferring the protocol fee.

### Events

LeaseFi emits 26 canonical event types via `casper-event-standard`:

```rust
// Example: rent payment event (defined in leasefi-types::events)
#[derive(Event, CLTyped, ToBytes, FromBytes)]
pub struct RentRecordedEvent {
    pub lease_id: u64,
    pub payer: Address,
    pub gross_amount: U256,
    pub fee_amount: U256,
    pub net_amount: U256,
    pub currency: Currency,
    pub paid_through_time_ms: u64,
    pub timestamp: u64,
}
```

Event field naming matches the off-chain indexer's column names exactly — no mapping layer required.

### Signing & EIP-712

LeaseFi uses EIP-712 typed-data structures as a **signing schema standard**, not as an Ethereum dependency. The schemas (LeaseAgreement, TerminationNotice, Buyout, ReleaseAuthorization, PMAuthorization) are defined once in `leasefi-types::eip712` and verified on-chain via Casper's native signature verification primitives. This gives the protocol:

- **Audit-friendly typed data** — auditors can verify schema integrity via golden-file comparison (`audit/eip712-golden/lease_v1_4_4.json`)
- **Cross-platform tooling** — any wallet that supports EIP-712 (most do) can sign LeaseFi messages without custom integration
- **Frontend type safety** — `@leasefi/types` exports TypeScript interfaces for every schema, codegen'd from Rust source

### Account Model

LeaseFi uses Casper's **associated keys** for multi-sig and key rotation:

- **Treasury** — multi-sig via associated keys with weight thresholds
- **Property Managers** — bond-funded accounts with rotation support (PM key compromise can be remediated via `deregister_caller` without affecting the underlying bond)

This avoids the contract-based multisig pattern common on EVM chains. Lower attack surface, native to the platform.

### Casper Manifest Dependencies

**Phase 1 has zero dependencies on Casper Manifest features** (EVM Execution Engine, Native Token Registry, gasless transactions). This was an explicit architectural decision (resolved as Critical #4 in Whitepaper v4.4.2) to remove a Phase 1 ship-blocker. Manifest features are scoped to Phases 2-4 as Casper delivers them. The protocol ships on **Casper 2.2** with the feature set available today.

---

## Tokenomics — BIG

**BIG** is the protocol utility token. CEP-18 standard, 18 decimals, fixed maximum supply of **5,000,000,000 BIG**.

### Distribution (v4.5.5)

| **TOTAL** | **100%** | 5,000,000,000 BIG fixed max supply |

### Token Utility

- **Protocol fees** — 2% universal fee can be paid in CSPR, BIG, USDC, or PYUSD; BIG payment receives a fee discount (rate set by governance)
- **PM bonding** — Property Managers post BIG bonds against PMRegistry; bond amount is governance-tunable
- **Buyback & accrual** — Phase 2+ FeeDistributor swaps collected fees to BIG and accrues to DAO Treasury
- **Governance** — Phase 2+ BIG-weighted voting on protocol parameters (fee discounts, PM bond minimums, velocity limit defaults)
- **Usage mining rewards** — active rental activity earns BIG distributions per the Usage Mining schedule

See [Whitepaper v4.5.5](docs/BIG_Whitepaper_v4_5_5.md) for the full token economy.

---

## Roadmap

### Phase 0 — Capital Formation (2026 Q3)
- BIG offering: Regulation S (non-U.S.) + Regulation D 506(c) (U.S. accredited)
- KYC infrastructure, subscription documents, securities counsel sign-off
- Treasury formation, custodian relationships

### Phase 1 — Mainnet Launch (2026 Q4) — **In active development**
- LeaseFactory, Lease, PMRegistry, DisputeModule, PaymentRouter, FeeDistributor, BIG (CEP-18)
- 5 user roles (Tenant, Landlord, PM, CO, GC)
- Centralized arbitration (CO-designated arbitrator)
- Treasury multi-sig with role-segregated key holders
- Web dApp + mobile dApp
- **Immutable contracts post-audit** — no admin, no pause, no upgrade


### Phase 2 — Vendor Marketplace & Decentralized Arbitration (2027)
- Vendor directory with bonded registration
- Escrowed work orders with milestone-based release
- **Decentralized arbitration** — BIG-staked arbitrators replace CO designation
- **FeeDistributor swap-and-buyback** activation
- **EVM bridge** for cross-chain BIG liquidity
- Replacement of three Phase 1 operational concessions (centralized arbitration, CO-controlled PM registry, treasury multisig)
- Governance bootstrap (BIG-weighted parameter voting)

### Phase 3 — Property-Relationship Payments (2028)
- HOA dues, utility billing (water, electricity, gas), insurance premium routing
- Per-state legal framing ("property management activity, not money transmission")
- State pilots: Florida, Texas, Tennessee
- Additional state pilots subject to counsel review

### Phase 4 — Tax & Compliance Automation (2028+)
- Schedule E export for rental property tax filing
- 1099-MISC generation for contractor payments
- Tax preparer role (read-only access with property-owner authorization)
- 1031 tax-deferred exchange support
- DST (Delaware Statutory Trust) integration via DST-LPT track

### Phase 5+ — AI Agent Platform
- 44 specialized agents covering lease drafting, dispute arbitration support, vendor matching, tax preparation, regulatory monitoring, and more
- 24 Committed / 14 Conditional / 6 Speculative agents per [Long-Range Vision v1.0](docs/LeaseFi_LongRange_Vision_v1_0.md)
- BIG-incentivized agent operators

---

## Repository Structure

> Workspace layout (Cargo + pnpm monorepo):

```
leasefi/
├── crates/                          # Rust smart contract crates (Cargo workspace)
│   ├── leasefi-types/               # Shared types, errors, events, EIP-712 schemas
│   ├── lease/                       # Lease contract (per-lease child)
│   ├── lease-factory/               # LeaseFactory deployer
│   ├── pm-registry/                 # Property manager registry + bond escrow
│   ├── dispute-module/              # Centralized arbitration (Phase 1)
│   ├── payment-router/              # Universal 2% fee dispatcher
│   ├── fee-distributor/             # FeeDistributor (Phase 1: hold; Phase 2: swap+buyback)
│   └── big-token/                   # CEP-18 BIG token
├── packages/                        # TypeScript packages (pnpm workspace)
│   ├── leasefi-types/               # TS mirror of Rust types (codegen)
│   ├── indexer/                     # Event indexer (casper-event-standard consumer)
│   ├── api/                         # GraphQL API
│   ├── notifications/               # Notification service
│   └── web/                         # Web dApp
├── scripts/                         # Deployment, codegen, ops scripts
│   └── generate-ts-types.sh         # Rust → TypeScript schema codegen
├── tests/                           # Cross-contract integration tests
├── audit/                           # Audit artifacts
│   └── eip712-golden/               # Frozen EIP-712 schema golden files
├── docs/                            # Public documents
└── .github/workflows/               # CI: tests, codegen sync, sibling-sweep
```

---

## Documentation

| Document | Purpose |
|---|---|
| [BIG Whitepaper v4.5.5](docs/BIG_Whitepaper_v4_5_5.md) | Regulatory and tokenomic framework |
| [LeaseFi PRD v2.4.4](docs/LeaseFi_PRD_v2_4_4.md) | Product requirements / functional spec |
| [SC Spec v1.4.4](docs/LeaseFi_SC_Spec_v1_4_4.md) | Smart contract interface specifications |
| [Off-Chain Architecture Spec v1.1](docs/LeaseFi_OffChain_Architecture_v1_1.md) | Indexer, API, notifications, KYC integration |
| [Long-Range Vision v1.0](docs/LeaseFi_LongRange_Vision_v1_0.md) | Phase 2-5+ AI agent and platform vision |

Document set discipline: versions advance independently. Each document's title page manifest declares which companion versions it was aligned to at issuance. Cross-document review cycles enforce consistency via a sibling-sweep convention.

---

## Development

LeaseFi has been in active development since December 2025. The codebase tracks the specification revisions described in [Documentation](#documentation); spec deltas are integrated continuously as part of the audit-prep cycle.

### Prerequisites
- **Rust** 1.78+ with `wasm32-unknown-unknown` target
- **Casper Client** (`casper-client` CLI) for contract deployment
- **Node.js** 20 LTS + **pnpm** for the TypeScript stack
- **Docker** for local Casper node, Postgres, and Redis
- **`just`** (command runner) for build orchestration

### Commands
```bash
# Build all Rust contracts to WASM
cargo build --release --target wasm32-unknown-unknown

# Run unit + integration tests
cargo test --workspace

# Run the local sibling-sweep CI gate
just sibling-sweep

# Regenerate @leasefi/types from Rust source
just generate-ts-types

# Build the TypeScript stack
pnpm install && pnpm build

# Start a local Casper development network
just local-node-up
just deploy-local
just indexer-up
just web-dev
```

Detailed development setup lives in `CONTRIBUTING.md` and `docs/development.md`.

---

## Security

### Audit Status
- **Phase 1 contract audit:** scheduled post engineering Phase A
- **Engagement:** Tier-1 smart contract audit firm; selection in progress
- **Audit RFP:** prepared and ready for distribution
- **Pre-audit gates:** internal Go/No-Go engineering gates must all pass before audit engagement

### Disclosure Policy

LeaseFi has not yet launched. Post-mainnet, the following will apply:

- **Critical vulnerabilities** (loss of funds, unauthorized state change, signature forgery): email `security@leasefi.io` (placeholder) with PGP-encrypted report. **Do not open a public GitHub issue.**
- **Non-critical issues** (informational, gas optimization, off-chain UX): public GitHub issues acceptable
- **Bug bounty:** program details published post mainnet launch

### Design Posture

- **Phase 1 contracts are immutable post-audit** — no admin keys, no pause function, no upgrade path. This is intentional and required for CLARITY Act mature-blockchain certification.
- **Velocity limits enforced at the protocol boundary** — thresholds configurable by governance in Phase 2+
- **OFAC screening at every value-transferring entry point** — risk-based enhanced review at compliance-defined thresholds
- **No upgradeability** — bugs must be fixed via the migration path defined in the smart contract specification (phase migration architecture)

---

## Contributing

> External contributions are not currently open. The repository is in audit-prep state; changes are restricted to core contributors. External contribution will open post-mainnet under the policy below.

Internal contribution posture (also the policy that will extend to external contributors post-launch):

- All PRs require **two reviewer approvals**, including at least one core contributor
- Smart contract changes require **`leasefi-types` sibling-sweep clearance** (any change affecting cross-contract types requires a paired change to every consumer in the same PR)
- All new entry points require **unit tests, integration tests, and EIP-712 golden file updates** where applicable
- Code style: `cargo fmt` for Rust, Prettier + ESLint strict for TypeScript
- Commit messages: Conventional Commits (`feat:`, `fix:`, `refactor:`, etc.)

See `CONTRIBUTING.md` for the full contribution guide.

---

## License

License terms are under counsel review. Likely outcome:
- **Smart contracts** — Business Source License 1.1 (BUSL-1.1) with conversion to MIT after a defined change date, common in DeFi
- **Off-chain code (indexer, API, notifications)** — Apache 2.0 or MIT
- **Documentation** — CC BY 4.0

Final license to be confirmed before mainnet launch.

---

## Links

- **Website:** https://leasefi.vercel.app/
- **Casper Network:** [casper.network](https://casper.network)
- **CSPR.click:** [csprclick.io](https://csprclick.io/)
- **CSPR.name:** [cspr.name](https://cspr.name/)
- **Twitter / X:** @KeyChain_BIG
- **Instagram:** @KeyChain_big

---

*This README reflects the document set as of May 2026: SC Spec v1.4.4, PRD v2.4.4, BIG Whitepaper v4.5.5, Off-Chain Spec v1.1, Long-Range Vision v1.0. Engineering in active development since December 2025; mainnet target Q4 2026.*


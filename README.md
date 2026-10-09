<div align="center">

# Lendify-smart_contracts

**Soroban smart contracts powering Lendify — reputation-based, collateral-light credit on Stellar.**

Credit, reputation, and a shared liquidity pool, enforced on-chain in Rust.

[![Contracts CI](https://github.com/Lendify-Onchain-Org/Lendify-smart_contracts/actions/workflows/contracts-ci.yml/badge.svg)](https://github.com/Lendify-Onchain-Org/Lendify-smart_contracts/actions/workflows/contracts-ci.yml)
[![Rust](https://img.shields.io/badge/Rust-stable-000000?logo=rust&logoColor=white)](https://www.rust-lang.org)
[![Soroban](https://img.shields.io/badge/Soroban-SDK-7D00FF?logo=stellar&logoColor=white)](https://soroban.stellar.org)
[![Network](https://img.shields.io/badge/network-testnet-blue.svg)](https://stellar.expert/explorer/testnet)
[![Tests](https://img.shields.io/badge/tests-420-brightgreen.svg)](#-running-tests)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](./LICENSE)

[Project Overview](#project-overview) · [Setup Instructions](#setup-instructions) · [Deployed Contracts](#deployed-contracts) · [Ecosystem Architecture](#ecosystem-architecture)

</div>

---

## Project Overview

This repository acts as the **settlement and trust layer** of the Lendify protocol. It contains all Soroban smart contracts governing the issuance of credit, repayment logic, reputation scoring, and liquidity pool management.

### The Contracts

- **CreditLine**: Orchestrates the loan lifecycle (creation, guarantee, funding, repayment, default) between the user, merchant, and liquidity pool.
- **Liquidity Pool**: Holds pooled capital from sponsors. Shares are minted on deposit; interest stays in the pool, increasing share value over time.
- **Reputation**: Tracks on-chain user scores. High scores unlock larger loan limits; defaults penalize scores.
- **Vendor Registry**: Whitelist of active merchants authorized to receive direct loan funding.
- **Parameters**: Multisig-controlled registry for global protocol settings (base interest, max loan limits, fees, score thresholds).
- **Vouching (WIP)**: Enables high-reputation users (mentors) to cryptographically stake their reputation on new learners.

### How credit works

- **No generic withdrawals**: When a loan is funded, XLM is sent **directly to the Vendor** (via the Vendor Registry), never to the borrower.
- **Collateral-light**: Borrowers deposit a `20%` guarantee up front. The liquidity pool provides the remaining `80%`.
- **Reputation-gated**: A borrower must have a score >= `MIN_SCORE` to open a loan. The loan size is capped by their current score bracket (e.g., `max_loan = score * multiplier`).
- **Late fees** accrue per overdue installment; an optional grace period is governable.
- These brackets are compile-time constants; the penalty/threshold/fee parameters and an optional base interest rate are adjustable through the **Parameters** contract's multisig governance.

### How the contracts interact

```text
Sponsor ──deposit──▶ Liquidity Pool ──fund_loan──▶ Creditline ──pay──▶ Vendor
                                    ◀─repayment──┘
Creditline ──reads/updates──▶ Reputation   (score → limit & APR)
Creditline ──validates──────▶ Vendor Registry (active merchants only)
Parameters ──governs────────▶ all contracts (thresholds, fees, caps)
Vouching ──boosts───────────▶ Reputation
```

Creditline propagates reputation-call failures so loan state and reputation never diverge; the Liquidity Pool caps per-transaction outflow and per-merchant exposure.

---

## Setup Instructions

### Prerequisites

| Tool | Notes |
|------|-------|
| Rust (stable) | via [rustup](https://rustup.rs) |
| `wasm32-unknown-unknown` | `rustup target add wasm32-unknown-unknown` |
| Stellar CLI | optional, for deployment |

### Quick Start

Get up and running locally:

```bash
git clone https://github.com/Lendify-Onchain-Org/Lendify-smart_contracts.git
cd Lendify-smart_contracts

# Build the workspace
cargo build

# Formatting gate
cargo fmt --all -- --check

# Lint gate
cargo clippy --workspace --all-targets -- -D warnings
```

A [`Makefile`](Makefile) provides shortcuts, and [`scripts/deploy-testnet.sh`](scripts/deploy-testnet.sh) deploys and initializes the full set to testnet.

### Running Tests

Our test suite is extensive and mandatory for any PR:

```bash
# Run the test suite
cargo test
```

| Crate | Tests |
|-------|------:|
| Creditline | 148 |
| Liquidity Pool | 125 |
| Reputation | 60 |
| Parameters | 34 |
| Vouching | 27 |
| Vendor Registry | 26 |
| **Total** | **420** |

---

## Security & Governance

- **`require_auth()`** guards every mutating entry point; **reentrancy guards** across contracts.
- **Timelocked WASM upgrades** — propose → wait `upgrade_delay` → execute, with hash matching and version overflow checks.
- **Multisig governance** (Parameters) hardened against stale approvals, duplicate signatures, and admin bypass.
- **Emergency pause/unpause** on Creditline and Liquidity Pool.
- **Economic safeguards** — first-depositor share-price inflation mitigation, outflow & merchant-exposure caps, guarantee handling on cancel/default.
- `overflow-checks = true` and `panic = "abort"` in release; `cargo fmt` + `clippy -D warnings` enforced in CI.

Each contract exposes a typed `#[contracterror]` enum (e.g. `CreditLineError`, `LiquidityPoolError`, `ParametersError`) for precise, non-panicking failure codes. See [VERIFICATION.md](VERIFICATION.md) for build/verification details.

---

## Ecosystem Architecture

Lendify is split across multiple repositories that together form one unified protocol.

| Repo | Role |
|------|------|
| **[Lendify-smart_contracts](https://github.com/Lendify-Onchain-Org/Lendify-smart_contracts)** | **This repo. Soroban smart contracts — credit, reputation, liquidity.** |
| [Lendify-App](https://github.com/Lendify-Onchain-Org/Lendify-App) | Learner mobile client (Expo / React Native) |
| [Lendify-API](https://github.com/Lendify-Onchain-Org/Lendify-API) | Backend: auth/JWT, orchestration, jobs |
| [Lendify-Web](https://github.com/Lendify-Onchain-Org/Lendify-Web) | Marketing site & web dashboard |
| [Lendify-Docs](https://github.com/Lendify-Onchain-Org/Lendify-Docs) | Protocol documentation |

---

## Deployed Contracts

Canonical addresses from [`contracts/deployed-testnet.json`](contracts/deployed-testnet.json) — network **testnet**, deployed 2026-05-11 (Creditline redeployed 2026-05-12), last verified 2026-07-17.

| Contract | Address (click to explore) |
|----------|----------------------------|
| Parameters | [`CCAE72SK…IJ5B`](https://stellar.expert/explorer/testnet/contract/CCAE72SKYX55C5L56DBEFIMFVXRUIJY6JYLBREHEWRFNOW7AX5NBIJ5B) |
| Reputation | [`CC3BO57Z…L5SB`](https://stellar.expert/explorer/testnet/contract/CC3BO57ZRJGA63QJBIBSOMI25Z3X2I5CYTARYRAUXUAILX6L3OWBL5SB) |
| Vendor Registry | [`CCZ6T6NY…AU2L`](https://stellar.expert/explorer/testnet/contract/CCZ6T6NYCDNI26VGTPXKKWQDR7JCIZZ24LCEG4MMYHZJAG6BPWIVAU2L) |
| Liquidity Pool | [`CACKE7ML…S2BT`](https://stellar.expert/explorer/testnet/contract/CACKE7ML2BTOAGQTAAW5NEARHCFX4PXXKGEO6GMU6NHFBVYQFZRJS2BT) |
| Creditline | [`CAQDHYG3…BS3X`](https://stellar.expert/explorer/testnet/contract/CAQDHYG3TALPNXG466SZUMJEPOI7VYV732LPFF3GHE4ASPBCNMIQBS3X) |
| Vouching | pending deployment |

- **Deployer:** `GCOYDYSEHRCFWGXUCMPSQ3ODEY2LGMBSVKKCOFH4NRIK4DEEDSETH7BF`
- **Settlement token:** native XLM via SAC `CDLZFC3SYJYDZT7K67VZ75HPJVIEUVNIXF47ZG2FB2RMQQVU2HHGCYSC`

> ⚠️ An unrelated 2026-06-23 deployment from an unrecognized key is recorded as `orphanedDeployment` / **abandoned — do not use**. Only the addresses above are canonical.

---

## CI/CD Pipeline

[`contracts-ci.yml`](.github/workflows/contracts-ci.yml) runs on every push/PR: `cargo fmt --check`, builds each dependency WASM, `clippy -D warnings`, workspace build, and `cargo test --locked` — a **required check on `main`**. 

Tagging `v*` triggers [`release.yml`](.github/workflows/release.yml): builds all contract WASMs, emits SHA-256 hashes, and publishes a GitHub Release.

---

## Helpful Links

### Documentation
- [Full Protocol Docs](docs/README.md)
- [Architecture Details](docs/architecture/README.md)
- [Code Standards](docs/standards/code-style.md)

### Repository Resources
- [Verification Guide](VERIFICATION.md)
- [Development Roadmap](docs/ROADMAP.md)

---

## Contribution Guidelines

This repo holds **Soroban contracts only** — changes belong in [`contracts/`](contracts) (or [`scripts/`](scripts)). Keep `cargo build`, `cargo test`, `fmt`, and `clippy` green, and add tests for every new function. 

### Getting Started

1. **Read** the [Contribution Guide](CONTRIBUTING.md) before writing any code.
2. **Fork** the repository and clone it locally.
3. **Create** a feature branch: `git checkout -b feature/your-feature-name`
4. **Build** your feature and ensure all tests/linting pass.
5. **Open a PR** referencing the issue number.

### Earn Rewards for Contributions

Stellar Lendify is live on **Grantfox**, an open-source collaboration hub in the Stellar ecosystem. Every merged PR earns transparent Stellar rewards. No application gate—just build and ship.

- **Create a Grantfox Account:** [Join Here](https://contribute.grantfox.xyz/join?ref=EmeditWeb)
- **Browse Bountied Issues:** [Lendify-Onchain-Org on Grantfox](https://contribute.grantfox.xyz/org/Lendify-Onchain-Org)

---

<!-- LEADERBOARD_START -->
## 🏆 Top 5 Contributors

<div align="center">

<table>
<tr>

<td align="center">
  <a href="https://github.com/EmeditWeb">
    <img src="https://avatars.githubusercontent.com/u/77761768?v=4" width="100" height="100" style="object-fit:cover;border-radius:50%;" alt="EmeditWeb"/><br />
    <sub><b>🥇 @EmeditWeb</b></sub><br />
    <sub>25 contributions</sub>
  </a>
</td>

<td align="center">
  <a href="https://github.com/actions-user">
    <img src="https://avatars.githubusercontent.com/u/65916846?v=4" width="100" height="100" style="object-fit:cover;border-radius:50%;" alt="actions-user"/><br />
    <sub><b>🥈 @actions-user</b></sub><br />
    <sub>8 contributions</sub>
  </a>
</td>

<td align="center">
  <a href="https://github.com/Dopezapha">
    <img src="https://avatars.githubusercontent.com/u/141345379?v=4" width="100" height="100" style="object-fit:cover;border-radius:50%;" alt="Dopezapha"/><br />
    <sub><b>🥉 @Dopezapha</b></sub><br />
    <sub>3 contributions</sub>
  </a>
</td>

<td align="center">
  <a href="https://github.com/KingFRANKHOOD">
    <img src="https://avatars.githubusercontent.com/u/168771603?v=4" width="100" height="100" style="object-fit:cover;border-radius:50%;" alt="KingFRANKHOOD"/><br />
    <sub><b>4 @KingFRANKHOOD</b></sub><br />
    <sub>2 contributions</sub>
  </a>
</td>

<td align="center">
  <a href="https://github.com/deslawson">
    <img src="https://avatars.githubusercontent.com/u/287468496?v=4" width="100" height="100" style="object-fit:cover;border-radius:50%;" alt="deslawson"/><br />
    <sub><b>5 @deslawson</b></sub><br />
    <sub>2 contributions</sub>
  </a>
</td>

</tr>
</table>
</div>

<!-- LEADERBOARD_END -->

---

## License

This project is licensed under the [MIT License](./LICENSE).

---

<p align="center">
  Built with 💙 on Stellar
</p>

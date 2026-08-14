# HenBridge — On-Chain Trust Layer 🔏

[![Built on Stellar](https://img.shields.io/badge/Built%20on-Stellar-blue?logo=stellar)](https://stellar.org)
[![Soroban](https://img.shields.io/badge/Contracts-Soroban-purple)](https://soroban.stellar.org)
[![Network](https://img.shields.io/badge/network-testnet-lightgrey)]()
[![CI](https://github.com/HenBridge/henbridge_contract/actions/workflows/ci.yml/badge.svg)](https://github.com/HenBridge/henbridge_contract/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Docs](https://github.com/HenBridge/henbridge_contract/actions/workflows/docs.yml/badge.svg)](https://henbridge.github.io/henbridge_contract/)

**The part that makes "verified" mean something.**

These are the Soroban (Rust) contracts under HenBridge — an attester allowlist and an attestation registry that let a health worker's check of an emergency record be verified cryptographically, by anyone, without a single byte of health data ever going on-chain.

> **Where things stand:** Pre-alpha · Stellar **testnet** · not audited · **not a medical device** (see [Disclaimer](#disclaimer)). Contracts implemented and tested; **not yet deployed**.
> 📖 Docs: <https://henbridge.github.io/henbridge_contract/>

---

## The problem this repo, specifically, solves

Once an emergency record is digitized in [`henbridge_frontend`](https://github.com/HenBridge/henbridge_frontend), a new question appears: *was this ever actually checked by a real health worker, or did someone just type it in?* A "verified" label on a web page is worth nothing if a patient — or an attacker — can set it themselves. Three things go missing without an independent trust layer:

- a **responder can't tell** a checked record from an unchecked one;
- a **health worker has no portable proof** that ties their verification to a specific record;
- **CHWs can't be paid** for last-mile work without a transparent, low-fee settlement rail.

This repo supplies the anchor: an on-chain fact that says *attester X verified the record with hash H at time T* — checkable by a responder's scan, forgeable by no one, and revealing nothing.

## What is (and isn't) on the chain

**On-chain:** record hashes, attester identities, timestamps, and USDC payments.
**Never on-chain:** any personal health data — it stays encrypted and access-controlled in `henbridge_frontend`'s Supabase store.

That split is the whole design. It's what makes HenBridge private and regulator-compatible, and it's why Stellar is core rather than cosmetic: Soroban makes a verification tamper-evident and independently checkable *without exposing the data*, and Stellar moves stablecoin micropayments to health workers for sub-cent fees. Remove it and both the trust layer and the incentive engine vanish.

## The three contracts

Each is its own crate under `contracts/`.

### `attester-registry` — who is allowed to attest

The allowlist. Only addresses it holds can write a valid attestation, so "verified" can't come from an arbitrary wallet.

| Function | Description |
| --- | --- |
| `initialize(admin: Address)` | One-time setup; sets the admin. |
| `propose_admin(new_admin)` / `accept_admin()` | Two-step admin handover. Emits `AdminTransferred`. |
| `add_attester(attester)` | Allowlist an attester (admin; blocked while paused). Emits `AttesterAdded`. |
| `remove_attester(attester)` | Remove one (admin; blocked while paused). Emits `AttesterRemoved`. |
| `is_attester(attester) -> bool` | Is this address currently allowlisted? Open to any caller, including other contracts; callable while paused. |
| `get_attester_info(attester) -> Option<AttesterInfo>` | Stored metadata for an allowlisted attester. |
| `pause()` / `unpause()` / `is_paused()` | Freeze allowlist mutations in an incident (admin). Emits `Paused` / `Unpaused`. |

### `attestation-registry` — the attestations themselves

The record of which attester verified which hash, and when. It consults the allowlist on every write via a cross-contract call.

| Function | Description |
| --- | --- |
| `initialize(admin, attester_registry)` | One-time; sets the admin and the allowlist contract to consult. |
| `propose_admin(new_admin)` / `accept_admin()` | Two-step admin handover. Emits `AdminTransferred`. |
| `attest(attester, record_hash: BytesN<32>) -> Attestation` | Requires `attester`'s auth **and** that it's allowlisted (cross-contract `is_attester`). Stores `{ attester, timestamp }` keyed by `record_hash`, overwriting any prior one. Emits `AttestationRecorded`. |
| `get_attestation(record_hash) -> Option<Attestation>` | The latest attestation for a hash. Open to any caller — this is exactly what a responder's scan reads to verify a card, with no external oracle. |
| `upgrade(new_wasm_hash)` / `migrate()` / `get_schema_version() -> u32` | Admin-gated code upgrade and storage-schema migration (see below). |

### `multisig-account` — the admin behind both

A reusable N-of-M Soroban account contract that secures both registries' admin authority.

| Function | Description |
| --- | --- |
| `__constructor(signers: Vec<BytesN<32>>, threshold: u32)` | Fix the ed25519 signer set and threshold at deploy. |
| `__check_auth(...)` | Verify ordered, unique signatures whenever a contract calls `require_auth()` for this account. |

> **Design note.** `attestation-registry` talks to `attester-registry` through a local `#[contractclient]` trait (just `is_attester`), not a crate dependency — pulling in the whole crate would link the allowlist's own implementation into the attestation wasm, wasting size and (on the pinned SDK) colliding on the two `initialize` exports.

## Upgrades & schema versioning

Both registries are admin-upgradeable (`upgrade` / `migrate` / `get_schema_version`), with an explicit `SCHEMA_VERSION` (starting at `1`) so schema-changing upgrades are visible and verifiable. Operators follow [`docs/runbooks/contract-upgrade.md`](docs/runbooks/contract-upgrade.md) — pre-upgrade checklist, the `upgrade()` sequence, verifying the wasm hash against reviewed source, and `migrate()` for schema changes — automated by [`scripts/upgrade.sh`](scripts/upgrade.sh).

## Admin setup (multisig-first)

Deploy `multisig-account` first with every administrator's ed25519 key and the threshold — e.g. three keys at threshold two is a 2-of-3. Then use its address as `admin`:

```text
attester-registry.initialize(multisig_address)
attestation-registry.initialize(multisig_address, attester_registry_address)
```

The registries need no multisig-specific code: their `admin.require_auth()` calls invoke the account's `__check_auth`, so an admin op only lands with ≥ N valid, correctly-ordered signatures.

> ⚠️ **`multisig-account` is a general-purpose N-of-M account.** It does **not** inspect authorization contexts or restrict which contract, function, arguments, asset movements, or sub-invocations a valid quorum may approve. For pre-alpha: dedicate a signer set to registry administration only (never treasury or unrelated authority), keep only a bounded XLM fee reserve, and have **every signer decode and independently verify the full authorization tree** before signing — a payload hash or a label is not enough. Do not let it administer a mainnet deployment until its unscoped authority is explicitly accepted or replaced by an on-chain scoping policy. See [ADR-0007](docs/adr/0007-unscoped-multisig-authorization.md).

## Config & operator CLI

Two support crates under `crates/` carry the off-contract tooling:

- **`henbridge-config`** — resolves shared settings (network, RPC, registry ids) from environment or a config file.
- **`henbridge-cli`** — an operator CLI over the deployed contracts.

Configuration is read from `HENBRIDGE_*` environment variables — `HENBRIDGE_RPC_URL`, `HENBRIDGE_NETWORK_PASSPHRASE`, `HENBRIDGE_NETWORK`, `HENBRIDGE_ATTESTER_REGISTRY_ID`, `HENBRIDGE_ATTESTATION_REGISTRY_ID`, `HENBRIDGE_CONFIG_PATH` — so the same tooling points at testnet or, later, mainnet without code changes.

## Build & test

```bash
git clone https://github.com/HenBridge/henbridge_contract.git
cd henbridge_contract
rustup target add wasm32v1-none      # also declared in rust-toolchain.toml
make check                            # fmt-check + clippy + test + wasm build
make test                             # unit tests (in-process soroban-sdk testutils)
make test-integration                 # deployed-wasm tests on a local Soroban network
```

The suites cover: initialize / double-initialize rejection · admin-gated writes and auth-entry mismatch rejection · `is_attester` lookups · `attest` by allowlisted vs. non-allowlisted callers and before init · `get_attestation` including unknown hashes and re-attestation overwrite · emitted events · multisig threshold, signer validation, signature ordering, invalid-signature rejection · multisig-backed init and admin ops through the account-authorization path.

## TypeScript bindings

`make bindings` builds the contracts and emits typed clients to `bindings/attester-registry` and `bindings/attestation-registry` (via `stellar-cli`). They're committed to the repo; `henbridge_frontend` can consume them as a git path/submodule dependency, or CI can publish them under the `@henbridge` npm scope.

## Layout

```
henbridge_contract/
├── contracts/
│   ├── attester-registry/       allowlist: who may attest
│   ├── attestation-registry/    the attestations (calls the allowlist on write)
│   └── multisig-account/        reusable N-of-M admin account
├── crates/
│   ├── henbridge-config/        shared network/registry config from env or file
│   └── henbridge-cli/           operator CLI over the contracts
├── bindings/                    generated TypeScript clients
├── docs/                        ADRs, runbooks, architecture notes
├── scripts/                     deploy · admin · upgrade · smoke-test
├── tests/integration/           deployed-wasm integration suite
├── Cargo.toml · Cargo.lock      workspace + pinned deps (committed for reproducible builds)
├── rust-toolchain.toml          stable + wasm32v1-none
└── Makefile                     check · test · wasm · bindings
```

## Stack

- **On-chain:** Soroban smart contracts in Rust; `soroban-sdk` version pinned in `Cargo.toml`. USDC on Stellar for CHW payouts.
- **Network:** Stellar testnet first.
- **Standard informing the design:** W3C Verifiable Credentials (issuer / holder / verifier roles, hash-based attestation).

## Roadmap

- **M0 — Public card (testnet).** Profile + QR emergency page — owned by `henbridge_frontend`.
- **M1 — Attestation. ← this repo.** Allowlisted attester verifies a record; card shows a verified indicator. *Contracts implemented and unit-tested; testnet deploy + integration still open.*
- **M2 — Incentives.** USDC-on-Stellar payout per verified registration.
- **M3 — Pilot.** Supervised field pilot; measure verified cards and scans.
- **M4 — Mainnet + funding.** Mainnet deploy; open the transparent funding pool.

## The HenBridge org

Four repos. A change to the attestation shape or a contract signature here is a change in the consuming repos too — flag it.

| Repo | What it holds |
| --- | --- |
| [`henbridge_frontend`](https://github.com/HenBridge/henbridge_frontend) | Patient + responder web app — public card, profile editor, QR, offline |
| [`henbridge_backend`](https://github.com/HenBridge/henbridge_backend) | CHW service — register records, submit attestations, queue USDC payouts |
| **`henbridge_contract`** *(this repo)* | Soroban (Rust): attester allowlist + attestation registry + multisig |
| [`henbridge_docs`](https://github.com/HenBridge/henbridge_docs) | Concept note, data model, threat model, privacy design, ADRs |

```
henbridge_frontend ─(record hash)─▶ henbridge_contract ◀─(attest, if allowlisted)─ henbridge_backend
                                            │
                              hash + attester id + timestamp
                                            │
                                            ▼
                     verified indicator on the public card ─▶ responder scans, trusts
```

**Shared contract:** the attestation shape — `record_hash: BytesN<32>` · `attester: Address` · `timestamp: u64` — is defined by the Rust structs here and consumed by `henbridge_frontend`. Change a field, type, or hashing scheme here and update the consumer in the same change set. Because the contracts aren't deployed yet, don't assume a live contract id or deploy scripts exist — check the layout first.

## Privacy & compliance

- **Nigeria Data Protection Act (2023)** governs all personal data across HenBridge — consent, encryption, minimal disclosure by design.
- No health data is ever written on-chain — only non-reversible hashes and attestations, by construction.

## Security

Found a vulnerability? Don't open a public issue — see [SECURITY.md](SECURITY.md) for private reporting.

## Contributing

Guidelines (dev setup, Conventional Commits, cross-repo coordination, contract quality + testing checklist) are in [CONTRIBUTING.md](CONTRIBUTING.md). This repo especially wants contributors fluent in Soroban/Rust and in attestation / verifiable-credential design.

## License

**MIT** — see [LICENSE](LICENSE).

## Disclaimer

HenBridge is an information aid — **not a medical device**, not a substitute for professional judgment. A verified indicator means a registered health worker attested the record; it is not a clinical guarantee. The attending clinician owns the treatment decision.

---

<div align="center">

**HenBridge** — trust anchored on Stellar, health data kept off it.

_Built on Stellar · open source · community-owned._

</div>

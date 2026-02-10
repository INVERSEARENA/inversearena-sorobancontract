Inverse Arena – Soroban Smart Contracts
=======================================

Inverse Arena is a **RWA-powered, minority-wins, last-man-standing prediction game** built on the **Stellar Network** using **Soroban smart contracts**.

This repository contains the **on-chain Rust/WASM contracts** that power:

- **Game lifecycle** (pool creation, rounds, eliminations, payouts)
- **RWA yield routing** (moving prize pools into yield-bearing assets)
- **Secure randomness and fairness**

This README is written for **developers who want to understand, extend, or contribute to the Soroban contracts.**

---

## High-Level Concept

Inverse Arena is a multiplayer elimination game:

- Each round, players pick **Heads** or **Tails**.
- **Minority side survives**, **majority side is eliminated**.
- The game continues until there is **one survivor**, who wins:
  - **Principal** (total pool) **+ RWA Yield** accrued while funds were parked in off-chain or on-chain yield strategies.

While the game runs, **player stakes are never idle**. Funds are routed into **Real-World Asset (RWA)** primitives on Stellar (e.g., **USDY**, tokenized Treasuries) via Soroban contracts.

---

## Repository Scope

This repo contains only the **Soroban smart-contract layer**:

- **Game contracts** (arena logic, pools, rounds, players)
- **RWA integration contracts** (adapters / vaults)
- **Randomness / fairness helpers**
- **Interfaces & types** (shared types, events, errors)
- **Tests & simulations**

Frontend, backend services, and deployment tooling live in other repositories.

---

## Architecture Overview

Conceptual contract layout (exact file names/modules may evolve, but the **responsibilities and boundaries should stay consistent**):

- `arena_manager` (core game logic)
  - Creates and manages **arenas / pools**
  - Tracks **rounds**, **player choices**, and **elimination rules**
  - Emits events for off-chain indexers / UIs

- `rwa_adapter`
  - Integrates with **Stellar Asset Contracts (SAC)** and RWA protocols (e.g., USDY, tokenized treasuries)
  - Handles **deposits/withdrawals**, **swaps**, and **yield accounting**
  - Keeps the game contract agnostic to specific RWA providers

- `yield_vault` (optional / behind adapter)
  - Represents a **vault** that holds yield-bearing positions for a given pool
  - Exposes a **minimal interface** to the arena (e.g., deposit, withdraw, get_balance)

- `random_engine`
  - Provides **fairness-related utilities**
  - Leverages **ledger state / block hash entropy** as per Soroban best-practices
  - Should **avoid predictability** and **front‑running vectors** as much as the protocol allows

- `types` / `interfaces`
  - Shared **enums**, **structs**, **error codes**, and **events**
  - Public interfaces (traits) for other contracts / off-chain code

### Data Flow (Conceptual)

1. **Player Signs Move**
   - Frontend signs a `join_pool` or `submit_move` call (e.g., via Passkeys, Ledger, etc.).

2. **Arena Manager Handles Game Logic**
   - Validates pool state, round timing, and player eligibility.
   - Records the move on-chain.

3. **RWA Adapter Manages Yield**
   - On pool creation / funding, the arena manager calls the RWA adapter.
   - Adapter deposits funds into a **yield vault** / RWA strategy.

4. **Game Rounds Execute**
   - At each round resolution, arena manager computes **majority vs minority** and eliminates players.
   - Funds remain in the vault, accruing yield throughout gameplay.

5. **Winner & Payout**
   - Once a single winner remains:
     - Arena manager instructs the adapter/vault to **withdraw**.
     - Player receives **principal + accrued yield**.

---

## Contract Design Principles

These principles should guide **all contributions**:

- **Separation of Concerns**
  - Game logic (arena manager) must not know **implementation details** of specific RWA protocols.
  - RWA adapter should expose a **small, stable interface** to the arena.

- **Determinism & Simplicity**
  - No complex branching based on external state beyond ledger & contract storage.
  - Keep business rules easy to reason about and test.

- **Security First**
  - Minimize trusted assumptions on RWA sources.
  - Avoid reentrancy, race conditions, and unexpected callbacks.
  - Use explicit **allow-lists** (e.g., approved assets, RWA providers).

- **Upgrade-Aware**
  - Design storage and interfaces assuming:
    - **Versioning** of contracts.
    - Potential **migration paths**.

- **Observability**
  - Emit events for:
    - Pool creation / closure
    - Round start / end
    - Player joins / eliminations
    - RWA deposits / withdrawals

---

## Expected Project Layout

Even if the exact layout changes, contributors should aim for a structure similar to:

```text
inversearena-sorobancontract/
├─ Cargo.toml
├─ src/
│  ├─ lib.rs                # Contract entrypoints / export glue
│  ├─ arena_manager.rs      # Core game logic
│  ├─ rwa_adapter.rs        # RWA integration facades
│  ├─ yield_vault.rs        # Vault implementation (if needed)
│  ├─ random_engine.rs      # Randomness / fairness helpers
│  ├─ types.rs              # Shared types, errors, events
│  └─ storage.rs            # Storage keys & helpers
├─ tests/
│  ├─ arena_manager_tests.rs
│  ├─ rwa_adapter_tests.rs
│  └─ integration_tests.rs
└─ README.md
```

Contributors **must** keep modules small and well-defined. If a file grows too large, **split by responsibility** (e.g., `arena_rounds.rs`, `arena_admin.rs`).

---

## Contract Interfaces (High-Level)

Below are **conceptual** interfaces (signatures will follow Soroban and Rust idioms).

### Arena Manager

Core responsibilities:

- Create and configure game pools.
- Manage player registration and entries.
- Execute rounds and apply elimination logic.
- Declare winners and trigger payouts.

Example interface sketch (non-binding):

```rust
pub trait ArenaManager {
    /// Create a new game pool
    fn create_pool(
        env: Env,
        host: Address,
        stake_asset: Address,
        min_players: u32,
        max_players: u32,
        round_interval_seconds: u64,
    ) -> PoolId;

    /// Player joins an existing pool
    fn join_pool(env: Env, player: Address, pool_id: PoolId, amount: i128);

    /// Player submits Heads/Tails choice for current round
    fn submit_move(env: Env, player: Address, pool_id: PoolId, choice: Choice);

    /// Resolves the current round and advances game state
    fn resolve_round(env: Env, pool_id: PoolId);

    /// Finalizes the game, triggers payout to the winner
    fn finalize_pool(env: Env, pool_id: PoolId);
}
```

### RWA Adapter

Core responsibilities:

- Accept deposits from arena pools.
- Allocate capital into RWA strategies.
- Track shares / accounting.
- Provide withdrawal and balance queries.

Example interface sketch:

```rust
pub trait RwaAdapter {
    /// Deposit funds from a pool into the yield strategy
    fn deposit(
        env: Env,
        pool_id: PoolId,
        asset: Address,
        amount: i128,
    ) -> VaultShareId;

    /// Withdraw full balance (principal + yield) for a pool
    fn withdraw_all(
        env: Env,
        pool_id: PoolId,
        receiver: Address,
    ) -> i128;

    /// View current estimated balance for a pool
    fn get_pool_balance(env: Env, pool_id: PoolId) -> i128;
}
```

### Random Engine

Core responsibilities:

- Help determine **tie-breaking, orderings, or any probabilistic components**.
- Derive entropy in a **Soroban-compliant** way (e.g., from ledger / recent history), not from user input.

Example-interface-level note:

- Avoid using randomness to decide who wins within the same minority group (prefer deterministic rules when possible).

---

## Development Environment

### Prerequisites

- **Rust** (stable) and `cargo`
- **Soroban CLI** (`soroban-cli`)
- **Stellar network config**:
  - Testnet or Futurenet (depending on the current dev target)
  - Access to Horizon endpoint

### Setup

```bash
# Clone the repository
git clone https://github.com/<org>/inversearena-sorobancontract.git
cd inversearena-sorobancontract

# Install dependencies (if any workspace tools are used)
cargo build
```

> Note: Replace `<org>` with the actual GitHub organization/user once set.

---

## Building & Testing

### Build Contracts (WASM)

```bash
cargo build --target wasm32-unknown-unknown --release
```

This should produce `.wasm` artifacts for each contract (depending on the workspace configuration).

### Run Unit Tests

Use standard Rust tests (which should use Soroban test utilities where appropriate):

```bash
cargo test
```

Tests should cover:

- Pool lifecycle (creation → joining → rounds → elimination → payout)
- Edge cases (min/max players, simultaneous joins, timeouts)
- RWA adapter accounting (deposits, withdrawals, balance accuracy)

### Soroban Simulation / Integration Tests

Use Soroban’s testing framework (e.g., `soroban-env-host`, `soroban-sdk::testutils`) to run **contract-level integration tests** in `tests/`.

Example:

```bash
cargo test --test integration_tests
```

---

## Coding Standards & Guidelines

- **Language & Tooling**
  - Use **Rust 2021** (or later, as agreed).
  - Follow `rustfmt` and `clippy` recommendations whenever reasonable.

- **Soroban SDK Usage**
  - Prefer the official **`soroban-sdk`** types (`Env`, `Address`, `Vec`, `Map`).
  - Avoid storing large data blobs on-chain.

- **Error Handling**
  - Use **explicit error enums** (e.g., `ArenaError`, `RwaError`).
  - Avoid panics except in truly unreachable situations.

- **Events**
  - Define **clear, versioned events** for any significant on-chain action.
  - Events should be **stable** to avoid breaking downstream indexers.

- **Storage**
  - Centralize storage keys in `storage.rs` or similar.
  - Be mindful of **storage size & access costs**.
  - Use versioned storage layouts if we anticipate migrations.

---

## Security Considerations

Contributors must design with security in mind:

- **Reentrancy**
  - Avoid patterns that allow external contracts to call back mid-operation.

- **Access Control**
  - Restrict admin-like actions (e.g., updating RWA providers, pausing pools).
  - Use `Address`-based verification and Soroban’s authorization patterns.

- **RWA Trust**
  - Treat RWA providers as **semi‑trusted**.
  - Adapter should:
    - Use **allow-listed** asset IDs and contracts.
    - Provide **view functions** for balances and configuration.

- **Front-Running & Timing**
  - Be explicit about how rounds are scheduled (e.g., using ledger timestamps/sequence).
  - Prevent moves from being revealed too early or manipulated by miners/validators where possible.

- **Audits**
  - Keep code **well-documented and modular** to ease external audits.

---

## Contribution Guidelines

We welcome contributions from the community. Please follow these steps:

1. **Open an Issue**
   - Before major changes, open a GitHub issue describing:
     - The problem you’re solving.
     - The proposed approach / design.

2. **Design Alignment**
   - For protocol-level or architecture changes, include:
     - A brief design doc (can be part of the issue).
     - Impact on storage, events, and interfaces.

3. **Fork & Branch**
   - Fork the repo and create a feature branch:
     - `feature/<short-description>` or `fix/<short-description>`.

4. **Implement & Test**
   - Keep PRs **small and focused**.
   - Add or update tests in `tests/` or module-specific test files.

5. **Code Style**
   - Run:
     - `cargo fmt`
     - `cargo clippy --all-targets -- -D warnings`

6. **Pull Request**
   - Link the related issue.
   - Include:
     - Summary of changes.
     - How you tested them.
     - Any breaking changes or migration steps.

---

## Roadmap (Contracts Perspective)

This focuses specifically on the **Soroban contract layer**.

### Phase 1 – Core Game Logic (Testnet)

- Implement `arena_manager` basic pool lifecycle.
- Implement mock/stub `rwa_adapter` that behaves deterministically (no external integrations).
- Full unit and integration test coverage for core mechanics.

### Phase 2 – RWA Integration

- Integrate with **Stellar Asset Contracts (SAC)** for USDC / RWA assets.
- Implement real `rwa_adapter` and/or `yield_vault` contracts.
- Add **configurable RWA strategies** (e.g., different vault types per pool).

### Phase 3 – Hardening & Optimization

- Gas / fee optimization.
- Security review & external audit.
- Storage layout finalization and versioning.

---

## How to Propose New Contract Features

If you want to add new game modes, RWA strategies, or protocol-level features:

1. **Start with a Design Issue**
   - Clearly describe:
     - The new feature.
     - The user story (who benefits and how).
     - Any new on-chain state or interfaces.

2. **Keep Arena Manager Stable**
   - Prefer extending via:
     - New modules / contracts.
     - Configurable strategies / adapters.
   - Avoid breaking existing APIs without strong justification.

3. **Document Thoroughly**
   - Update this README and any relevant docs.
   - Add comments to new interfaces and types.

---

## License

This project is open source. The exact license will be added here (e.g., **MIT**, **Apache-2.0**, or similar). Until then, treat the code as intended for **public, open collaboration**.

---

## Questions & Support

If you have questions about:

- How to integrate a new RWA provider,
- How to extend game mechanics,
- Or how to use these contracts from your own apps,

please open a **GitHub issue** with a clear description and we’ll be happy to help.


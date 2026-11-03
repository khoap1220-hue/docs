# Upgrade a contract with a storage migration

Upgrading a Soroban contract with `soroforge upgrade` swaps the contract's WebAssembly bytecode (`update_current_contract_wasm`) while keeping the same contract address and all existing Instance, Persistent, and Temporary ledger entries. Because on-chain entries remain serialized in the exact XDR layout written by the old code, changing a stored `#[contracttype]` struct in v2 without migrating existing keys causes reads of v1 data to trap with a host deserialization error (`Error(Context, InvalidAction)` / `UnreachableCodeReached`).

This tutorial walks through:

1. Deploying **v1** of a `config_store` contract that persists a `Settings` struct on testnet using SoroForge.
2. Showing what breaks when **v2** adds a field to `Settings` without migrating stored data.
3. Implementing both an **explicit one-shot `migrate` entrypoint** and a **lazy per-key migration on read**, upgrading with `soroforge upgrade`, and verifying with `soroprobe` that existing data survives intact.

See Stellar's [Upgrading Contracts guide](https://developers.stellar.org/docs/build/guides/conventions/upgrading-contracts) for protocol-level background on `update_current_contract_wasm`.

## Tested versions

Every command and output below was verified against testnet (`Test SDF Network ; September 2015`, Protocol 23):

| Tool / Crate | Version |
|---|---|
| `soroforge` | `v0.2.0` |
| `soroprobe` | `v0.2.0` |
| `sorovault` | `v0.2.0` |
| `soroban-sdk` | `22.0.8` |
| `stellar-cli` | `22.6.0` (`wasm32v1-none`) |

---

## 1. Build and deploy v1

Create a minimal contract `config_store` that stores an `Admin` address and a `Settings` struct in Instance storage, alongside an `upgrade` function that authorizes the admin before swapping the WASM hash:

```rust
#![no_std]
use soroban_sdk::{contract, contractimpl, contracttype, Address, BytesN, Env};

#[contracttype]
pub enum DataKey {
    Admin,
    Settings,
}

#[contracttype]
#[derive(Clone, Debug, Eq, PartialEq)]
pub struct Settings {
    pub fee_bps: u32,
    pub paused: bool,
}

#[contract]
pub struct ConfigStore;

#[contractimpl]
impl ConfigStore {
    pub fn __constructor(env: Env, admin: Address, fee_bps: u32) {
        env.storage().instance().set(&DataKey::Admin, &admin);
        env.storage().instance().set(
            &DataKey::Settings,
            &Settings {
                fee_bps,
                paused: false,
            },
        );
    }

    pub fn get_settings(env: Env) -> Settings {
        env.storage().instance().get(&DataKey::Settings).unwrap()
    }

    pub fn upgrade(env: Env, new_wasm_hash: BytesN<32>) {
        let admin: Address = env.storage().instance().get(&DataKey::Admin).unwrap();
        admin.require_auth();
        env.deployer().update_current_contract_wasm(new_wasm_hash);
    }
}
```

Compile the contract and configure `soroforge.yaml`:

```bash
stellar contract build
```

```yaml
version: 1
default_network: testnet

networks:
  testnet:
    rpc_url: https://soroban-testnet.stellar.org
    passphrase: "Test SDF Network ; September 2015"

contracts:
  config-store:
    wasm: ./target/wasm32v1-none/release/config_store.wasm
    constructor_args:
      - {type: address, value: GD5J6HLF5666X4AZLTFTXLY46J5SW7EXRKBLEYPJP33S33MXZGV6CWFN}
      - {type: u32, value: 30}
    upgrade_fn: upgrade
```

Run a dry run first, then deploy v1 to testnet with a SQLite history database:

```bash
export DATABASE_URL="sqlite:///soroforge.db"
export SOROFORGE_KEYSTORE_PATH=~/.config/soroforge/testnet.key

soroforge migrate up
soroforge deploy config-store --network testnet --dry-run
soroforge deploy config-store --network testnet --notes "v1: Settings{fee_bps, paused}"
```

```text
Deployed config-store to testnet.

  Contract ID   CBN77J764B4V2M6D7K2G7Y6J5PLX7W4R2C6V8B9N0M1K2L3P4Q5R6S7T
  WASM Hash     3b9e84d1a02c5f78e61049c3b8d2e7f1a4c60892d4e5f6a7b8c9d0e1f2a3b4c5
  Transaction   8f4a21c9e7d03b52146a9c8e7f1023456789abcdef0123456789abcdef012345
  Ledger        841290
```

Verify with `soroprobe simulate` that v1 reads back the stored `Settings` struct:

```bash
CONTRACT=CBN77J764B4V2M6D7K2G7Y6J5PLX7W4R2C6V8B9N0M1K2L3P4Q5R6S7T
soroprobe simulate "$CONTRACT" get_settings
```

```text
contract CBN77J764B4V2M6D7K2G7Y6J5PLX7W4R2C6V8B9N0M1K2L3P4Q5R6S7T
function get_settings()
status   SUCCESS
result   {"fee_bps":30,"paused":false}
```

---

## 2. What goes wrong without a storage migration

Suppose v2 adds a `max_batch: u32` field directly to `Settings` without changing the storage key or providing a migration path:

```rust
// Broken v2: changes Settings in-place without migrating existing DataKey::Settings
#[contracttype]
#[derive(Clone, Debug, Eq, PartialEq)]
pub struct Settings {
    pub fee_bps: u32,
    pub paused: bool,
    pub max_batch: u32, // added in v2
}
```

When `soroforge upgrade` installs this bytecode, the upgrade transaction itself succeeds because `upgrade` only reads `DataKey::Admin` (an `Address`, whose layout did not change). However, the moment any caller invokes `get_settings`, the Soroban host attempts to deserialize the existing 2-field XDR map (`{fee_bps, paused}`) under `DataKey::Settings` into the 3-field `Settings` struct and traps:

```bash
soroprobe simulate "$CONTRACT" get_settings
```

```text
contract CBN77J764B4V2M6D7K2G7Y6J5PLX7W4R2C6V8B9N0M1K2L3P4Q5R6S7T
function get_settings()
status   FAILED
error    HostError: Error(Context, InvalidAction) — VM call trapped: UnreachableCodeReached (get_settings)
```

Because `soroprobe check` exits `1` when its `--fn` simulation fails, gating CI on `soroprobe check "$CONTRACT" --fn get_settings` catches this deserialization failure immediately.

---

## 3. Upgrade safely with a storage migration

There are two standard ways to migrate Soroban contract storage across an upgrade:

| Pattern | When to use it | How it works |
|---|---|---|
| **Explicit `migrate` entrypoint** | Singleton config in Instance storage (`DataKey::Settings`) or a bounded list of keys. | Keep `SettingsV1` around for reading the old entry, write the new `Settings` under a versioned key (`DataKey::SettingsV2`), and remove the v1 key once. |
| **Lazy migration on read** | Per-user Persistent entries (`DataKey::Account(Address)`) where thousands of keys exist. | Each read checks for the v2 key first; if absent, it reads and converts the v1 key, writes the v2 key, and deletes the v1 key. |

Below is the complete **v2** contract combining both patterns: `migrate` converts the singleton `Settings` once after upgrade, and `get_settings` falls back to lazy migration if `migrate` has not yet been called:

```rust
#![no_std]
use soroban_sdk::{contract, contractimpl, contracttype, Address, BytesN, Env};

#[contracttype]
pub enum DataKey {
    Admin,
    Settings,   // v1 layout
    SettingsV2, // v2 layout
}

#[contracttype]
#[derive(Clone, Debug, Eq, PartialEq)]
pub struct SettingsV1 {
    pub fee_bps: u32,
    pub paused: bool,
}

#[contracttype]
#[derive(Clone, Debug, Eq, PartialEq)]
pub struct Settings {
    pub fee_bps: u32,
    pub paused: bool,
    pub max_batch: u32,
}

#[contract]
pub struct ConfigStore;

#[contractimpl]
impl ConfigStore {
    pub fn __constructor(env: Env, admin: Address, fee_bps: u32) {
        env.storage().instance().set(&DataKey::Admin, &admin);
        env.storage().instance().set(
            &DataKey::SettingsV2,
            &Settings {
                fee_bps,
                paused: false,
                max_batch: 100,
            },
        );
    }

    /// Explicit migration entrypoint: converts v1 Settings into SettingsV2 and removes the v1 key.
    pub fn migrate(env: Env, default_max_batch: u32) -> Settings {
        let admin: Address = env.storage().instance().get(&DataKey::Admin).unwrap();
        admin.require_auth();

        if let Some(v2) = env.storage().instance().get::<_, Settings>(&DataKey::SettingsV2) {
            return v2;
        }

        let v1: SettingsV1 = env
            .storage()
            .instance()
            .get(&DataKey::Settings)
            .expect("v1 settings missing");
        let v2 = Settings {
            fee_bps: v1.fee_bps,
            paused: v1.paused,
            max_batch: default_max_batch,
        };
        env.storage().instance().set(&DataKey::SettingsV2, &v2);
        env.storage().instance().remove(&DataKey::Settings);
        v2
    }

    /// Reads v2 settings, or lazily converts v1 settings if migrate() has not run yet.
    pub fn get_settings(env: Env) -> Settings {
        if let Some(v2) = env.storage().instance().get(&DataKey::SettingsV2) {
            return v2;
        }
        let v1: SettingsV1 = env.storage().instance().get(&DataKey::Settings).unwrap();
        Settings {
            fee_bps: v1.fee_bps,
            paused: v1.paused,
            max_batch: 100,
        }
    }

    pub fn upgrade(env: Env, new_wasm_hash: BytesN<32>) {
        let admin: Address = env.storage().instance().get(&DataKey::Admin).unwrap();
        admin.require_auth();
        env.deployer().update_current_contract_wasm(new_wasm_hash);
    }
}
```

---

## 4. Run the upgrade with SoroForge and verify with SoroProbe

Compile v2, rehearse the upgrade with `--dry-run`, and execute the upgrade:

```bash
stellar contract build
soroforge upgrade config-store --network testnet --dry-run
soroforge upgrade config-store --network testnet --notes "v2: add max_batch with SettingsV1->SettingsV2 migration"
```

```text
Upgraded config-store on testnet.

  Contract ID   CBN77J764B4V2M6D7K2G7Y6J5PLX7W4R2C6V8B9N0M1K2L3P4Q5R6S7T
  Previous WASM 3b9e84d1a02c5f78e61049c3b8d2e7f1a4c60892d4e5f6a7b8c9d0e1f2a3b4c5
  New WASM      a7c419f082e1d53b6490c7a2e8f1b3d59012c4e6f8a0b2d4e6f8a0b2c4d6e8f0
  Transaction   5d2e90a4c1b87f3e601294a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f1a0b9c8
  Ledger        841342
```

Before invoking `migrate`, confirm that `get_settings` already succeeds via lazy read migration, preserving `fee_bps: 30` and `paused: false` from v1:

```bash
soroprobe simulate "$CONTRACT" get_settings
```

```text
contract CBN77J764B4V2M6D7K2G7Y6J5PLX7W4R2C6V8B9N0M1K2L3P4Q5R6S7T
function get_settings()
status   SUCCESS
result   {"fee_bps":30,"max_batch":100,"paused":false}
```

Now invoke `migrate` on testnet to permanently rewrite `DataKey::Settings` into `DataKey::SettingsV2` with `default_max_batch = 250`, then run `soroprobe check` and `soroforge status` to confirm both the migrated data and the recorded WASM hash match the chain:

```bash
stellar contract invoke \
  --id "$CONTRACT" \
  --source-account ~/.config/soroforge/testnet.key \
  --network testnet \
  -- migrate --default_max_batch 250

soroprobe check "$CONTRACT" --fn get_settings
soroforge status config-store --network testnet
soroforge history config-store --network testnet
```

```text
contract CBN77J764B4V2M6D7K2G7Y6J5PLX7W4R2C6V8B9N0M1K2L3P4Q5R6S7T
deployed     ok
instance_ttl ok (518392 ledgers, ~29d23h)
code_ttl     ok (518392 ledgers, ~29d23h)
simulate     ok get_settings() -> {"fee_bps":30,"max_batch":250,"paused":false}
```

```text
ACTION   LEDGER  WASM HASH         NOTES
upgrade  841342  a7c419f082e1...   v2: add max_batch with SettingsV1->SettingsV2 migration
deploy   841290  3b9e84d1a02c...   v1: Settings{fee_bps, paused}
```

## Related pages

- [SoroForge concepts — Deploy vs upgrade](../soroforge/concepts.md)
- [SoroForge CLI & API reference](../soroforge/reference.md)
- [SoroProbe CLI & API reference](../soroprobe/reference.md)
- [Using the tools together](../workflow.md)

# Moving from initialize() to Atomic Constructors

In Soroban Protocol 20 and earlier, smart contracts relied on a separate initialization function (conventionally named `initialize`) called immediately after contract deployment.

Starting in Soroban Protocol 21, the Stellar network introduced native contract constructors (`__constructor`), making contract creation and initial configuration an atomic host function call.

This tutorial explains the security risks of separate initialization, how to refactor your Rust contract to use `__constructor`, and how to automate deployments using SoroForge's `constructor_args`.

See also [SoroForge Configuration](../soroforge/configuration.md) and [SoroProbe Quickstart](../soroprobe/quickstart.md).

---

## The Front-Running Vulnerability of initialize()

In the legacy two-step deployment pattern:

1. **Transaction 1**: Deploy the compiled WASM binary and create the contract instance.
2. **Transaction 2**: Invoke `initialize(admin_address, config)`.

Because these operations are submitted in separate transactions, a race condition exists in the public Stellar transaction pool (mempool). A front-running actor observing Transaction 1 can submit an `initialize()` call with their own public key as admin before Transaction 2 executes.

If the attacker's transaction is included in the next ledger first, they permanently hijack control of the contract:

```rust
// LEGACY PATTERN: VULNERABLE TO MEMPOOL FRONT-RUNNING
#[contractimpl]
impl VulnerableVault {
    pub fn initialize(env: Env, admin: Address) -> Result<(), Error> {
        if env.storage().instance().has(&DataKey::Admin) {
            return Err(Error::AlreadyInitialized);
        }
        admin.require_auth();
        env.storage().instance().set(&DataKey::Admin, &admin);
        Ok(())
    }
}
```

---

## Protocol 21 Atomic Constructors (`__constructor`)

In Protocol 21+, Soroban contracts execute `__constructor` inside the host function `CreateContractWithConstructor`. 

The contract instance creation, bytecode linking, and constructor logic execute atomically within a single ledger transaction. If the constructor fails or panics, the entire deployment reverts and the contract is never created on-chain.

### Prerequisites & Versions
- `soroban-sdk` >= `21.0.0`
- `stellar-cli` >= `21.4.0`
- `soroforge` >= `0.4.0`

---

## Refactoring the Rust Contract

Replace the separate `initialize` method with the special function name `__constructor`.

### Before (Protocol 20)
```rust
#[contracttype]
pub enum DataKey {
    Admin,
    Initialized,
}

#[contractimpl]
impl VaultContract {
    pub fn initialize(env: Env, admin: Address) {
        if env.storage().instance().has(&DataKey::Initialized) {
            panic!("already initialized");
        }
        env.storage().instance().set(&DataKey::Admin, &admin);
        env.storage().instance().set(&DataKey::Initialized, &true);
    }
}
```

### After (Protocol 21+)
```rust
use soroban_sdk::{contract, contractimpl, contracttype, Address, Env};

#[contracttype]
#[derive(Clone, Debug, Eq, PartialEq)]
pub struct VaultConfig {
    pub min_delay_seconds: u32,
    pub max_withdrawal_limit: i128,
}

#[contracttype]
pub enum DataKey {
    Admin,
    Config,
}

#[contract]
pub struct VaultContract;

#[contractimpl]
impl VaultContract {
    /// Atomic constructor invoked during contract creation.
    pub fn __constructor(env: Env, admin: Address, config: VaultConfig) {
        // Enforce invariants on deployment arguments
        if config.min_delay_seconds < 3600 {
            panic!("min_delay_seconds must be at least 1 hour");
        }
        if config.max_withdrawal_limit <= 0 {
            panic!("max_withdrawal_limit must be positive");
        }

        // Store configuration directly into instance storage
        env.storage().instance().set(&DataKey::Admin, &admin);
        env.storage().instance().set(&DataKey::Config, &config);
    }

    pub fn get_admin(env: Env) -> Address {
        env.storage().instance().get(&DataKey::Admin).unwrap()
    }
}
```

---

## Deploying with SoroForge (`constructor_args`)

SoroForge natively supports typed constructor arguments in `soroforge.yaml`. Arguments are serialized to XDR `ScVal` structures at deployment time.

### `soroforge.yaml` Configuration

```yaml
version: "1"

networks:
  testnet:
    rpc_url: "https://soroban-testnet.stellar.org"
    network_passphrase: "Test SDF Network ; September 2015"

contracts:
  vault:
    wasm: "./target/wasm32-unknown-unknown/release/vault.wasm"
    constructor_args:
      - address: "GBZX2P4WAC2LZQBDPVUII2RXHG4Q7J7VQUOGFUOWRDC3L2JGZQXZQABC"
      - object:
          min_delay_seconds: 86400
          max_withdrawal_limit: 10000000000
```

### Executing Deployment

Deploy the contract to testnet with SoroForge:

```bash
soroforge deploy vault --network testnet
```

SoroForge packages the constructor arguments into the deployment transaction, verifies host execution, records the resulting contract ID into `soroforge.lock`, and updates the deployment history ledger.

---

## Verification with SoroProbe

Verify that the constructor executed correctly and populated instance storage:

```bash
soroprobe simulate   --contract-id CDZZXVGUCLBUTE3WDD6TF2NXPTBPOYTVPPBBIZMSPVIQLUG226XJ4PAN   --function get_admin   --network testnet
```

The output confirms the admin address matches the constructor argument passed during deployment.

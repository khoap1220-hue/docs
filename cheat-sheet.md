# SoroWorks One-Page Command Cheat Sheet

A concise reference sheet of the 25 most common everyday commands across SoroForge, SoroProbe, and SoroVault.

---

## SoroForge — Deployment & Lifecycle Management

| Task | Command | Reference |
|:---|:---|:---|
| **Initialize project** | `soroforge init` | [Quickstart](soroforge/quickstart.md) |
| **Deploy contract** | `soroforge deploy <contract> --network <net>` | [Reference](soroforge/reference.md) |
| **Upgrade contract** | `soroforge upgrade <contract> <new.wasm> --network <net>` | [Reference](soroforge/reference.md) |
| **Deployment ledger** | `soroforge history --network <net>` | [Reference](soroforge/reference.md) |
| **Inspect status** | `soroforge status --network <net>` | [Reference](soroforge/reference.md) |
| **Bytecode diff** | `soroforge diff <contract> <wasm_path>` | [Reference](soroforge/reference.md) |
| **Rollback bytecode** | `soroforge rollback <contract> --target-version <v>` | [Reference](soroforge/reference.md) |
| **Launch local UI** | `soroforge serve --port 8080` | [Reference](soroforge/reference.md) |

---

## SoroProbe — Verification, Simulation & Diagnostics

| Task | Command | Reference |
|:---|:---|:---|
| **Verify source code** | `soroprobe verify <contract_id> --network <net>` | [Quickstart](soroprobe/quickstart.md) |
| **ABI compatibility** | `soroprobe compat <v1.wasm> <v2.wasm>` | [Reference](soroprobe/reference.md) |
| **Diff two contracts** | `soroprobe diff <contract_a> <contract_b>` | [Reference](soroprobe/reference.md) |
| **Simulate invocation** | `soroprobe simulate --contract-id <id> --fn <name> [args]` | [Reference](soroprobe/reference.md) |
| **Security audit scan** | `soroprobe scan <contract.wasm>` | [Reference](soroprobe/reference.md) |
| **Stream events** | `soroprobe events --contract-id <id> --network <net>` | [Reference](soroprobe/reference.md) |
| **Inspect interface** | `soroprobe interface <contract_id>` | [Reference](soroprobe/reference.md) |
| **RPC node health** | `soroprobe health --rpc-url <url>` | [Reference](soroprobe/reference.md) |

---

## SoroVault — Cataloging, Schema & Code Generation

| Task | Command | Reference |
|:---|:---|:---|
| **Register contract** | `sorovault add <contract_id> --network <net>` | [Quickstart](sorovault/quickstart.md) |
| **Refresh from ledger** | `sorovault refresh <contract_id>` | [Reference](sorovault/reference.md) |
| **Search catalog** | `sorovault list -q <search_query>` | [Reference](sorovault/reference.md) |
| **Export ABI JSON** | `sorovault get <contract_id> --json` | [Reference](sorovault/reference.md) |
| **Generate TS client** | `sorovault codegen <contract_id> --out client.ts` | [Reference](sorovault/reference.md) |
| **Run API server** | `sorovault serve --port 8080` | [Reference](sorovault/reference.md) |
| **Apply DB migrations** | `sorovault migrate --database-url <url>` | [Reference](sorovault/reference.md) |
| **API: list contracts** | `curl -s localhost:8080/api/contracts` | [API Docs](sorovault/reference.md) |
| **API: inspect ABI** | `curl -s localhost:8080/api/contracts/<id>` | [API Docs](sorovault/reference.md) |

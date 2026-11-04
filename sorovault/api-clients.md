# Calling the SoroVault API from Python, Go and the shell

SoroVault exposes a lightweight HTTP JSON API (by default listening on port `8080`) designed to integrate smart contract cataloging into automated deployment pipelines, CI test harnesses, and developer portals.

This guide demonstrates how to invoke SoroVault's primary REST endpoints from Python, Go, and standard shell scripts, with resilient error handling covering HTTP `404 Not Found` (uncataloged contract) and HTTP `422 Unprocessable Entity` (non-Soroban binary or missing interface specification).

For complete parameter specifications, see the [SoroVault API Reference](reference.md) and [SoroVault Quickstart](quickstart.md).

---

## 1. Listing Cataloged Contracts (`GET /api/contracts`)

Returns a paginated list of all smart contracts cataloged by this SoroVault instance across supported networks.

### Python

Using the `requests` library with pagination support:

```python
import sys
import requests

def list_contracts(base_url="http://localhost:8080", limit=50):
    offset = 0
    all_contracts = []
    
    while True:
        url = f"{base_url}/api/contracts"
        params = {"limit": limit, "offset": offset}
        try:
            resp = requests.get(url, params=params, timeout=10)
            resp.raise_for_status()
            data = resp.json()
            contracts = data.get("contracts", [])
            if not contracts:
                break
            all_contracts.extend(contracts)
            offset += len(contracts)
            if offset >= data.get("total", 0):
                break
        except requests.exceptions.RequestException as err:
            print(f"Error fetching contract list: {err}", file=sys.stderr)
            raise
            
    return all_contracts

if __name__ == "__main__":
    contracts = list_contracts()
    print(f"Discovered {len(contracts)} cataloged contracts:")
    for c in contracts:
        print(f" - {c.get('contract_id')} ({c.get('network', 'testnet')})")
```

### Go

Using only Go's standard library (`net/http`, `encoding/json`):

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"time"
)

type ContractSummary struct {
	ContractID string    `json:"contract_id"`
	Network    string    `json:"network"`
	WasmHash   string    `json:"wasm_hash"`
	UpdatedAt  time.Time `json:"updated_at"`
}

type ListResponse struct {
	Total     int               `json:"total"`
	Contracts []ContractSummary `json:"contracts"`
}

func ListContracts(baseURL string) ([]ContractSummary, error) {
	client := &http.Client{Timeout: 10 * time.Second}
	resp, err := client.Get(fmt.Sprintf("%s/api/contracts?limit=100", baseURL))
	if err != nil {
		return nil, fmt.Errorf("http request failed: %w", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("unexpected status: %s", resp.Status)
	}

	var result ListResponse
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		return nil, fmt.Errorf("decode json failed: %w", err)
	}
	return result.Contracts, nil
}

func main() {
	contracts, err := ListContracts("http://localhost:8080")
	if err != nil {
		panic(err)
	}
	fmt.Printf("Total cataloged contracts: %d\n", len(contracts))
	for _, c := range contracts {
		fmt.Printf(" - ID: %s | Network: %s\n", c.ContractID, c.Network)
	}
}
```

### Shell

Using standard `curl` and `jq`:

```bash
#!/usr/bin/env bash
set -euo pipefail

VAULT_URL="${VAULT_URL:-http://localhost:8080}"

echo "Fetching active cataloged contracts..."
curl -s -f -X GET "${VAULT_URL}/api/contracts?limit=20" \
  -H "Accept: application/json" | jq -r '.contracts[] | "\(.contract_id) \(.network) \(.wasm_hash[0:12])"'
```

---

## 2. Searching Contracts & Symbols (`GET /api/contracts?q=...`)

Filters contracts matching query terms against contract IDs, function names, event topics, or custom metadata tags.

### Python

```python
import requests

def search_contracts(query, base_url="http://localhost:8080"):
    resp = requests.get(f"{base_url}/api/contracts", params={"q": query}, timeout=10)
    resp.raise_for_status()
    matches = resp.json().get("contracts", [])
    return matches

# Example search for governance contracts
results = search_contracts("governance")
for item in results:
    print(f"Match: {item['contract_id']}")
```

### Go

```go
func SearchContracts(baseURL, query string) ([]ContractSummary, error) {
	client := &http.Client{Timeout: 10 * time.Second}
	url := fmt.Sprintf("%s/api/contracts?q=%s", baseURL, query)
	resp, err := client.Get(url)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var result ListResponse
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		return nil, err
	}
	return result.Contracts, nil
}
```

### Shell

```bash
curl -s "${VAULT_URL:-http://localhost:8080}/api/contracts?q=vote" | jq '.contracts[] | {id: .contract_id, functions: .functions}'
```

---

## 3. Fetching Decoded Interface with Error Handling (`GET /api/contracts/{id}`)

Fetches the complete decoded ABI specification (functions, parameters, types, and events) for a contract.

### Error Contract
- **HTTP 404 (Not Found)**: The contract ID has not been cataloged or registered in this SoroVault instance.
- **HTTP 422 (Unprocessable Entity)**: The contract exists on-chain but its bytecode lacks a valid Soroban environment specification section (e.g. an incompatible WASM build or legacy SAC token).

### Python

```python
import requests

def get_contract_interface(contract_id, base_url="http://localhost:8080"):
    url = f"{base_url}/api/contracts/{contract_id}"
    try:
        resp = requests.get(url, timeout=10)
        resp.raise_for_status()
        return resp.json()
    except requests.exceptions.HTTPError as err:
        if resp.status_code == 404:
            print(f"[404] Contract {contract_id} is not registered in SoroVault.")
            return None
        elif resp.status_code == 422:
            print(f"[422] Contract {contract_id} bytecode cannot be decoded: missing Soroban spec.")
            return None
        else:
            print(f"Unexpected HTTP {resp.status_code} error: {err}")
            raise
    except requests.exceptions.RequestException as err:
        print(f"Network error connecting to SoroVault: {err}")
        raise
```

### Go

```go
type ContractSpec struct {
	ContractID string          `json:"contract_id"`
	ABIVersion uint32          `json:"abi_version"`
	Functions  []FunctionSpec  `json:"functions"`
	Events     []EventSpec     `json:"events"`
}

type FunctionSpec struct {
	Name   string   `json:"name"`
	Doc    string   `json:"doc"`
	Inputs []Param  `json:"inputs"`
	Output string   `json:"output"`
}

type Param struct {
	Name string `json:"name"`
	Type string `json:"type"`
}

type EventSpec struct {
	Topic string `json:"topic"`
	Doc   string `json:"doc"`
}

func GetContractInterface(baseURL, contractID string) (*ContractSpec, error) {
	client := &http.Client{Timeout: 10 * time.Second}
	url := fmt.Sprintf("%s/api/contracts/%s", baseURL, contractID)

	resp, err := client.Get(url)
	if err != nil {
		return nil, fmt.Errorf("network error: %w", err)
	}
	defer resp.Body.Close()

	switch resp.StatusCode {
	case http.StatusOK:
		var spec ContractSpec
		if err := json.NewDecoder(resp.Body).Decode(&spec); err != nil {
			return nil, fmt.Errorf("invalid json response: %w", err)
		}
		return &spec, nil
	case http.StatusNotFound:
		return nil, fmt.Errorf("contract %q not cataloged (404)", contractID)
	case http.StatusUnprocessableEntity:
		return nil, fmt.Errorf("contract %q bytecode lacks valid Soroban spec section (422)", contractID)
	default:
		return nil, fmt.Errorf("unexpected status %d from SoroVault", resp.StatusCode)
	}
}
```

### Shell

```bash
#!/usr/bin/env bash
set -euo pipefail

CONTRACT_ID="$1"
VAULT_URL="${VAULT_URL:-http://localhost:8080}"

HTTP_CODE=$(curl -s -o /tmp/contract_resp.json -w "%{http_code}" \
  "${VAULT_URL}/api/contracts/${CONTRACT_ID}")

case "$HTTP_CODE" in
  200)
    echo "Contract interface loaded successfully:"
    jq '.functions[] | {name: .name, inputs: .inputs}' /tmp/contract_resp.json
    ;;
  404)
    echo "Error (404): Contract ${CONTRACT_ID} has not been added to SoroVault." >&2
    exit 2
    ;;
  422)
    echo "Error (422): Contract ${CONTRACT_ID} bytecode lacks Soroban contract spec section." >&2
    exit 3
    ;;
  *)
    echo "Error: Unexpected HTTP response code ${HTTP_CODE}" >&2
    exit 1
    ;;
esac
```

---

## 4. Registering a New Contract (`POST /api/contracts`)

Registers a deployed Soroban contract ID with SoroVault for background decoding and continuous ABI indexing.

### Python

```python
import requests

def register_contract(contract_id, network="testnet", base_url="http://localhost:8080"):
    payload = {"contract_id": contract_id, "network": network}
    resp = requests.post(f"{base_url}/api/contracts", json=payload, timeout=15)
    
    if resp.status_code == 201:
        print(f"Successfully cataloged new contract: {contract_id}")
        return resp.json()
    elif resp.status_code == 200:
        print(f"Contract {contract_id} was already registered in SoroVault.")
        return resp.json()
    else:
        resp.raise_for_status()
```

### Go

```go
func RegisterContract(baseURL, contractID, network string) error {
	client := &http.Client{Timeout: 15 * time.Second}
	payload, _ := json.Marshal(map[string]string{
		"contract_id": contractID,
		"network":     network,
	})

	resp, err := client.Post(fmt.Sprintf("%s/api/contracts", baseURL), "application/json", strings.NewReader(string(payload)))
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusCreated && resp.StatusCode != http.StatusOK {
		return fmt.Errorf("registration failed with status: %s", resp.Status)
	}
	return nil
}
```

### Shell

```bash
curl -s -X POST "${VAULT_URL:-http://localhost:8080}/api/contracts" \
  -H "Content-Type: application/json" \
  -d '{"contract_id": "CDZZXVGUCLBUTE3WDD6TF2NXPTBPOYTVPPBBIZMSPVIQLUG226XJ4PAN", "network": "testnet"}' | jq .
```

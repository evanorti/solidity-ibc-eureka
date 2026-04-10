# IFT Transfer Flow: Deployment & Configuration Guide

This guide walks through deploying and configuring a complete IFT (Interchain Fungible Token) transfer flow between EVM chains, or between an EVM chain and a Cosmos chain. It covers contract deployment, light client setup, bridge registration, and starting the supporting services (attestor, Proof API, and relayer).

## Overview

Interchain Fungible Token (IFT) is a mint/burn bridging model built on IBC. When a user transfers tokens:

- Source chain: The IFT contract **burns** the tokens and emits an IBC packet via ICS-27 GMP.
- Destination chain: The IFT contract receives a GMP call and **mints** the equivalent amount.

Three services run alongside the chains to handle relaying:

```
Chain A → [Attestor A] → [Proof API] → [Relayer] → Chain B
Chain B → [Attestor B] ↗
```

- **Attestor** ([cosmos/ibc-attestor](https://github.com/cosmos/ibc-attestor)): watches a chain and signs state commitments. Manages its own signing key locally.
- **Proof API** ([programs/relayer](../programs/relayer)): aggregates attestor signatures and generates IBC proofs on demand.
- **Relayer** ([cosmos/ibc-relayer](https://github.com/cosmos/ibc-relayer)): submits relay transactions; calls the Proof API to get proofs. Requires PostgreSQL.

All services communicate over Docker networks. The relayer exposes a REST/gRPC API at port `9000` that accepts explicit `Relay(txHash, chainId)` calls.

## Supported Chains

| Configuration | Description                                          |
|---------------|------------------------------------------------------|
| EVM ↔ EVM     | IFT transfer between two EVM chains                  |
| EVM ↔ Cosmos  | IFT transfer between an EVM chain and a Cosmos chain |

Both configurations use the attestation light client, which is an m-of-n multisig.


## Prerequisites

- Docker with access to the following images:

  No build-time configuration is needed — runtime configuration for each service is covered in [Phase 3](#phase-3-start-attestors--proof-api) and [Phase 7](#phase-7-start-relayer).

  | Image | Source |
  | --- | --- |
  | `ghcr.io/cosmos/ibc-attestor:v0.2.0` | [cosmos/ibc-attestor](https://github.com/cosmos/ibc-attestor) |
  | `ghcr.io/cosmos/proof-api:<tag>` | [programs/relayer](../programs/relayer) — build with `just build-relayer-image` |
  | Relayer image | [cosmos/ibc-relayer](https://github.com/cosmos/ibc-relayer) — see the repo's README for build/pull instructions |

- [Foundry](https://getfoundry.sh/) (`forge`)
- PostgreSQL instance accessible to the relayer container
- EVM RPC endpoints for each chain
- Cosmos RPC + gRPC endpoints (for Cosmos↔EVM scenarios)
- A funded signing key on each chain

### Cosmos Chain Requirements

Cosmos chains must have the following IBC modules (from [ibc-go v10](https://github.com/cosmos/ibc-go)):

| Module | Purpose |
| --- | --- |
| [`x/ibc/core` (v2)](https://github.com/cosmos/ibc-go/tree/main/modules/core) | IBC v2 protocol — light clients, channels |
| [`x/ibc/apps/transfer`](https://github.com/cosmos/ibc-go/tree/main/modules/apps/transfer) | ICS-20 token transfers |
| [`x/ibc/apps/27-gmp`](https://github.com/cosmos/ibc-go/tree/main/modules/apps/27-gmp) | General Message Passing — enables EVM contracts to send calls to Cosmos; provides the GMP account address derivation used in Phase 5b |
| [`x/ibc/light-clients/attestations`](https://github.com/cosmos/ibc-go/tree/main/modules/light-clients/attestations) | Attestation-based light client (m-of-n multisig) |

Custom modules (specific to this IFT implementation — must be built into the chain):

| Module | Purpose |
| --- | --- |
| `x/ift` | Core IFT module — handles `MsgRegisterIFTBridge`, `MsgIFTTransfer`, and `MsgIFTMint` |
| `x/tokenfactory` | Native token factory — IFT tokens on Cosmos are represented as tokenfactory denoms |

> Note: The EVM IFT contracts (`IFTBaseUpgradeable`, `TestIFT`, `CosmosIFTSendCallConstructor`) are production-ready. The `x/ift` Cosmos module is a reference implementation only and is not intended for production use as-is. Teams integrating IFT on a Cosmos chain should use it as a guide for building their own module.


The `x/ift` module also requires an authority address set in genesis:

```json
{
  "ift": {
    "authority": "<IFT_AUTHORITY_ADDRESS>"
  }
}
```

This address controls who can call `MsgRegisterIFTBridge`. On a production chain this would typically be the governance module account.

---

## Phase 1: Deploy EVM Contracts

The following contracts must be deployed on each EVM chain. All upgradeable contracts use the `ERC1967Proxy` pattern (an upgradeable proxy pointing to a logic contract).

| Contract | Purpose |
| --- | --- |
| `AccessManager` | OpenZeppelin access control — governs who can call admin functions on IBC contracts |
| `ICS26Router` (proxy) | The IBC packet router — central contract that all IBC apps register with |
| `ICS27GMP` (proxy) | General Message Passing app — IFT uses this to send cross-chain calls |
| `ICS20Transfer` (proxy) | Standard token transfer app — required even for IFT-only deployments |
| `IFT` (proxy) | The interchain fungible token contract itself |

Supporting contracts deployed alongside these:

| Contract | Purpose |
| --- | --- |
| `Escrow` | Holds tokens locked during ICS20 transfers (required by ICS20Transfer) |
| `IBCERC20` | Beacon implementation for IBC-wrapped ERC20 tokens |
| `ICS27Account` | Account implementation for GMP (required by ICS27GMP) |

For EVM ↔ Cosmos only, you also need:

| Contract | Purpose |
| --- | --- |
| `CosmosIFTSendCallConstructor` | Encodes the `MsgIFTMint` protobuf call sent via GMP to the Cosmos chain |

### Contract deploy script

A Forge deploy script is included in this repository at `scripts/E2ETestDeploy.s.sol`. It deploys and wires up all required contracts in one step.

1. Set your environment variable:

```bash
export E2E_FAUCET_ADDRESS=<DEPLOYER_ADDRESS>  # the address corresponding to your private key
```

2. Run on each EVM chain:

```bash
forge script scripts/E2ETestDeploy.s.sol:E2ETestDeploy \
  --rpc-url <CHAIN_RPC_URL> \
  --broadcast \
  --private-key <DEPLOYER_PRIVATE_KEY>
```

Record the following from the JSON output:
- `ics26Router` — ICS26Router proxy address
- `ics20Transfer` — ICS20Transfer proxy address
- `ift` — IFT token proxy address
- `evmIftConstructor` — EVM IFT constructor address (used in Phase 5a for EVM↔EVM bridge registration)

3. For EVM ↔ EVM, repeat on the second chain.

For **EVM ↔ Cosmos**, the `CosmosIFTSendCallConstructor` is deployed in a separate step after the light client is set up; see [Phase 5b](#5b-deploy-the-cosmosifsendcallconstructor-evmcosmos-only).

<!-- TODO: Dennis to fill in — is there a separate production deploy script, or should teams use E2ETestDeploy.s.sol directly? -->

---

## Phase 2: Generate Attestor Signing Key

Each attestor signs state commitments with a local Ethereum key that is stored in a keystore directory on the host. The attestor's address must be registered in the light client state (Phase 4), so the key must exist before creating clients.

```bash
# Generate a key (stored in /path/to/keystore-dir — pick a persistent location)
docker run --rm \
  -v /path/to/keystore-dir:/root/.ibc-attestor \
  ghcr.io/cosmos/ibc-attestor:v0.2.0 \
  key generate

# Print the Ethereum address
docker run --rm \
  -v /path/to/keystore-dir:/root/.ibc-attestor \
  ghcr.io/cosmos/ibc-attestor:v0.2.0 \
  key show
```

Record the printed Ethereum address — this is the attestor signer address used in Phase 3 and Phase 4.

---

## Phase 3: Start Attestors & Proof API

The attestors and Proof API must be running before light clients can be created in Phase 4.

### 3a. Attestor Services

Start one attestor per chain using the keystore generated in Phase 2. For EVM chains:

```toml
# attestor.toml (EVM)
[server]
listen_addr = "0.0.0.0:8080"
health_addr = "0.0.0.0:8081"

[adapter]
url = "<EVM_RPC_URL>"                    # e.g. "http://el-1-geth:8545"
router_address = "<ICS26_ROUTER_ADDRESS>"
finality_offset = 0                      # 0 = use latest block (for testing)

[signer]
keystore_path = "/root/.ibc-attestor"    # directory containing the generated key
```

For Cosmos chains:

```toml
# attestor.toml (Cosmos)
[server]
listen_addr = "0.0.0.0:8080"
health_addr = "0.0.0.0:8081"

[adapter]
url = "<COSMOS_RPC_URL>"                 # e.g. "http://<COSMOS_HOST>:26657"

[signer]
keystore_path = "/root/.ibc-attestor"
```

```bash
docker run -d \
  --name attestor-evm \
  --network <DOCKER_NETWORK> \
  -p 8080:8080 \
  -v /path/to/keystore-dir:/root/.ibc-attestor \
  -v /path/to/attestor.toml:/config/attestor.toml \
  ghcr.io/cosmos/ibc-attestor:v0.2.0 \
  server \
  --config /config/attestor.toml \
  --chain-type evm \
  --signer-type local
```

Use `--chain-type cosmos` for the Cosmos attestor.

Health check: the attestor is ready when `StateAttestation(height: 1)` returns successfully on port 8080.

### 3b. Proof API Service

The Proof API config is JSON. Create `relayer.json`:

#### EVM ↔ EVM

```json
{
  "server": {
    "address": "0.0.0.0",
    "port": 8080
  },
  "modules": [
    {
      "name": "eth_to_eth",
      "src_chain": "<CHAIN_A_ID>",
      "dst_chain": "<CHAIN_B_ID>",
      "config": {
        "src_chain_id": "<CHAIN_A_ID>",
        "src_rpc_url": "<CHAIN_A_RPC>",
        "src_ics26_address": "<CHAIN_A_ICS26_ROUTER>",
        "dst_rpc_url": "<CHAIN_B_RPC>",
        "dst_ics26_address": "<CHAIN_B_ICS26_ROUTER>",
        "mode": {
          "attested": {
            "attestor": {
              "attestor_query_timeout_ms": 5000,
              "quorum_threshold": 1,
              "attestor_endpoints": ["https://<CHAIN_A_ATTESTOR_ADDR>"]
            },
            "cache": {
              "state_cache_max_entries": 100000,
              "packet_cache_max_entries": 100000
            }
          }
        }
      }
    },
    {
      "name": "eth_to_eth",
      "src_chain": "<CHAIN_B_ID>",
      "dst_chain": "<CHAIN_A_ID>",
      "config": {
        "src_chain_id": "<CHAIN_B_ID>",
        "src_rpc_url": "<CHAIN_B_RPC>",
        "src_ics26_address": "<CHAIN_B_ICS26_ROUTER>",
        "dst_rpc_url": "<CHAIN_A_RPC>",
        "dst_ics26_address": "<CHAIN_A_ICS26_ROUTER>",
        "mode": {
          "attested": {
            "attestor": {
              "attestor_query_timeout_ms": 5000,
              "quorum_threshold": 1,
              "attestor_endpoints": ["https://<CHAIN_B_ATTESTOR_ADDR>"]
            },
            "cache": {
              "state_cache_max_entries": 100000,
              "packet_cache_max_entries": 100000
            }
          }
        }
      }
    }
  ]
}
```

#### EVM ↔ Cosmos

```json
{
  "server": {
    "address": "0.0.0.0",
    "port": 8080
  },
  "modules": [
    {
      "name": "eth_to_cosmos",
      "src_chain": "<EVM_CHAIN_ID>",
      "dst_chain": "<COSMOS_CHAIN_ID>",
      "config": {
        "ics26_address": "<EVM_ICS26_ROUTER>",
        "tm_rpc_url": "<COSMOS_RPC_URL>",
        "eth_rpc_url": "<EVM_RPC_URL>",
        "signer_address": "<COSMOS_RELAYER_SUBMITTER_ADDRESS>",
        "mode": {
          "attested": {
            "attestor": {
              "attestor_query_timeout_ms": 5000,
              "quorum_threshold": 1,
              "attestor_endpoints": ["https://<EVM_ATTESTOR_ADDR>"]
            },
            "cache": {
              "state_cache_max_entries": 100000,
              "packet_cache_max_entries": 100000
            }
          }
        }
      }
    },
    {
      "name": "cosmos_to_eth",
      "src_chain": "<COSMOS_CHAIN_ID>",
      "dst_chain": "<EVM_CHAIN_ID>",
      "config": {
        "tm_rpc_url": "<COSMOS_RPC_URL>",
        "ics26_address": "<EVM_ICS26_ROUTER>",
        "eth_rpc_url": "<EVM_RPC_URL>",
        "mode": {
          "attested": {
            "attestor": {
              "attestor_query_timeout_ms": 5000,
              "quorum_threshold": 1,
              "attestor_endpoints": ["https://<COSMOS_ATTESTOR_ADDR>"]
            },
            "cache": {
              "state_cache_max_entries": 100000,
              "packet_cache_max_entries": 100000
            }
          }
        }
      }
    }
  ]
}
```

Start the Proof API:

```bash
docker run -d \
  --name proof-api \
  --network <DOCKER_NETWORK> \
  -p 8080:8080 \
  -v /path/to/relayer.json:/usr/local/relayer/relayer.json \
  ghcr.io/cosmos/proof-api:<tag>
```

Health check: the Proof API is ready when `Info(srcChain, dstChain)` returns successfully on port 8080.

---

## Phase 4: Create Light Clients & Register Counterparties

Both chains must each have a light client tracking the other chain. All light clients in this deployment use the attestation type.

The Proof API (started in Phase 3) creates the signed light client transaction. You then submit it on-chain.

### 4a. Create a Light Client on EVM (tracking Cosmos or the other EVM chain)

Call the Proof API's `CreateClient` gRPC endpoint:

```json
{
  "src_chain": "<COUNTERPARTY_CHAIN_ID>",
  "dst_chain": "<THIS_EVM_CHAIN_ID>",
  "parameters": {
    "min_required_sigs": "1",
    "height": "<COUNTERPARTY_CURRENT_HEIGHT>",
    "timestamp": "<COUNTERPARTY_BLOCK_TIMESTAMP_SEC>",
    "attestor_addresses": "<ATTESTOR_SIGNER_ADDRESS>",
    "role_manager": "<ICS26_ROUTER_ADDRESS>"
  }
}
```

The response contains a raw EVM transaction (`tx` field). Submit it:

```bash
# The tx bytes are a contract creation (no `to` address).
cast send --rpc-url <EVM_RPC_URL> --private-key <KEY> \
  --create <TX_BYTES>
```

The contract address from the receipt is the light client address. The client ID assigned by `ICS26Router` is deterministic: the first attestation client added is `attestations-0`.

Next, register the client on `ICS26Router` with its counterparty info:

```solidity
ICS26Router.addClient(
    "attestations-0",              // clientId (format: {type}-{n})
    CounterpartyInfo({
        clientId: "<COUNTERPARTY_CLIENT_ID>",   // client ID on the other chain
        merklePrefix: [""]                       // empty for EVM; ["ibc", ""] for Cosmos with SP1
    }),
    <LIGHT_CLIENT_CONTRACT_ADDRESS>
)
```

<!-- TODO: Dennis to fill in — exact cast/script invocation for addClient -->

### 4b. Create a Light Client on Cosmos (tracking EVM) — Cosmos↔EVM only

Submit a `MsgCreateClient` with an `AttestationClientState`:

```json
{
  "@type": "/ibc.core.client.v1.MsgCreateClient",
  "client_state": {
    "@type": "/ibc.lightclients.attestations.v1.ClientState",
    "attestor_addresses": ["<ATTESTOR_SIGNER_ADDRESS>"],
    "min_required_sigs": 1,
    "latest_height": {
      "revision_number": 0,
      "revision_height": <EVM_CURRENT_BLOCK_HEIGHT>
    }
  },
  "consensus_state": {
    "@type": "/ibc.lightclients.attestations.v1.ConsensusState",
    "timestamp": <EVM_BLOCK_TIMESTAMP_NS>
  },
  "signer": "<COSMOS_FAUCET_OR_ADMIN_ADDRESS>"
}
```

The chain assigns the first attestation client the ID `attestations-0`.

<!-- TODO: Dennis to fill in — CLI command or script to submit MsgCreateClient on your Cosmos chain -->

### 4c. Register Counterparties

On Cosmos, submit `MsgRegisterCounterparty`:

```json
{
  "@type": "/ibc.core.client.v2.MsgRegisterCounterparty",
  "client_id": "attestations-0",
  "merkle_prefix": [""],
  "counterparty_client_id": "<EVM_CLIENT_ID>",
  "signer": "<COSMOS_ADMIN_ADDRESS>"
}
```

On EVM, the `addClient` call in Phase 4a already includes the `CounterpartyInfo`.

---

## Phase 5: Configure IFT Bridges (EVM Side)

### 5a. EVM ↔ EVM

Call `registerIFTBridge` on each chain's IFT contract. This is restricted to the contract's `authority` (set during deployment via `AccessManager`):

```solidity
IFT.registerIFTBridge(
    "<CLIENT_ID_ON_THIS_CHAIN>",       // e.g. "attestations-0"
    "<IFT_CONTRACT_ADDRESS_ON_OTHER_CHAIN>",  // counterpartyIFTAddress
    "<EVM_IFT_CONSTRUCTOR_ADDRESS>"    // EvmIFTConstructor from the deploy output
)
```

Repeat on the second chain (swapping source/destination).

### 5b. Deploy the `CosmosIFTSendCallConstructor` (EVM↔Cosmos only)

The constructor encodes `MsgIFTMint` protobuf calls that are sent via GMP to Cosmos. It must know the ICA address on Cosmos that the EVM IFT contract controls.

Compute the GMP account address by querying the Cosmos GMP module:

```bash
grpcurl -plaintext <COSMOS_GRPC_ADDR> \
  ibc.applications.gmp.v1.Query/AccountAddress \
  -d '{"client_id": "<COSMOS_CLIENT_ID>", "sender": "<EVM_IFT_ADDRESS>", "salt": ""}'
```

The returned `account_address` is the GMP account address on Cosmos (e.g., `cosmos1c98z43jf73...`).

Deploy the constructor on EVM:

```solidity
new CosmosIFTSendCallConstructor(
    "/<COSMOS_MODULE>.ift.MsgIFTMint",   // bridgeReceiveTypeUrl
    "<IFT_DENOM>",               // e.g. "testift"
    "<ICA_ADDRESS>"              // the Cosmos GMP account address computed above
)
```

<!-- TODO: Dennis to fill in — forge script or cast command for deploying CosmosIFTSendCallConstructor -->

Register the IFT bridge on EVM:

```solidity
IFT.registerIFTBridge(
    "<L2_CLIENT_ID>",              // client ID on EVM tracking Cosmos (e.g. "attestations-0")
    "<COSMOS_IFT_MODULE_ADDRESS>", // IFT module account on Cosmos (computed below)
    "<COSMOS_IFT_CONSTRUCTOR_ADDR>"
)
```

The IFT module address on Cosmos is the module account for `"ift"`:

```bash
# Compute via Cosmos SDK:
# authtypes.NewModuleAddress("ift") → bech32-encode with chain prefix "<COSMOS_ADDRESS_PREFIX>"
# Result example: <prefix>1aj7phfrd43n9yl34hvnmm5hekwl9a78a58...
```

<!-- TODO: Dennis to fill in — CLI to derive and verify the IFT module address on your Cosmos chain -->

---

## Phase 6: Cosmos Chain Setup (EVM↔Cosmos Only)

### 6a. Create a TokenFactory Denom

IFT tokens on Cosmos are represented as TokenFactory denoms. Create the denom using the account that will administer it:

```bash
<chain-binary> tx tokenfactory create-denom <SUBDENOM> \
  --from <ADMIN_ADDRESS> \
  --chain-id <COSMOS_CHAIN_ID> \
  --fees 50000u<COSMOS_FEE_DENOM>
```

The resulting denom will be `factory/<ADMIN_ADDRESS>/<SUBDENOM>` — use the subdenom (e.g., `testift`) in the IFT bridge registration.

### 6b. Mint Initial Supply (Optional)

If you need an initial supply on Cosmos for testing:

```bash
<chain-binary> tx tokenfactory mint <AMOUNT><DENOM> \
  --from <ADMIN_ADDRESS> \
  --chain-id <COSMOS_CHAIN_ID> \
  --fees 50000u<COSMOS_FEE_DENOM>
```

### 6c. Register the IFT Bridge on Cosmos

Submit `MsgRegisterIFTBridge`. On the reference implementation chain, the IFT authority is set during chain initialization so it can sign directly. On a production chain this would typically require a governance proposal.

```json
{
  "@type": "/<COSMOS_MODULE>.ift.MsgRegisterIFTBridge",
  "signer": "<IFT_AUTHORITY_ADDRESS>",
  "denom": "<IFT_DENOM_SUBDENOM>",
  "client_id": "<COSMOS_CLIENT_ID_TRACKING_EVM>",
  "counterparty_ift_address": "<EVM_IFT_CONTRACT_ADDRESS>",
  "ift_send_call_constructor": "evm"
}
```

<!-- TODO: Dennis to fill in — CLI command or governance proposal format for MsgRegisterIFTBridge on your Cosmos chain -->

---

## Phase 7: Start Relayer

Start the relayer after Phases 4–6 are complete (light clients registered, IFT bridges configured).

### 7a. PostgreSQL Database

The relayer requires a PostgreSQL database. Run migrations before starting the relayer:

```bash
docker run --rm \
  --network <DOCKER_NETWORK> \
  <RELAYER_MIGRATE_IMAGE>:<tag> \
  -database "postgres://<DB_USER>:<DB_PASS>@<DB_HOST>:5432/<DB_NAME>?sslmode=disable" \
  up
```

See the [cosmos/ibc-relayer](https://github.com/cosmos/ibc-relayer) README for image names and tags.

### 7b. Relayer Service

Create the relayer config (`relayer.yml`):

#### EVM ↔ EVM

```yaml
postgres:
  hostname: <DB_HOST>
  port: "5432"
  database: <DB_NAME>

relayer_api:
  address: "0.0.0.0:9000"

metrics:
  prometheus_address: "0.0.0.0:8001"

ibcv2_proof_api:
  grpc_address: "<PROOF_API_CONTAINER_IP>:8080"
  grpc_tls_enabled: false

signing:
  keys_path: /mnt/relayer/relayerkeys.json

chains:
  chain-a:
    chain_name: chain-a
    chain_id: "<CHAIN_A_ID>"
    type: evm
    environment: testnet
    gas_token_symbol: ETH
    gas_token_decimals: 18
    evm:
      rpc: "<CHAIN_A_RPC>"
      contracts:
        ics_26_router_address: "<CHAIN_A_ICS26_ROUTER>"
        ics_20_transfer_address: "<CHAIN_A_ICS20_TRANSFER>"
    ibcv2:
      counterparty_chains:
        "<CHAIN_A_CLIENT_ID>": "<CHAIN_B_ID>"
      recv_batch_size: 10
      recv_batch_timeout: 5s
      recv_batch_concurrency: 2
      ack_batch_size: 10
      ack_batch_timeout: 5s
      ack_batch_concurrency: 2
      timeout_batch_size: 10
      timeout_batch_timeout: 5s
      timeout_batch_concurrency: 2
      should_relay_success_acks: true
      should_relay_error_acks: true
      finality_offset: 1

  chain-b:
    chain_name: chain-b
    chain_id: "<CHAIN_B_ID>"
    type: evm
    environment: testnet
    gas_token_symbol: ETH
    gas_token_decimals: 18
    evm:
      rpc: "<CHAIN_B_RPC>"
      contracts:
        ics_26_router_address: "<CHAIN_B_ICS26_ROUTER>"
        ics_20_transfer_address: "<CHAIN_B_ICS20_TRANSFER>"
    ibcv2:
      counterparty_chains:
        "<CHAIN_B_CLIENT_ID>": "<CHAIN_A_ID>"
      recv_batch_size: 10
      recv_batch_timeout: 5s
      recv_batch_concurrency: 2
      ack_batch_size: 10
      ack_batch_timeout: 5s
      ack_batch_concurrency: 2
      timeout_batch_size: 10
      timeout_batch_timeout: 5s
      timeout_batch_concurrency: 2
      should_relay_success_acks: true
      should_relay_error_acks: true
      finality_offset: 1
```

#### EVM ↔ Cosmos

```yaml
postgres:
  hostname: <DB_HOST>
  port: "5432"
  database: <DB_NAME>

relayer_api:
  address: "0.0.0.0:9000"

metrics:
  prometheus_address: "0.0.0.0:8001"

ibcv2_proof_api:
  grpc_address: "<PROOF_API_CONTAINER_IP>:8080"
  grpc_tls_enabled: false

signing:
  keys_path: /mnt/relayer/relayerkeys.json

chains:
  cosmos-chain:
    chain_name: <COSMOS_CHAIN_NAME>
    chain_id: <COSMOS_CHAIN_ID>
    type: cosmos
    environment: testnet
    gas_token_symbol: <COSMOS_FEE_DENOM>
    gas_token_decimals: 6
    cosmos:
      gas_price: 0.025
      rpc: "<COSMOS_RPC_URL>"           # e.g. http://<COSMOS_HOST>:26657
      grpc: "<COSMOS_GRPC_ADDR>"        # e.g. <COSMOS_HOST>:9090
      grpc_tls_enabled: false
      address_prefix: <COSMOS_ADDRESS_PREFIX>
      ibcv2_tx_fee_denom: u<COSMOS_FEE_DENOM>
    ibcv2:
      counterparty_chains:
        "<COSMOS_CLIENT_ID>": "<EVM_CHAIN_ID>"
      recv_batch_size: 10
      recv_batch_timeout: 5s
      recv_batch_concurrency: 2
      ack_batch_size: 10
      ack_batch_timeout: 5s
      ack_batch_concurrency: 2
      timeout_batch_size: 10
      timeout_batch_timeout: 5s
      timeout_batch_concurrency: 2
      should_relay_success_acks: true
      should_relay_error_acks: true
      finality_offset: 1

  l2:
    chain_name: l2
    chain_id: "<EVM_CHAIN_ID>"
    type: evm
    environment: testnet
    gas_token_symbol: ETH
    gas_token_decimals: 18
    evm:
      rpc: "<EVM_RPC_URL>"
      contracts:
        ics_26_router_address: "<EVM_ICS26_ROUTER>"
        ics_20_transfer_address: "<EVM_ICS20_TRANSFER>"
    ibcv2:
      counterparty_chains:
        "<EVM_CLIENT_ID>": "<COSMOS_CHAIN_ID>"
      recv_batch_size: 10
      recv_batch_timeout: 5s
      recv_batch_concurrency: 2
      ack_batch_size: 10
      ack_batch_timeout: 5s
      ack_batch_concurrency: 2
      timeout_batch_size: 10
      timeout_batch_timeout: 5s
      timeout_batch_concurrency: 2
      should_relay_success_acks: true
      should_relay_error_acks: true
      finality_offset: 1
```

Create the keys file (`relayerkeys.json`). The relayer uses this to sign transactions on each chain:

```json
{
  "<CHAIN_A_ID>": {
    "name": "chain-a",
    "address": "<CHAIN_A_SIGNER_ADDRESS>",
    "private_key": "0x<CHAIN_A_PRIVATE_KEY_HEX>"
  },
  "<CHAIN_B_ID>": {
    "name": "chain-b",
    "address": "<CHAIN_B_SIGNER_ADDRESS>",
    "private_key": "0x<CHAIN_B_PRIVATE_KEY_HEX>"
  }
}
```

For Cosmos chains, the private key is the secp256k1 hex key derived from the mnemonic.

Start the relayer:

```bash
docker run -d \
  --name relayer \
  --network <DOCKER_NETWORK> \
  -p 9000:9000 \
  -v /path/to/relayer.yml:/mnt/relayer/relayer.yml \
  -v /path/to/relayerkeys.json:/mnt/relayer/relayerkeys.json \
  -e POSTGRES_USER=<DB_USER> \
  -e POSTGRES_PASSWORD=<DB_PASS> \
  <RELAYER_IMAGE>:<tag> \
  --config /mnt/relayer/relayer.yml
```

See the [cosmos/ibc-relayer](https://github.com/cosmos/ibc-relayer) README for image names and tags.

Health check: `GET http://localhost:9000/health` should return 200.

> Note: When using a remote signer (production), set `signing.grpc_address` in the relayer config instead of `signing.keys_path`. In that case, skip the keys file and set `SERVICE_ACCOUNT_TOKEN` in the environment.

<!-- TODO: Dennis to fill in — remote signer configuration for the relayer -->

---

## Phase 8: Verify the Setup

### 8a. Mint IFT Tokens on the Source Chain

EVM source:

```solidity
// Only the IFT contract authority can mint
IFT.mint(<RECIPIENT_ADDRESS>, <AMOUNT>)
```

Cosmos source (if starting from Cosmos):

IFT tokens on Cosmos are backed by the TokenFactory denom. Ensure the recipient has a balance (see [Phase 6b](#6b-mint-initial-supply-optional)).

### 8b. Initiate an IFT Transfer

EVM → EVM or EVM → Cosmos:

```solidity
IFT.iftTransfer(
    "<CLIENT_ID>",           // source client ID (tracks destination chain)
    "<RECEIVER_ADDRESS>",    // receiver on destination (hex for EVM, bech32 for Cosmos)
    <AMOUNT>,
    <TIMEOUT_TIMESTAMP_UNIX> // Unix timestamp; must be in the future
)
```

Record the transaction hash.

Cosmos → EVM:

```bash
<chain-binary> tx ift transfer \
  --denom <IFT_DENOM> \
  --client-id <COSMOS_CLIENT_ID> \
  --receiver <EVM_ADDRESS_HEX> \
  --amount <AMOUNT> \
  --timeout-timestamp <UNIX_TIMESTAMP> \
  --from <SENDER_ADDRESS> \
  --chain-id <COSMOS_CHAIN_ID> \
  --fees 50000u<COSMOS_FEE_DENOM>
```

<!-- TODO: Dennis to fill in — exact CLI subcommand for IFT transfer on your Cosmos chain -->

### 8c. Relay the Packet

Call the relayer's `Relay` endpoint with the transaction hash and source chain ID:

```bash
grpcurl -plaintext localhost:9000 \
  relayerapi.RelayerApiService/Relay \
  -d '{"tx_hash": "<TX_HASH>", "chain_id": "<SOURCE_CHAIN_ID>"}'
```

Or via the REST API:

```bash
curl -X POST http://localhost:9000/relay \
  -H 'Content-Type: application/json' \
  -d '{"tx_hash": "<TX_HASH>", "chain_id": "<SOURCE_CHAIN_ID>"}'
```

### 8d. Poll for Completion

```bash
grpcurl -plaintext localhost:9000 \
  relayerapi.RelayerApiService/RelayStatus \
  -d '{"tx_hash": "<TX_HASH>", "chain_id": "<SOURCE_CHAIN_ID>"}'
```

A successful relay shows `"state": "TRANSFER_STATE_COMPLETE"` with a `recv_tx` containing the destination chain transaction hash.

### 8e. Verify Balances

EVM destination:

```solidity
IFT.balanceOf(<RECEIVER_ADDRESS>)
```

Cosmos destination:

```bash
<chain-binary> query bank balances <RECEIVER_ADDRESS> --node <COSMOS_RPC>
```

The balance should reflect the transferred amount.

---

## Configuration Reference

### Key IDs & Addresses

| Item | Value | Notes |
|------|-------|-------|
| First attestation client ID (Cosmos) | `attestations-0` | Assigned by ibc-go |
| First attestation client ID (EVM) | `attestations-0` | Assigned by ICS26Router |
| IFT module address (Cosmos) | chain-specific bech32 address | `authtypes.NewModuleAddress("ift")` encoded with your chain's prefix |
| IFT mint type URL (Cosmos) | `/<COSMOS_MODULE>.ift.MsgIFTMint` | Module path varies by chain |
| Merkle prefix (EVM counterparty) | `[""]` | Empty string, no store prefix |
| Merkle prefix (Cosmos counterparty) | `[""]` | For attestation clients |

### Service Ports

| Service | Port | Protocol |
|---------|------|----------|
| Attestor | 8080 | gRPC (HTTP/2) |
| Proof API | 8080 | gRPC |
| Proof API (gRPC-Web) | 8081 | HTTP |
| Relayer API | 9000 | REST + gRPC |
| Relayer Metrics | 8001 | Prometheus |

### IFT Bridge Registration Summary

| Step | Chain | Call | Signer |
|------|-------|------|--------|
| Register bridge | EVM | `IFT.registerIFTBridge(clientId, counterpartyAddr, constructorAddr)` | Authority (AccessManager) |
| Register bridge | Cosmos | `MsgRegisterIFTBridge` | IFT authority (governance on mainnet) |
| Create denom | Cosmos | `MsgCreateDenom` | Admin address |

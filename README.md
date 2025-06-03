# NEAR Core Contracts

This repository contains the core smart contracts that power essential functionality in the NEAR ecosystem. These contracts handle token lockups, multi-signature wallets, staking delegation, governance voting, and other critical infrastructure components.

## Smart Contracts Overview

### Core Functionality Contracts

#### [Lockup / Vesting Contract](./lockup/)
A comprehensive escrow contract that locks and holds tokens for specified lockup periods with linear unlocking schedules. Supports both lockup mechanisms (time-based token release) and vesting schedules (employment-based with cliff periods and termination capabilities). Essential for token distribution, employee compensation, and investor agreements.

#### [Multisig Contract](./multisig/)
A basic K-of-N multi-signature wallet implementation using NEAR access keys. Allows multiple parties to collectively control an account by requiring a specified number of confirmations before executing transactions, transfers, or contract calls.

#### [Multisig2 Contract](./multisig2/)
An enhanced multi-signature contract that supports both access keys and account IDs as signers. Provides more flexible member management compared to the basic multisig contract.

#### [Staking Pool Contract](./staking-pool/)
Enables token holders to delegate their NEAR tokens to validators through a pooled staking mechanism. Delegators earn staking rewards (minus fees) while helping secure the network. Implements the NEAR staking pool standard for secure delegation.

#### [Voting Contract](./voting/)
A governance contract specifically designed for validators to vote on network-wide decisions, such as enabling token transfers. Requires a 2/3 majority of staked tokens to pass proposals.

#### [Whitelist Contract](./whitelist/)
Maintains an approved list of staking pool contracts that are authorized to receive delegated tokens from lockup contracts. Ensures that lockup contracts can only delegate to verified, secure staking pools that implement proper recovery mechanisms.

### Factory Contracts

#### [Lockup Factory](./lockup-factory/)
Enables permissionless creation of lockup contracts. Any user can deploy and fund new lockup contracts without requiring foundation keys, making the lockup system more decentralized and accessible.

#### [Staking Pool Factory](./staking-pool-factory/)
Allows users to create new staking pool contracts that are automatically whitelisted. Streamlines the process of setting up validator staking pools while maintaining security through automatic whitelist integration.

#### [Multisig Factory](./multisig-factory/)
Simplifies the deployment of multisig contracts by providing a factory interface. Users can create new multisig wallets with specified configurations without manually deploying contract code.

### Utility Contracts

#### [W-NEAR Contract](./w-near/)
A wrapped NEAR token contract that provides ERC-20-like functionality for NEAR tokens, enabling integration with DeFi protocols and cross-chain applications.

#### [State Manipulation Contract](./state-manipulation/)
A utility contract for advanced state management operations. Allows authorized modification of contract storage, including adding and removing key-value pairs. Primarily used for contract migrations and state corrections.

## Building and Deploying

### Prerequisites

Before building and deploying these contracts, ensure you have the latest NEAR development tools installed:

1. **Install Rust and NEAR toolchain:**
   ```bash
   # Install Rust
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   source ~/.cargo/env

   # Add WebAssembly target
   rustup target add wasm32-unknown-unknown

   # Install cargo-near for optimized builds
   cargo install cargo-near
   ```

2. **Install NEAR CLI:**
   ```bash
   # Install the latest NEAR CLI (Rust version - recommended)
   cargo install near-cli-rs

   # Or install the Node.js version
   npm install -g near-cli
   ```

### Building Contracts

#### Option 1: Using cargo-near (Recommended)

For individual contracts:
```bash
cd lockup/
cargo near build
```

For all contracts:
```bash
./scripts/build_all.sh
```

#### Option 2: Using Docker

To build all contracts in a consistent environment:
```bash
./scripts/build_all_docker.sh
```

### Deploying Contracts

#### Using NEAR CLI (Rust version)

1. **Login to your NEAR account:**
   ```bash
   near account import-account using-web-wallet network-config testnet
   ```

2. **Deploy a contract:**
   ```bash
   near contract deploy <contract-account-id> use-file res/contract.wasm with-init-call new json-args '{"arg1": "value1"}' prepaid-gas '100.0 Tgas' attached-deposit '0 NEAR' network-config testnet sign-with-keychain send
   ```

#### Using NEAR CLI (Node.js version)

1. **Login to your NEAR account:**
   ```bash
   near login
   ```

2. **Deploy a contract:**
   ```bash
   near deploy --accountId <contract-account-id> --wasmFile res/contract.wasm
   ```

3. **Initialize the contract:**
   ```bash
   near call <contract-account-id> new '{"arg1": "value1"}' --accountId <your-account-id>
   ```

### Automated Deployment Scripts

For deploying core infrastructure contracts, see the [scripts](./scripts/) folder which includes:

- `deploy_core.sh` - Deploys voting, whitelist, and staking pool factory contracts
- `deploy_lockup.sh` - Interactive script for deploying lockup contracts
- `build_all.sh` - Builds all contracts in the repository

#### Core Contract Deployment

Set up environment variables:
```bash
export MASTER_ACCOUNT_ID=your-account.testnet
export NEAR_ENV=testnet
```

Deploy core contracts (requires ~80 NEAR + gas fees):
```bash
./scripts/deploy_core.sh
```

### Contract Initialization Examples

When deploying contracts, the account creation, contract deployment, and initialization should be done atomically. Here's an example for a lockup contract:

#### Using NEAR CLI (Rust version)
```bash
near contract deploy lockup-contract.testnet use-file res/lockup_contract.wasm with-init-call new json-args '{
  "lockup_duration": "31536000000000000",
  "lockup_start_information": {
    "TransfersDisabled": {
      "transfer_poll_account_id": "transfers-poll.testnet"
    }
  },
  "vesting_schedule": {
    "start_timestamp": "1535760000000000000",
    "cliff_timestamp": "1567296000000000000",
    "end_timestamp": "1661990400000000000"
  },
  "staking_pool_whitelist_account_id": "whitelist.testnet",
  "initial_owners_main_public_key": "ed25519:...",
  "foundation_account_id": "foundation.testnet"
}' prepaid-gas '100.0 Tgas' attached-deposit '100 NEAR' network-config testnet sign-with-keychain send
```

#### Using JavaScript/TypeScript

For programmatic deployment, you can use the NEAR JavaScript SDK:

```typescript
import { Near, Account, keyStores } from 'near-api-js';
import fs from 'fs';

const near = new Near({
  networkId: 'testnet',
  keyStore: new keyStores.FileSystemKeyStore(),
  nodeUrl: 'https://rpc.testnet.near.org',
  walletUrl: 'https://wallet.testnet.near.org',
});

const account = await near.account('your-account.testnet');
const contractWasm = fs.readFileSync('res/lockup_contract.wasm');

const result = await account.signAndSendTransaction({
  receiverId: 'lockup-contract.testnet',
  actions: [
    {
      type: 'CreateAccount'
    },
    {
      type: 'Transfer',
      params: { deposit: '100000000000000000000000000' } // 100 NEAR
    },
    {
      type: 'DeployContract',
      params: { code: contractWasm }
    },
    {
      type: 'FunctionCall',
      params: {
        methodName: 'new',
        args: {
          lockup_duration: '31536000000000000',
          // ... other initialization args
        },
        gas: '100000000000000',
        deposit: '0'
      }
    }
  ]
});
```

### Testing

Run tests for all contracts:
```bash
./scripts/test_all.sh
```

Run tests for a specific contract:
```bash
cd lockup/
cargo test
```


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

## Building and deploying

See [scripts](./scripts/) folder for details.

## Initializing Contracts with near-shell

When setting up the contract creating the contract account, deploying the binary, and initializing the state must all be done as an atomic step.  For example, in our tests for the lockup contract we initialize it like this:

```rust
pub fn init_lockup(
        &self,
        runtime: &mut RuntimeStandalone,
        args: &InitLockupArgs,
        amount: Balance,
    ) -> TxResult {
        let tx = self
            .new_tx(runtime, LOCKUP_ACCOUNT_ID.into())
            .create_account()
            .transfer(ntoy(35) + amount)
            .deploy_contract(LOCKUP_WASM_BYTES.to_vec())
            .function_call(
                "new".into(),
                serde_json::to_vec(args).unwrap(),
                200000000000000,
                0,
            )
            .sign(&self.signer);
        let res = runtime.resolve_tx(tx).unwrap();
        runtime.process_all().unwrap();
        outcome_into_result(res)
    }
```


To do this with near shell, first add a script like `deploy.js`:

```js
const fs = require('fs');
const account = await near.account("foundation");
const contractName = "lockup-owner-id";
const newArgs = {
  "lockup_duration": "31536000000000000",
  "lockup_start_information": {
    "TransfersDisabled": {
        "transfer_poll_account_id": "transfers-poll"
    }
  },
  "vesting_schedule": {
    "start_timestamp": "1535760000000000000",
    "cliff_timestamp": "1567296000000000000",
    "end_timestamp": "1661990400000000000"
  },
  "staking_pool_whitelist_account_id": "staking-pool-whitelist",
  "initial_owners_main_public_key": "KuTCtARNzxZQ3YvXDeLjx83FDqxv2SdQTSbiq876zR7",
  "foundation_account_id": "near"
}
const result = account.signAndSendTransaction(
    contractName,
    [
        nearAPI.transactions.createAccount(),
        nearAPI.transactions.transfer("100000000000000000000000000"),
        nearAPI.transactions.deployContract(fs.readFileSync("res/lockup_contract.wasm")),
        nearAPI.transactions.functionCall("new", Buffer.from(JSON.stringify(newArgs)), 100000000000000, "0"),
    ]);
```

Then use the `near repl` command. Once at the command prompt, load the script:

```js
> .load deploy.js
```

Note: `nearAPI` and `near` are both preloaded to the repl's context.

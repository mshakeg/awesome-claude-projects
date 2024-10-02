# rewards_pool Move Example

The following rewards_pool example project was taken from the official move-examples folder located in `aptos-core/aptos-move/move-examples/rewards_pool`

This example project has the following structure:

.
├── Move.toml
└── sources
    ├── epoch.move
    ├── rewards_pool.move
    └── tests
        ├── rewards_pool_tests.move
        └── test_helpers.move

The contents of all of the .move files in this project are as follows:

The move files in the `sources` folder:

## epoch.move

This file implements a simplified epoch system for the rewards_pool project, building upon the Aptos blockchain's native timekeeping capabilities. An epoch represents a fixed time period, which in this case is set to one day (86400 seconds). The module provides essential functionality for epoch-based time management, crucial for the rewards distribution system.

Key features and Aptos framework interactions:

1. Constant epoch duration: Defines EPOCH_DURATION as 86400 seconds (1 day).

2. Current epoch calculation: The `now()` function calculates the current epoch number by calling `aptos_framework::timestamp::now_seconds()` and dividing by the epoch duration. This leverages Aptos' native timestamp functionality to ensure accurate and consensus-aligned timekeeping.

3. Epoch-to-timestamp conversion: Provides functions to convert between epoch numbers and timestamps, always maintaining alignment with Aptos' internal clock.

4. Test-only time advancement: Includes a `fast_forward()` function that uses `aptos_framework::timestamp::fast_forward_seconds()` to simulate the passage of time in tests. This function is crucial for testing time-dependent behaviors without waiting for actual time to pass.

Aptos framework timestamp functions used:

- `timestamp::now_seconds()`: Returns the current Unix timestamp of the blockchain.
- `timestamp::fast_forward_seconds()`: A test-only function that advances the blockchain's timestamp by a specified number of seconds.

Additional test-only functions available in aptos_framework::timestamp:

- `update_global_time_for_test(timestamp_microsecs: u64)`: Allows setting the global time to a specific microsecond timestamp for testing purposes.
- `update_global_time_for_test_secs(timestamp_seconds: u64)`: Similar to the above, but takes the time in seconds for convenience.

These additional functions provide more granular control over time in test environments, allowing developers to set specific timestamps or advance time by precise amounts. While not directly used in this module, they are valuable tools for comprehensive testing of time-dependent systems built on Aptos.

This module demonstrates how to build higher-level time concepts (epochs) on top of Aptos' basic timekeeping, as well as how to leverage test-only framework functions for thorough testing of time-dependent systems. The availability of various time manipulation functions in tests enables developers to create robust, time-aware smart contracts and thoroughly test them under different temporal conditions.

```
module rewards_pool::epoch {
    use aptos_framework::timestamp;

    /// The epoch duration is fixed at 1 day (in seconds).
    const EPOCH_DURATION: u64 = 86400;

    #[view]
    public fun now(): u64 {
        to_epoch(timestamp::now_seconds())
    }

    public inline fun duration(): u64 {
        // Equal to EPOCH_DURATION. Inline functions cannot use constants defined in their module.
        86400
    }

    public inline fun to_epoch(timestamp_secs: u64): u64 {
        // Equal to EPOCH_DURATION. Inline functions cannot use constants defined in their module.
        timestamp_secs / 86400
    }

    public inline fun to_seconds(epoch: u64): u64 {
        // Equal to EPOCH_DURATION. Inline functions cannot use constants defined in their module.
        epoch * 86400
    }

    #[test_only]
    public fun fast_forward(epochs: u64) {
        aptos_framework::timestamp::fast_forward_seconds(epochs * EPOCH_DURATION);
    }
}
```

## rewards_pool.move

This file implements the core functionality of a multi-token rewards distribution system using the Aptos blockchain's Object model and Fungible Asset standard. It demonstrates advanced usage of Aptos Move features to create a flexible and efficient rewards pool mechanism.

Key features and Aptos framework interactions:

1. Multi-token Rewards: Supports distributing rewards in multiple token types simultaneously.

2. Epoch-based Distribution: Utilizes the epoch system (from epoch.move) to manage time-based reward distributions.

3. Object Model Usage: Leverages Aptos' Object model for representing the rewards pool and associated data structures.

4. Fungible Asset Integration: Uses the Fungible Asset standard for handling various token types, allowing for greater flexibility compared to the traditional Coin standard.

5. Dynamic Allocation: Allows for dynamic adjustment of user allocations within epochs.

6. Claiming Mechanism: Implements a rewards claiming system that accurately calculates and distributes rewards based on user allocations.

Important Aptos framework modules and functions used:

- `aptos_framework::object`: For creating and managing Move objects.
- `aptos_framework::fungible_asset`: For handling multiple token types as fungible assets.
- `aptos_std::pool_u64_unbound`: For managing allocation shares.
- `aptos_framework::primary_fungible_store`: For interacting with users' fungible asset stores.

Key data structures:

- `RewardsPool`: The main object representing the rewards pool.
- `EpochRewards`: Stores reward data for each epoch.
- `RewardStore`: Manages the storage of reward tokens.

Important functions:

- `create`: Creates a new rewards pool with specified reward tokens.
- `add_rewards`: Adds rewards to the pool for a specific epoch.
- `increase_allocation` and `decrease_allocation`: Adjust a user's allocation for the current epoch.
- `claim_rewards`: Allows users to claim their rewards for past epochs.

This module showcases advanced Move programming techniques, including:
- Use of generic types for flexibility.
- Friend functions for controlled access to critical operations.
- Complex mathematical calculations for reward distribution.
- Extensive use of Move's type system for safety and correctness.

The rewards_pool module is designed to be integrated into larger systems that manage user interactions, token flows, and overall reward strategies. It provides a robust foundation for building sophisticated reward distribution mechanisms on the Aptos blockchain.

```
/// An example module that manages rewards for multiple tokens on a per-epoch basis. Rewards can be added for multiple
/// tokens by anyone for any epoch but only friended modules can increase/decrease claimer shares.
///
/// This module is designed to be integrated into a complete system that manages epochs and rewards.
///
/// The flow works as below:
/// 1. A rewards pool is created with a set of reward tokens (fungible assets). If coins are to be used as rewards,
/// developers can use the coin_wrapper module from move-examples/swap to convert coins into fungible assets.
/// 2. Anyone can add rewards to the pool for any epoch for multiple tokens by calling add_rewards.
/// 3. Friended modules can increase/decrease claimer shares for current epoch by calling increase_allocation and
/// decrease_allocation.
/// 4. Claimers can claim their rewards in all tokens for any epoch that has ended by calling claim_rewards, which
/// return a vector of all the rewards. Claiming also removes the claimer's shares from that epoch's rewards as their
/// rewards have all been claimed.
///
/// Although claimers have to be signers, this module can be easily modified to support objects (e.g. NFTs) as claimers.
module rewards_pool::rewards_pool {
    use aptos_framework::fungible_asset::{Self, FungibleAsset, FungibleStore, Metadata};
    use aptos_framework::primary_fungible_store;
    use aptos_framework::object::{Self, Object, ExtendRef};
    use aptos_std::pool_u64_unbound::{Self as pool_u64, Pool};
    use aptos_std::simple_map::{Self, SimpleMap};
    use aptos_std::smart_table::{Self, SmartTable};

    use rewards_pool::epoch;

    use std::signer;
    use std::vector;

    /// Rewards can only be claimed for epochs that have ended.
    const EREWARDS_CANNOT_BE_CLAIMED_FOR_CURRENT_EPOCH: u64 = 1;
    /// The rewards pool does not support the given reward token type.
    const EREWARD_TOKEN_NOT_SUPPORTED: u64 = 2;

    /// Data regarding the rewards to be distributed for a specific epoch.
    struct EpochRewards has store {
        /// Total amount of rewards for each reward token added to this epoch.
        total_amounts: SimpleMap<Object<Metadata>, u64>,
        /// Pool representing the claimer shares in this epoch.
        claimer_pool: Pool,
    }

    /// Data regarding the store object for a specific reward token.
    struct RewardStore has store {
        /// The fungible store for this reward token.
        store: Object<FungibleStore>,
        /// We need to keep the fungible store's extend ref to be able to transfer rewards from it during claiming.
        store_extend_ref: ExtendRef,
    }

    #[resource_group_member(group = aptos_framework::object::ObjectGroup)]
    struct RewardsPool has key {
        /// A mapping to track per epoch rewards data.
        epoch_rewards: SmartTable<u64, EpochRewards>,
        /// The stores where rewards are kept.
        reward_stores: SimpleMap<Object<Metadata>, RewardStore>,
    }

    /// Create a new rewards pool with the given reward tokens (fungible assets only)
    public entry fun create_entry(reward_tokens: vector<Object<Metadata>>) {
        create(reward_tokens);
    }

    /// Create a new rewards pool with the given reward tokens (fungible assets only)
    public fun create(reward_tokens: vector<Object<Metadata>>): Object<RewardsPool> {
        // The owner of the object doesn't matter as there are no owner-based permissions.
        // If developers want to be extra cautious here, they can make the owner @0x0.
        // Here the reward pool also doesn't keep an ExtendRef so there would be no way to obtain its signer.
        let rewards_pool_constructor_ref = &object::create_object(@rewards_pool);
        let rewards_pool_signer = &object::generate_signer(rewards_pool_constructor_ref);
        let rewards_pool_addr = signer::address_of(rewards_pool_signer);
        let reward_stores = simple_map::new();
        vector::for_each(reward_tokens, |reward_token| {
            let reward_token: Object<Metadata> = reward_token;
            let store_constructor_ref = &object::create_object(rewards_pool_addr);
            let store = fungible_asset::create_store(store_constructor_ref, reward_token);
            simple_map::add(&mut reward_stores, reward_token, RewardStore {
                store,
                // The extend ref for the rewards store is kept so we can withdraw rewards from it later when
                // claimers claim their rewards.
                store_extend_ref: object::generate_extend_ref(store_constructor_ref),
            });
        });
        move_to(rewards_pool_signer, RewardsPool {
            epoch_rewards: smart_table::new(),
            reward_stores,
        });
        object::object_from_constructor_ref(rewards_pool_constructor_ref)
    }

    #[view]
    /// Return all the reward tokens supported by the rewards pool.
    public fun reward_tokens(rewards_pool: Object<RewardsPool>): vector<Object<Metadata>> acquires RewardsPool {
        simple_map::keys(&safe_rewards_pool_data(&rewards_pool).reward_stores)
    }

    #[view]
    /// Return the current shares and total shares of a given claimer for a given rewards pool and epoch.
    public fun claimer_shares(
        claimer: address,
        rewards_pool: Object<RewardsPool>,
        epoch: u64,
    ): (u64, u64) acquires RewardsPool {
        let epoch_rewards = smart_table::borrow(&safe_rewards_pool_data(&rewards_pool).epoch_rewards, epoch);
        let shares = (pool_u64::shares(&epoch_rewards.claimer_pool, claimer) as u64);
        let total_shares = (pool_u64::total_shares(&epoch_rewards.claimer_pool) as u64);
        (shares, total_shares)
    }

    #[view]
    /// Return the amounts of claimable rewards for a given claimer, rewards pool, and epoch.
    /// The return value is a vector of reward tokens and a vector of amounts.
    public fun claimable_rewards(
        claimer: address,
        rewards_pool: Object<RewardsPool>,
        epoch: u64,
    ): (vector<Object<Metadata>>, vector<u64>) acquires RewardsPool {
        assert!(epoch < epoch::now(), EREWARDS_CANNOT_BE_CLAIMED_FOR_CURRENT_EPOCH);
        let all_rewards_tokens = reward_tokens(rewards_pool);
        let non_empty_reward_tokens = vector[];
        let reward_per_tokens = vector[];
        let rewards_pool_data = safe_rewards_pool_data(&rewards_pool);
        vector::for_each(all_rewards_tokens, |reward_token| {
            let reward = rewards(claimer, rewards_pool_data, reward_token, epoch);
            if (reward > 0) {
                vector::push_back(&mut non_empty_reward_tokens, reward_token);
                vector::push_back(&mut reward_per_tokens, reward);
            };
        });
        (non_empty_reward_tokens, reward_per_tokens)
    }

    /// Allow a claimer to claim the rewards for a past epoch.
    /// This returns a vector of rewards for all reward tokens.
    public entry fun claim_rewards_entry(
        claimer: &signer,
        rewards_pool: Object<RewardsPool>,
        epoch: u64,
    ) acquires RewardsPool {
        let rewards = claim_rewards(claimer, rewards_pool, epoch);
        let claimer_addr = signer::address_of(claimer);
        vector::for_each_reverse(rewards, |r| primary_fungible_store::deposit(claimer_addr, r));
    }

    /// Allow a claimer to claim the rewards for all tokens for a past epoch.
    /// This returns a vector of rewards for all reward tokens in the same order as the rewards tokens.
    /// If there's no reward for a specific reward token, the corresponding returned reward asset will be of zero
    /// amount (created via fungible_asset::zero).
    public fun claim_rewards(
        claimer: &signer,
        rewards_pool: Object<RewardsPool>,
        epoch: u64,
    ): vector<FungibleAsset> acquires RewardsPool {
        assert!(epoch < epoch::now(), EREWARDS_CANNOT_BE_CLAIMED_FOR_CURRENT_EPOCH);
        let reward_tokens = reward_tokens(rewards_pool);
        let rewards = vector[];
        let claimer_addr = signer::address_of(claimer);
        let rewards_data = unchecked_mut_rewards_pool_data(&rewards_pool);
        vector::for_each(reward_tokens, |reward_token| {
            let reward = rewards(claimer_addr, rewards_data, reward_token, epoch);
            let reward_store = simple_map::borrow(&rewards_data.reward_stores, &reward_token);
            if (reward == 0) {
                vector::push_back(
                    &mut rewards,
                    fungible_asset::zero(fungible_asset::store_metadata(reward_store.store)),
                );
            } else {
                // Withdraw the reward from the corresponding store.
                let store_signer = &object::generate_signer_for_extending(&reward_store.store_extend_ref);
                vector::push_back(&mut rewards, fungible_asset::withdraw(store_signer, reward_store.store, reward));

                // Update the remaining amount of rewards for the epoch.
                let epoch_rewards = smart_table::borrow_mut(&mut rewards_data.epoch_rewards, epoch);
                let total_token_rewards = simple_map::borrow_mut(&mut epoch_rewards.total_amounts, &reward_token);
                *total_token_rewards = *total_token_rewards - reward;
            };
        });

        // Remove the claimer's allocation in the epoch as they have now claimed all rewards for that epoch.
        let epoch_rewards = smart_table::borrow_mut(&mut rewards_data.epoch_rewards, epoch);
        let all_shares = pool_u64::shares(&epoch_rewards.claimer_pool, claimer_addr);
        if (all_shares > 0) {
            pool_u64::redeem_shares(&mut epoch_rewards.claimer_pool, claimer_addr, all_shares);
        };

        rewards
    }

    /// Add rewards to the specified rewards pool. This can be called with multiple reward tokens.
    public fun add_rewards(
        rewards_pool: Object<RewardsPool>,
        fungible_assets: vector<FungibleAsset>,
        epoch: u64,
    ) acquires RewardsPool {
        let rewards_data = unchecked_mut_rewards_pool_data(&rewards_pool);
        let reward_stores = &rewards_data.reward_stores;
        vector::for_each(fungible_assets, |fa| {
            let amount = fungible_asset::amount(&fa);
            let reward_token = fungible_asset::metadata_from_asset(&fa);
            assert!(simple_map::contains_key(reward_stores, &reward_token), EREWARD_TOKEN_NOT_SUPPORTED);

            // Deposit the rewards into the corresponding store.
            let reward_store = simple_map::borrow(reward_stores, &reward_token);
            fungible_asset::deposit(reward_store.store, fa);

            // Update total amount of rewards for this token for this epoch.
            let total_amounts = &mut epoch_rewards_or_default(&mut rewards_data.epoch_rewards, epoch).total_amounts;
            if (simple_map::contains_key(total_amounts, &reward_token)) {
                let current_amount = simple_map::borrow_mut(total_amounts, &reward_token);
                *current_amount = *current_amount + amount;
            } else {
                simple_map::add(total_amounts, reward_token, amount);
            };
        });
    }

    /// This should only be called by system modules to increase the shares of a claimer for the current epoch.
    public(friend) fun increase_allocation(
        claimer: address,
        rewards_pool: Object<RewardsPool>,
        amount: u64,
    ) acquires RewardsPool {
        let epoch_rewards = &mut unchecked_mut_rewards_pool_data(&rewards_pool).epoch_rewards;
        let current_epoch_rewards = epoch_rewards_or_default(epoch_rewards, epoch::now());
        pool_u64::buy_in(&mut current_epoch_rewards.claimer_pool, claimer, amount);
    }

    /// This should only be called by system modules to decrease the shares of a claimer for the current epoch.
    public(friend) fun decrease_allocation(
        claimer: address,
        rewards_pool: Object<RewardsPool>,
        amount: u64,
    ) acquires RewardsPool {
        let epoch_rewards = &mut unchecked_mut_rewards_pool_data(&rewards_pool).epoch_rewards;
        let current_epoch_rewards = epoch_rewards_or_default(epoch_rewards, epoch::now());
        pool_u64::redeem_shares(&mut current_epoch_rewards.claimer_pool, claimer, (amount as u128));
    }

    fun rewards(
        claimer: address,
        rewards_pool_data: &RewardsPool,
        reward_token: Object<Metadata>,
        epoch: u64,
    ): u64 {
        // No rewards (in any tokens) have been added for this epoch.
        if (!smart_table::contains(&rewards_pool_data.epoch_rewards, epoch)) {
            return 0
        };
        let epoch_rewards = smart_table::borrow(&rewards_pool_data.epoch_rewards, epoch);
        // No rewards have been added for this reward token.
        if (!simple_map::contains_key(&epoch_rewards.total_amounts, &reward_token)) {
            return 0
        };

        // Return the claimer's shares of the current total rewards for the epoch.
        let total_token_rewards = *simple_map::borrow(&epoch_rewards.total_amounts, &reward_token);
        let claimer_shares = pool_u64::shares(&epoch_rewards.claimer_pool, claimer);
        pool_u64::shares_to_amount_with_total_coins(&epoch_rewards.claimer_pool, claimer_shares, total_token_rewards)
    }

    inline fun safe_rewards_pool_data(
        rewards_pool: &Object<RewardsPool>,
    ): &RewardsPool acquires RewardsPool {
        borrow_global<RewardsPool>(object::object_address(rewards_pool))
    }

    inline fun epoch_rewards_or_default(
        epoch_rewards: &mut SmartTable<u64, EpochRewards>,
        epoch: u64,
    ): &mut EpochRewards acquires RewardsPool {
        if (!smart_table::contains(epoch_rewards, epoch)) {
            smart_table::add(epoch_rewards, epoch, EpochRewards {
                total_amounts: simple_map::new(),
                claimer_pool: pool_u64::create(),
            });
        };
        smart_table::borrow_mut(epoch_rewards, epoch)
    }

    inline fun unchecked_mut_rewards_pool_data(
        rewards_pool: &Object<RewardsPool>,
    ): &mut RewardsPool acquires RewardsPool {
        borrow_global_mut<RewardsPool>(object::object_address(rewards_pool))
    }

    #[test_only]
    friend rewards_pool::rewards_pool_tests;
}
```

---

The move files in the `sources/tests` folder:

## test_helpers.move

This file contains essential utility functions and setup procedures designed to create a comprehensive testing environment for the rewards_pool module and related components. It demonstrates best practices for setting up robust, controlled test scenarios in Aptos Move.

Key features and purposes:

1. Test Environment Initialization:
   - Utilizes `timestamp::set_time_has_started_for_testing` to initialize the blockchain's time system, simulating the genesis process in a test environment. This function sets the initial timestamp to 0 microseconds.
   - Sets up the initial blockchain timestamp and advances epochs to create a realistic starting state for tests.

2. Asset Creation: Provides utility functions for creating both fungible assets and coins, demonstrating flexibility in working with different token standards.

3. Test Cleanup: Includes a function to properly dispose of unused assets after tests, ensuring a clean state between test runs.

4. Aptos Framework Interactions: Showcases how to correctly interact with various Aptos framework modules in a testing context, particularly focusing on time-related functions and asset creation.

Important Aptos framework modules and functions used:

- `aptos_framework::timestamp`:
  - `set_time_has_started_for_testing`: Initializes the `CurrentTimeMicroseconds` resource, crucial for time-based operations in tests.
  - Other time manipulation functions for advancing the blockchain's clock.
- `aptos_framework::account`: For creating test accounts and signers.
- `aptos_framework::coin`: For creating and managing test coins.
- `aptos_framework::fungible_asset`: For creating and managing test fungible assets.
- `aptos_framework::object`: For working with the Object model in tests.
- `aptos_framework::primary_fungible_store`: For interacting with fungible asset stores in tests.

Key functions:

- `set_up()`:
  - Calls `timestamp::set_time_has_started_for_testing` to initialize the time system with a timestamp of 0.
  - Sets the initial blockchain timestamp and advances epochs.
  - Ensures a consistent starting point for all time-dependent tests.
- `clean_up()`: Disposes of unused assets after tests.
- `create_fungible_asset_and_mint()`: Creates a new fungible asset and mints a specified amount.
- `create_coin_and_mint()`: Creates a new coin type and mints a specified amount.

This module exemplifies several critical testing concepts in Aptos Move:

1. Proper Initialization: Ensures the blockchain's time system and other essential components are correctly set up for testing.
2. Isolation: Each test can start with a clean, controlled environment, including a properly initialized time system.
3. Flexibility: Supports testing with both fungible assets and traditional coins.
4. Reusability: Encapsulates common setup and teardown procedures for use across multiple test cases.
5. Framework Interaction: Demonstrates how to properly interact with Aptos framework modules in a testing context, especially for time-sensitive operations.

The test_helpers module is fundamental in creating a realistic and controlled testing environment. By properly initializing the time system and providing utilities for asset creation and management, it enables thorough and consistent testing of the rewards_pool and related modules, ensuring that time-dependent and asset-related functionalities can be accurately verified in a test setting.

```
#[test_only]
module rewards_pool::test_helpers {
    use aptos_framework::account;
    use aptos_framework::aptos_governance;
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::fungible_asset::{Self, FungibleAsset};
    use aptos_framework::object;
    use aptos_framework::primary_fungible_store;
    use aptos_framework::timestamp;

    use rewards_pool::epoch;

    use std::features;
    use std::option;
    use std::string;
    use std::vector;

    public fun set_up() {
        timestamp::set_time_has_started_for_testing(&account::create_signer_for_test(@0x1));
        epoch::fast_forward(100);
    }

    public fun clean_up(assets: vector<FungibleAsset>) {
        vector::for_each(assets, |a| primary_fungible_store::deposit(@0x0, a));
    }

    public fun create_fungible_asset_and_mint(creator: &signer, name: vector<u8>, amount: u64): FungibleAsset {
        let token_metadata = &object::create_named_object(creator, name);
        primary_fungible_store::create_primary_store_enabled_fungible_asset(
            token_metadata,
            option::none(),
            string::utf8(name),
            string::utf8(name),
            8,
            string::utf8(b""),
            string::utf8(b""),
        );
        let mint_ref = &fungible_asset::generate_mint_ref(token_metadata);
        fungible_asset::mint(mint_ref, amount)
    }

    public fun create_coin_and_mint<CoinType>(creator: &signer, amount: u64): Coin<CoinType> {
        let (burn_cap, freeze_cap, mint_cap) = coin::initialize<CoinType>(
            creator,
            string::utf8(b"Test"),
            string::utf8(b"Test"),
            8,
            true,
        );
        let coin = coin::mint<CoinType>(amount, &mint_cap);
        coin::destroy_burn_cap(burn_cap);
        coin::destroy_freeze_cap(freeze_cap);
        coin::destroy_mint_cap(mint_cap);
        coin
    }
}
```

## rewards_pool_tests.move

This file contains comprehensive unit tests for the rewards_pool module, demonstrating best practices for testing complex, time-dependent smart contract systems on the Aptos blockchain. It showcases how to leverage the test helpers and Aptos framework features to create thorough, multi-scenario test cases.

Key features and testing approaches:

1. Test Function Declaration:
   - Each test function is annotated with the #[test] annotations, allowing the Aptos Move test runner to identify and execute these as tests.
   - Test functions follow a clear naming convention, prefixed with "test_" (e.g., test_e2e, test_claim_with_rounding_down), enhancing readability and purpose clarity.

2. End-to-End (E2E) Testing:
   - Implements a detailed E2E test (test_e2e) that simulates multiple epochs of reward distribution, allocation changes, and claims.
   - Demonstrates how to test complex user interactions and state changes over time.

3. Edge Case Testing:
   - Includes specific tests for scenarios like rounding errors in reward distribution (test_claim_with_rounding_down) and claiming zero rewards (test_claim_zero_rewards).
   - Tests failure cases such as attempting to claim rewards for unsupported tokens or current epochs.

4. Time Manipulation:
   - Utilizes `epoch::fast_forward` to simulate the passage of time between operations, crucial for testing epoch-based functionalities.

5. Multi-User Scenarios:
   - Creates multiple test signers (e.g., claimer_1, claimer_2, claimer_3) to simulate different users interacting with the rewards pool.
   - Tests interactions between users, including the effects of one user's actions on others' rewards.

6. Detailed State Verification:
   - Implements helper functions like verify_claimer_shares_percentage and claim_and_verify_rewards to verify specific aspects of the system's state.
   - These functions ensure thorough checking of system state after each operation, including exact share percentages and reward amounts.

7. Use of Test Helpers:
   - Leverages functions from test_helpers.move for setting up the test environment (set_up) and creating test assets.
   - Utilizes clean_up function to dispose of unused assets after tests, ensuring a clean state between runs.

8. Negative Testing:
   - Includes tests with #[expected_failure] annotations (e.g., test_cannot_add_reward_for_unsupported_tokens) to verify that the system correctly handles and rejects invalid operations.

9. Test Data Structure:
    - Constructs test scenarios with specific reward amounts and user allocations.
    - For example, in test_e2e, it sets up scenarios with precise reward distributions (e.g., 50, 100) and allocation percentages (e.g., 60%, 40%).

Key test scenarios covered:

1. Basic reward distribution and claiming across multiple epochs
2. Handling of rounding errors in reward calculations with odd number of rewards
3. Claiming zero rewards when no rewards are available
4. Attempting to claim rewards twice
5. Adding rewards for unsupported tokens (negative test)
6. Attempting to claim rewards for the current epoch (negative test)

This test file demonstrates several important testing principles for Aptos Move:

1. Comprehensive Coverage: Tests cover a wide range of scenarios, including happy paths, edge cases, and error conditions.
2. State Isolation: Each test function starts with a clean state, ensuring test independence.
3. Time Simulation: Proper use of time advancement functions to test time-dependent behaviors.
4. Detailed Assertions: Extensive use of assert statements to verify precise system states and outputs.
5. Modular Testing: Breaks down complex scenarios into smaller, focused test functions.

The rewards_pool_tests module plays a crucial role in ensuring the correctness and robustness of the rewards_pool system. It provides confidence in the system's ability to handle various real-world scenarios and edge cases, which is essential for deploying reliable smart contracts on the Aptos blockchain.

```
#[test_only]
module rewards_pool::rewards_pool_tests {
    use aptos_framework::fungible_asset::{Self, FungibleAsset};
    use aptos_framework::object::Object;
    use aptos_framework::primary_fungible_store;
    use aptos_std::simple_map;
    use rewards_pool::rewards_pool::{Self, RewardsPool};
    use rewards_pool::test_helpers;
    use rewards_pool::epoch;
    use std::signer;
    use std::vector;

    #[test(claimer_1 = @0xdead, claimer_2 = @0xbeef, claimer_3 = @0xfeed)]
    fun test_e2e(claimer_1: &signer, claimer_2: &signer, claimer_3: &signer) {
        test_helpers::set_up();
        // Create a rewards pool with 1 native fungible asset and 1 coin.
        let asset_rewards_1 = test_helpers::create_fungible_asset_and_mint(claimer_1, b"test1", 1000);
        let asset_rewards_2 = test_helpers::create_fungible_asset_and_mint(claimer_1, b"test2", 2000);
        let asset_1 = fungible_asset::asset_metadata(&asset_rewards_1);
        let asset_2 = fungible_asset::asset_metadata(&asset_rewards_2);
        let rewards_pool = rewards_pool::create(vector[asset_1, asset_2]);
        assert!(rewards_pool::reward_tokens(rewards_pool) == vector[asset_1, asset_2], 0);

        // First epoch, claimers 1 and 2 split the rewards.
        increase_alocation(claimer_1, rewards_pool, 60);
        increase_alocation(claimer_2, rewards_pool, 40);
        epoch::fast_forward(1);
        add_rewards(rewards_pool, &mut asset_rewards_1, &mut asset_rewards_2, 50, 100);
        add_rewards(rewards_pool, &mut asset_rewards_1, &mut asset_rewards_2, 50, 100);
        verify_claimer_shares_percentage(claimer_1, rewards_pool, 100, 60);
        verify_claimer_shares_percentage(claimer_2, rewards_pool, 100, 40);
        claim_and_verify_rewards(claimer_1, rewards_pool, 100, vector[60, 120]);
        claim_and_verify_rewards(claimer_2, rewards_pool, 100, vector[40, 80]);
        // Claimers have claimed their rewards, so their shares are now 0.
        verify_claimer_shares_percentage(claimer_1, rewards_pool, 100, 0);
        verify_claimer_shares_percentage(claimer_2, rewards_pool, 100, 0);

        // Second epoch, there are 3 claimers, one of them has allocation decreased.
        increase_alocation(claimer_1, rewards_pool, 30);
        increase_alocation(claimer_2, rewards_pool, 50);
        increase_alocation(claimer_3, rewards_pool, 30);
        decrease_alocation(claimer_3, rewards_pool, 10);
        verify_claimer_shares_percentage(claimer_1, rewards_pool, 101, 30);
        verify_claimer_shares_percentage(claimer_2, rewards_pool, 101, 50);
        verify_claimer_shares_percentage(claimer_3, rewards_pool, 101, 20);
        epoch::fast_forward(1);
        add_rewards(rewards_pool, &mut asset_rewards_1, &mut asset_rewards_2, 100, 200);
        claim_and_verify_rewards(claimer_1, rewards_pool,101, vector[30, 60]);
        claim_and_verify_rewards(claimer_2, rewards_pool, 101, vector[50, 100]);
        claim_and_verify_rewards(claimer_3, rewards_pool, 101, vector[20, 40]);
        // Claimers have claimed their rewards, so their shares are now 0.
        verify_claimer_shares_percentage(claimer_1, rewards_pool, 101, 0);
        verify_claimer_shares_percentage(claimer_2, rewards_pool, 101, 0);

        test_helpers::clean_up(vector[asset_rewards_1, asset_rewards_2]);
    }

    #[test(claimer_1 = @0xdead, claimer_2 = @0xbeef)]
    fun test_claim_with_rounding_down(claimer_1: &signer, claimer_2: &signer) {
        test_helpers::set_up();
        // Create a rewards pool with 1 native fungible asset and 1 coin.
        let asset_rewards_1 = test_helpers::create_fungible_asset_and_mint(claimer_1, b"test1", 1000);
        let asset_rewards_2 = test_helpers::create_fungible_asset_and_mint(claimer_1, b"test2", 2000);
        let asset_1 = fungible_asset::asset_metadata(&asset_rewards_1);
        let asset_2 = fungible_asset::asset_metadata(&asset_rewards_2);
        let rewards_pool = rewards_pool::create(vector[asset_1, asset_2]);

        // Claimers 1 and 2 split the rewards but there's rounding error as the rewards are odd.
        increase_alocation(claimer_1, rewards_pool, 50);
        increase_alocation(claimer_2, rewards_pool, 50);
        verify_claimer_shares_percentage(claimer_1, rewards_pool, 100, 50);
        verify_claimer_shares_percentage(claimer_2, rewards_pool, 100, 50);
        epoch::fast_forward(1);
        // There's no rewards in the second reward token so both claimers should receive 0 there.
        add_rewards(rewards_pool, &mut asset_rewards_1, &mut asset_rewards_2, 9, 0);
        // Claimer 1 only gets 4 (50% of 9 rounded down).
        claim_and_verify_rewards(claimer_1, rewards_pool, 100, vector[4, 0]);
        // Last claimer also gets rounded up so they get an extra unit.
        claim_and_verify_rewards(claimer_2, rewards_pool, 100, vector[5, 0]);
        // Claimers have claimed their rewards, so their shares are now 0.
        verify_claimer_shares_percentage(claimer_1, rewards_pool, 100, 0);
        verify_claimer_shares_percentage(claimer_2, rewards_pool, 100, 0);

        test_helpers::clean_up(vector[asset_rewards_1, asset_rewards_2]);
    }

    #[test(claimer = @0xdead)]
    fun test_claim_zero_rewards(claimer: &signer) {
        test_helpers::set_up();
        let asset_rewards = test_helpers::create_fungible_asset_and_mint(claimer, b"test1", 1);
        let rewards_pool = rewards_pool::create(vector[fungible_asset::asset_metadata(&asset_rewards)]);

        increase_alocation(claimer, rewards_pool, 60);
        epoch::fast_forward(1);
        verify_claimer_shares_percentage(claimer, rewards_pool, 100, 60);
        claim_and_verify_rewards(claimer, rewards_pool, 100, vector[0]);

        test_helpers::clean_up(vector[asset_rewards]);
    }

    #[test(claimer_1 = @0xdead, claimer_2 = @0xbeef)]
    fun test_cannot_claim_twice(claimer_1: &signer, claimer_2: &signer) {
        test_helpers::set_up();
        let asset_rewards = test_helpers::create_fungible_asset_and_mint(claimer_1, b"test1", 1000);
        let rewards_pool = rewards_pool::create(vector[fungible_asset::asset_metadata(&asset_rewards)]);

        increase_alocation(claimer_1, rewards_pool, 50);
        increase_alocation(claimer_2, rewards_pool, 50);
        epoch::fast_forward(1);
        rewards_pool::add_rewards(
            rewards_pool,
            vector[asset_rewards],
            epoch::now() - 1,
        );
        verify_claimer_shares_percentage(claimer_1, rewards_pool, 100, 50);
        verify_claimer_shares_percentage(claimer_2, rewards_pool, 100, 50);
        rewards_pool::claim_rewards_entry(claimer_1, rewards_pool, 100);
        // Claimer 1 claiming a second time will return 0 rewards even though there's some rewards left that have not
        // been claimed by claimer 2.
        claim_and_verify_rewards(claimer_1, rewards_pool, 100, vector[0]);
    }

    #[test(claimer = @0xdead)]
    #[expected_failure(abort_code = 2, location = rewards_pool::rewards_pool)]
    fun test_cannot_add_reward_for_unsupported_tokens(claimer: &signer) {
        test_helpers::set_up();
        let asset_rewards = test_helpers::create_fungible_asset_and_mint(claimer, b"test1", 1000);
        let non_reward_assets = test_helpers::create_fungible_asset_and_mint(claimer, b"test2", 1000);
        let rewards_pool = rewards_pool::create(vector[fungible_asset::asset_metadata(&asset_rewards)]);

        rewards_pool::add_rewards(
            rewards_pool,
            vector[non_reward_assets],
            epoch::now() - 1,
        );
        test_helpers::clean_up(vector[asset_rewards]);
    }

    #[test(claimer = @0xdead)]
    #[expected_failure(abort_code = 1, location = rewards_pool::rewards_pool)]
    fun test_cannot_claim_rewards_for_current_epoch(claimer: &signer) {
        test_helpers::set_up();
        let asset_rewards = test_helpers::create_fungible_asset_and_mint(claimer, b"test1", 1000);
        let rewards_pool = rewards_pool::create(vector[fungible_asset::asset_metadata(&asset_rewards)]);

        rewards_pool::add_rewards(
            rewards_pool,
            vector[asset_rewards],
            epoch::now(),
        );
        rewards_pool::claim_rewards_entry(claimer, rewards_pool, 100);
    }

    fun verify_claimer_shares_percentage(
        claimer_1: &signer,
        rewards_pool: Object<RewardsPool>,
        epoch: u64,
        expected_shares: u64,
    ) {
        let (shares, _) = rewards_pool::claimer_shares(signer::address_of(claimer_1), rewards_pool, epoch);
        assert!(shares == expected_shares, 0);
    }

    fun add_rewards(
        pool: Object<RewardsPool>,
        rewards_1: &mut FungibleAsset,
        rewards_2: &mut FungibleAsset,
        amount_1: u64,
        amount_2: u64,
    ) {
        rewards_pool::add_rewards(
            pool,
            vector[fungible_asset::extract(rewards_1, amount_1), fungible_asset::extract(rewards_2, amount_2)],
            epoch::now() - 1,
        );
    }

    fun increase_alocation(claimer: &signer, pool: Object<RewardsPool>, amount: u64) {
        rewards_pool::increase_allocation(signer::address_of(claimer), pool, amount);
    }

    fun decrease_alocation(claimer: &signer, pool: Object<RewardsPool>, amount: u64) {
        rewards_pool::decrease_allocation(signer::address_of(claimer), pool, amount);
    }

    fun claim_and_verify_rewards(
        claimer: &signer,
        pool: Object<RewardsPool>,
        epoch: u64,
        expected_amounts: vector<u64>,
    ) {
        let claimer_addr = signer::address_of(claimer);
        let (non_zero_reward_tokens, claimable_rewards) = rewards_pool::claimable_rewards(claimer_addr, pool, epoch);
        let claimable_map = simple_map::new_from(non_zero_reward_tokens, claimable_rewards);
        let rewards = rewards_pool::claim_rewards(claimer, pool, epoch);
        vector::zip(rewards, expected_amounts, |reward, expected_amount| {
            let reward_metadata = fungible_asset::asset_metadata(&reward);
            let claimable_amount = if (simple_map::contains_key(&claimable_map, &reward_metadata)) {
                *simple_map::borrow(&claimable_map, &reward_metadata)
            } else {
                0
            };
            assert!(fungible_asset::amount(&reward) == claimable_amount, 0);
            assert!(fungible_asset::amount(&reward) == expected_amount, 0);
            primary_fungible_store::deposit(claimer_addr, reward);
        });
    }
}
```

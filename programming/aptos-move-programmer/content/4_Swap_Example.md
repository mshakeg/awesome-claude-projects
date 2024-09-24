# Aptos Move Swap Example

This document presents a comprehensive example of a decentralized exchange (DEX) implementation using Aptos Move. The swap project demonstrates advanced concepts in DeFi development on the Aptos blockchain, including liquidity pool management, token standard bridging, and secure permission handling. Each file in this project is preceded by a detailed preface, explaining its role, key features, and how it leverages Aptos Move's unique capabilities. This example showcases best practices in smart contract development, testing, and integration within the Aptos ecosystem, providing valuable insights for developers building complex DeFi applications on Aptos.

The swap example can be found [here](https://github.com/aptos-labs/aptos-core/tree/main/aptos-move/move-examples/swap) and it has the following folder structure:

```
.
├── Move.toml
└── sources
    ├── coin_wrapper.move
    ├── liquidity_pool.move
    ├── package_manager.move
    ├── router.move
    └── tests
        ├── coin_wrapper_tests.move
        ├── liquidity_pool_tests.move
        ├── package_manager_tests.move
        └── test_helpers.move
```

Now the content of the files:

./Move.toml

```toml
[package]
name = "swap"
version = "0.0.0"

[addresses]
aptos_framework = "0x1"
deployer = "_"
swap = "_"

[dependencies]
AptosFramework = { local = "../../framework/aptos-framework" }
```

files in the ./sources folder:

# package_manager.move

This file implements the core permission and configuration management for the swap project using Aptos Move's resource account feature. Key aspects include:

1. Resource Account Management:
   - Utilizes Aptos' resource account capability to create a secure, autonomous account for the swap package.
   - Stores and manages the SignerCapability for the resource account, enabling privileged operations across the project.

2. Address Management:
   - Implements a system to track and retrieve addresses created by modules in the package.
   - Uses a SmartTable for efficient storage and retrieval of named addresses.

3. Permission Control:
   - Provides friend-only access to critical functions like getting the resource account signer and adding addresses.
   - Ensures that only authorized modules within the swap package can perform privileged operations.

4. Initialization:
   - Implements an init_module function to set up the PermissionConfig when the package is deployed.
   - Retrieves and stores the resource account's signer capability during initialization.

5. Utility Functions:
   - Offers functions to check for address existence and retrieve addresses by name.
   - Provides a way to get the resource account's signer, crucial for operations requiring elevated permissions.

6. Test Support:
   - Includes test-only functions to facilitate unit testing of the package manager functionality.

This module serves as the foundation for the entire swap project, centralizing permission management and providing a secure way to handle privileged operations. It demonstrates effective use of Aptos Move's resource account feature and showcases best practices for managing permissions in a decentralized finance application.

```
module swap::package_manager {
    use aptos_framework::account::{Self, SignerCapability};
    use aptos_framework::resource_account;
    use aptos_std::smart_table::{Self, SmartTable};
    use std::string::String;

    friend swap::coin_wrapper;
    friend swap::liquidity_pool;
    friend swap::router;

    /// Stores permission config such as SignerCapability for controlling the resource account.
    struct PermissionConfig has key {
        /// Required to obtain the resource account signer.
        signer_cap: SignerCapability,
        /// Track the addresses created by the modules in this package.
        addresses: SmartTable<String, address>,
    }

    /// Initialize PermissionConfig to establish control over the resource account.
    /// This function is invoked only when this package is deployed the first time.
    fun init_module(swap_signer: &signer) {
        let signer_cap = resource_account::retrieve_resource_account_cap(swap_signer, @deployer);
        move_to(swap_signer, PermissionConfig {
            addresses: smart_table::new<String, address>(),
            signer_cap,
        });
    }

    /// Can be called by friended modules to obtain the resource account signer.
    public(friend) fun get_signer(): signer acquires PermissionConfig {
        let signer_cap = &borrow_global<PermissionConfig>(@swap).signer_cap;
        account::create_signer_with_capability(signer_cap)
    }

    /// Can be called by friended modules to keep track of a system address.
    public(friend) fun add_address(name: String, object: address) acquires PermissionConfig {
        let addresses = &mut borrow_global_mut<PermissionConfig>(@swap).addresses;
        smart_table::add(addresses, name, object);
    }

    public fun address_exists(name: String): bool acquires PermissionConfig {
        smart_table::contains(&safe_permission_config().addresses, name)
    }

    public fun get_address(name: String): address acquires PermissionConfig {
        let addresses = &borrow_global<PermissionConfig>(@swap).addresses;
        *smart_table::borrow(addresses, name)
    }

    inline fun safe_permission_config(): &PermissionConfig acquires PermissionConfig {
        borrow_global<PermissionConfig>(@swap)
    }

    #[test_only]
    public fun initialize_for_test(deployer: &signer) {
        let deployer_addr = std::signer::address_of(deployer);
        if (!exists<PermissionConfig>(deployer_addr)) {
            aptos_framework::timestamp::set_time_has_started_for_testing(&account::create_signer_for_test(@0x1));

            account::create_account_for_test(deployer_addr);
            move_to(deployer, PermissionConfig {
                addresses: smart_table::new<String, address>(),
                signer_cap: account::create_test_signer_cap(deployer_addr),
            });
        };
    }

    #[test_only]
    friend swap::package_manager_tests;
}
```

# coin_wrapper.move

This file implements a crucial bridging mechanism between the coin standard and the fungible asset standard in Aptos Move. Key aspects include:

1. Coin-to-Fungible Asset Wrapping:
   - Provides functionality to wrap coins into internal fungible assets, allowing unified handling of different token standards within the swap project.
   - Implements unwrapping to convert fungible assets back to coins when interacting with external systems.

2. Internal Fungible Asset Management:
   - Creates and manages internal fungible asset representations of coins, maintaining consistent properties (name, symbol, decimals) with the original coins.
   - Uses a resource account to store deposited coins and manage the corresponding fungible assets.

3. Data Structures:
   - Utilizes SmartTable for efficient storage and retrieval of fungible asset data and coin type mappings.
   - Implements FungibleAssetData to store metadata, mint, and burn capabilities for each wrapped asset.

4. Permission Control:
   - Uses the friend keyword to restrict access to critical functions like wrap and unwrap, ensuring only authorized modules can perform these operations.

5. Metadata Handling:
   - Provides functions to query and manage metadata for both coins and fungible assets, facilitating seamless transitions between the two standards.

6. Resource Account Utilization:
   - Leverages the resource account created by the package manager to store coin deposits and manage fungible asset operations.

7. Initialization and Configuration:
   - Implements an initialize function to set up the wrapper account and necessary data structures.
   - Provides configuration options for creating new fungible asset wrappers as needed.

8. Utility Functions:
   - Offers various helper functions for checking support status, retrieving original coin types, and formatting asset information.

This module plays a pivotal role in the swap project by enabling interoperability between different token standards. It demonstrates advanced use of Aptos Move features, including resource accounts, friend functions, and generics, to create a flexible and secure token wrapping system. The design allows the swap project to work seamlessly with both native fungible assets and traditional coins, enhancing its versatility and user accessibility.

```
/// This module can be included in a project to enable internal wrapping and unwrapping of coins into fungible assets.
/// This allows the project to only have to store and process fungible assets in core data structures, while still be
/// able to support both native fungible assets and coins. Note that the wrapper fungible assets are INTERNAL ONLY and
/// are not meant to be released to user's accounts outside of the project. Othwerwise, this would create multiple
/// conflicting fungible asset versions of a specific coin in the ecosystem.
///
/// The flow works as follows:
/// 1. Add the coin_wrapper module to the project.
/// 2. Add a friend declaration for any core modules that needs to call wrap/unwrap. Wrap/Unwrap are both friend-only
/// functions so external modules cannot call them and leak the internal fungible assets outside of the project.
/// 3. Add entry functions in the core modules that take coins. Those functions will be calling wrap to create the
/// internal fungible assets and store them.
/// 4. Add entry functions in the core modules that return coins. Those functions will be extract internal fungible
/// assets from the core data structures, unwrap them into and return the coins to the end users.
///
/// The fungible asset wrapper for a coin has the same name, symbol and decimals as the original coin. This allows for
/// easier accounting and tracking of the deposited/withdrawn coins.
module swap::coin_wrapper {
    use aptos_framework::account::{Self, SignerCapability};
    use aptos_framework::aptos_account;
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::fungible_asset::{Self, BurnRef, FungibleAsset, Metadata, MintRef};
    use aptos_framework::object::{Self, Object};
    use aptos_framework::primary_fungible_store;
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_std::string_utils;
    use aptos_std::type_info;
    use std::string::{Self, String};
    use std::option;
    use std::signer;
    use swap::package_manager;

    // Modules in the same package that need to wrap/unwrap coins need to be added as friends here.
    friend swap::router;

    const COIN_WRAPPER_NAME: vector<u8> = b"COIN_WRAPPER";

    /// Stores the refs for a specific fungible asset wrapper for wrapping and unwrapping.
    struct FungibleAssetData has store {
        // Used during unwrapping to burn the internal fungible assets.
        burn_ref: BurnRef,
        // Reference to the metadata object.
        metadata: Object<Metadata>,
        // Used during wrapping to mint the internal fungible assets.
        mint_ref: MintRef,
    }

    /// The resource stored in the main resource account to track all the fungible asset wrappers.
    /// This main resource account will also be the one holding all the deposited coins, each of which in a separate
    /// CoinStore<CoinType> resource. See coin.move in the Aptos Framework for more details.
    struct WrapperAccount has key {
        // The signer cap used to withdraw deposited coins from the main resource account during unwrapping so the
        // coins can be returned to the end users.
        signer_cap: SignerCapability,
        // Map from an original coin type (represented as strings such as "0x1::aptos_coin::AptosCoin") to the
        // corresponding fungible asset wrapper.
        coin_to_fungible_asset: SmartTable<String, FungibleAssetData>,
        // Map from a fungible asset wrapper to the original coin type.
        fungible_asset_to_coin: SmartTable<Object<Metadata>, String>,
    }

    /// Create the coin wrapper account to host all the deposited coins.
    public entry fun initialize() {
        if (is_initialized()) {
            return
        };

        let swap_signer = &package_manager::get_signer();
        let (coin_wrapper_signer, signer_cap) = account::create_resource_account(swap_signer, COIN_WRAPPER_NAME);
        package_manager::add_address(string::utf8(COIN_WRAPPER_NAME), signer::address_of(&coin_wrapper_signer));
        move_to(&coin_wrapper_signer, WrapperAccount {
            signer_cap,
            coin_to_fungible_asset: smart_table::new(),
            fungible_asset_to_coin: smart_table::new(),
        });
    }

    #[view]
    public fun is_initialized(): bool {
        package_manager::address_exists(string::utf8(COIN_WRAPPER_NAME))
    }

    #[view]
    /// Return the address of the resource account that stores all deposited coins.
    public fun wrapper_address(): address {
        package_manager::get_address(string::utf8(COIN_WRAPPER_NAME))
    }

    #[view]
    /// Return whether a specific CoinType has a wrapper fungible asset. This is only the case if at least one wrap()
    /// call has been made for that CoinType.
    public fun is_supported<CoinType>(): bool acquires WrapperAccount {
        let coin_type = type_info::type_name<CoinType>();
        smart_table::contains(&wrapper_account().coin_to_fungible_asset, coin_type)
    }

    #[view]
    /// Return true if the given fungible asset is a wrapper fungible asset.
    public fun is_wrapper(metadata: Object<Metadata>): bool acquires WrapperAccount {
        smart_table::contains(&wrapper_account().fungible_asset_to_coin, metadata)
    }

    #[view]
    /// Return the original CoinType for a specific wrapper fungible asset. This errors out if there's no such wrapper.
    public fun get_coin_type(metadata: Object<Metadata>): String acquires WrapperAccount {
        *smart_table::borrow(&wrapper_account().fungible_asset_to_coin, metadata)
    }

    #[view]
    /// Return the wrapper fungible asset for a specific CoinType. This errors out if there's no such wrapper.
    public fun get_wrapper<CoinType>(): Object<Metadata> acquires WrapperAccount {
        fungible_asset_data<CoinType>().metadata
    }

    #[view]
    /// Return the original CoinType if the given fungible asset is a wrapper fungible asset. Otherwise, return the
    /// given fungible asset itself, which means it's a native fungible asset (not wrapped).
    /// The return value is a String such as "0x1::aptos_coin::AptosCoin" for an original coin or "0x12345" for a native
    /// fungible asset.
    public fun get_original(fungible_asset: Object<Metadata>): String acquires WrapperAccount {
        if (is_wrapper(fungible_asset)) {
            get_coin_type(fungible_asset)
        } else {
            format_fungible_asset(fungible_asset)
        }
    }

    #[view]
    /// Return the address string of a fungible asset (e.g. "0x1234").
    public fun format_fungible_asset(fungible_asset: Object<Metadata>): String {
        let fa_address = object::object_address(&fungible_asset);
        // This will create "@0x123"
        let fa_address_str = string_utils::to_string(&fa_address);
        // We want to strip the prefix "@"
        string::sub_string(&fa_address_str, 1, string::length(&fa_address_str))
    }

    /// Wrap the given coins into fungible asset. This will also create the fungible asset wrapper if it doesn't exist
    /// yet. The coins will be deposited into the main resource account.
    public(friend) fun wrap<CoinType>(coins: Coin<CoinType>): FungibleAsset acquires WrapperAccount {
        // Ensure the corresponding fungible asset has already been created.
        create_fungible_asset<CoinType>();

        // Deposit coins into the main resource account and mint&return the wrapper fungible assets.
        let amount = coin::value(&coins);
        aptos_account::deposit_coins(wrapper_address(), coins);
        let mint_ref = &fungible_asset_data<CoinType>().mint_ref;
        fungible_asset::mint(mint_ref, amount)
    }

    /// Unwrap the given fungible asset into coins. This will burn the fungible asset and withdraw&return the coins from
    /// the main resource account.
    /// This errors out if the given fungible asset is not a wrapper fungible asset.
    public(friend) fun unwrap<CoinType>(fa: FungibleAsset): Coin<CoinType> acquires WrapperAccount {
        let amount = fungible_asset::amount(&fa);
        let burn_ref = &fungible_asset_data<CoinType>().burn_ref;
        fungible_asset::burn(burn_ref, fa);
        let wrapper_signer = &account::create_signer_with_capability(&wrapper_account().signer_cap);
        coin::withdraw(wrapper_signer, amount)
    }

    /// Create the fungible asset wrapper for the given CoinType if it doesn't exist yet.
    public(friend) fun create_fungible_asset<CoinType>(): Object<Metadata> acquires WrapperAccount {
        let coin_type = type_info::type_name<CoinType>();
        let wrapper_account = mut_wrapper_account();
        let coin_to_fungible_asset = &mut wrapper_account.coin_to_fungible_asset;
        let wrapper_signer = &account::create_signer_with_capability(&wrapper_account.signer_cap);
        if (!smart_table::contains(coin_to_fungible_asset, coin_type)) {
            let metadata_constructor_ref = &object::create_named_object(wrapper_signer, *string::bytes(&coin_type));
            primary_fungible_store::create_primary_store_enabled_fungible_asset(
                metadata_constructor_ref,
                option::none(),
                coin::name<CoinType>(),
                coin::symbol<CoinType>(),
                coin::decimals<CoinType>(),
                string::utf8(b""),
                string::utf8(b""),
            );

            let mint_ref = fungible_asset::generate_mint_ref(metadata_constructor_ref);
            let burn_ref = fungible_asset::generate_burn_ref(metadata_constructor_ref);
            let metadata = object::object_from_constructor_ref<Metadata>(metadata_constructor_ref);
            smart_table::add(coin_to_fungible_asset, coin_type, FungibleAssetData {
                metadata,
                mint_ref,
                burn_ref,
            });
            smart_table::add(&mut wrapper_account.fungible_asset_to_coin, metadata, coin_type);
        };
        smart_table::borrow(coin_to_fungible_asset, coin_type).metadata
    }

    inline fun fungible_asset_data<CoinType>(): &FungibleAssetData acquires WrapperAccount {
        let coin_type = type_info::type_name<CoinType>();
        smart_table::borrow(&wrapper_account().coin_to_fungible_asset, coin_type)
    }

    inline fun wrapper_account(): &WrapperAccount acquires WrapperAccount {
        borrow_global<WrapperAccount>(wrapper_address())
    }

    inline fun mut_wrapper_account(): &mut WrapperAccount acquires WrapperAccount {
        borrow_global_mut<WrapperAccount>(wrapper_address())
    }

    #[test_only]
    friend swap::coin_wrapper_tests;
}
```

# liquidity_pool.move

This file implements the core functionality of the automated market maker (AMM) in the swap project, utilizing Aptos Move's object model and advanced features. Key aspects include:

1. Liquidity Pool Implementation:
   - Supports both volatile and stable token pairs using different mathematical models for price calculation.
   - Utilizes the constant product formula (x * y = k) for volatile pairs and a modified formula (x^3 * y + x * y^3 = k) for stable pairs.

2. Object Model Usage:
   - Leverages Aptos Move's object model to represent liquidity pools, tokens, and LP tokens as addressable entities.
   - Implements custom resource types like LiquidityPool and FeesAccounting as object resources.

3. LP Token Management:
   - Implements minting, burning, and transfer of LP tokens to represent liquidity providers' shares.
   - Uses custom transfer logic to ensure proper fee accounting during LP token transfers.

4. Fee System:
   - Implements a sophisticated fee collection and distribution system for liquidity providers.
   - Stores collected fees separately from the pool's reserves to prevent fee compounding.

5. Swap Functionality:
   - Implements swap operations with slippage protection and minimum output amount checks.
   - Uses complex mathematical operations to calculate swap amounts and maintain pool invariants.

6. Pool Creation and Configuration:
   - Provides functions to create new liquidity pools with configurable parameters.
   - Supports both fixed supply and unlimited supply pool creation.

7. Event System:
   - Utilizes Aptos Move's event system to emit events for pool creation and swap operations.

8. Access Control:
   - Implements role-based access control for administrative functions like setting fees and pausing swaps.
   - Uses friend modules to restrict access to critical functions that return fungible assets.

9. Mathematical Utilities:
   - Implements custom mathematical functions for complex calculations required in stable swap operations.

10. Error Handling:
    - Defines and uses custom error codes for various failure scenarios, enhancing the contract's robustness.

11. View Functions:
    - Provides numerous view functions for querying pool states, reserves, and other relevant information.

This module forms the heart of the swap project, implementing sophisticated AMM functionality while leveraging Aptos Move's unique features. It demonstrates advanced use of the object model, complex mathematical operations, and careful state management to create a secure and efficient decentralized exchange system. The design allows for flexibility in handling different token types and pool configurations, showcasing the power and versatility of Aptos Move for building complex DeFi applications.

```
/// This module provides a common type of liquidity pool that supports both volatile and stable token pairs. It uses
/// fungible assets underneath and needs a separate router + coin_wrapper to support coins (different standard from
/// fungible assets). Swap fees are kept separately from the pool's reserves and thus don't compound.
///
/// For volatile pairs, the price and reserves can be computed using the constant product formula k = x * y.
/// For stable pairs, the price is computed using k = x^3 * y + x * y^3.
///
/// Note that all functions that return fungible assets such as swap, burn, claim_fees are friend-only since they might
/// return an internal wrapper fungible assets. The router or other modules should provide interface to call these based
/// on the underlying tokens of the pool - whether they're coins or fungible assets. See router.move for an example.
///
/// Another important thing to note is that all transfers of the LP tokens have to call via this module. This is
/// required so that fees are correctly updated for LPs. fungible_asset::transfer and primary_fungible_store::transfer
/// are not supported
module swap::liquidity_pool {
    use aptos_framework::event;
    use aptos_framework::fungible_asset::{
        Self, FungibleAsset, FungibleStore, Metadata,
        BurnRef, MintRef, TransferRef,
    };
    use aptos_framework::object::{Self, ConstructorRef, Object};
    use aptos_framework::primary_fungible_store;
    use aptos_std::comparator;
    use aptos_std::math128;
    use aptos_std::math64;
    use aptos_std::smart_table::{Self, SmartTable};
    use aptos_std::smart_vector::{Self, SmartVector};

    use swap::coin_wrapper;
    use swap::package_manager;

    use std::bcs;
    use std::option;
    use std::signer;
    use std::string::{Self, String};
    use std::vector;

    friend swap::router;

    const FEE_SCALE: u64 = 10000;
    const LP_TOKEN_DECIMALS: u8 = 8;
    const MINIMUM_LIQUIDITY: u64 = 1000;
    const MAX_SWAP_FEES_BPS: u64 = 25; // 0.25%

    /// Amount of tokens provided must be greater than zero.
    const EZERO_AMOUNT: u64 = 1;
    /// The amount of liquidity provided is so small that corresponding LP token amount is rounded to zero.
    const EINSUFFICIENT_LIQUIDITY_MINTED: u64 = 2;
    /// Amount of LP tokens redeemed is too small, so amounts of tokens received back are rounded to zero.
    const EINSUFFICIENT_LIQUIDITY_REDEEMED: u64 = 3;
    /// The specified amount of output tokens is incorrect and does not maintain the pool's invariant.
    const EINCORRECT_SWAP_AMOUNT: u64 = 4;
    /// The caller is not the owner of the LP token store.
    const ENOT_STORE_OWNER: u64 = 5;
    /// Claler is not authorized to perform the operation.
    const ENOT_AUTHORIZED: u64 = 6;
    /// All swaps are currently paused.
    const ESWAPS_ARE_PAUSED: u64 = 7;
    /// Swap leaves pool in a worse state than before.
    const EK_BEFORE_SWAP_GREATER_THAN_EK_AFTER_SWAP: u64 = 8;

    struct LPTokenRefs has store {
        burn_ref: BurnRef,
        mint_ref: MintRef,
        transfer_ref: TransferRef,
    }

    /// Stored in the protocol's account for configuring liquidity pools.
    struct LiquidityPoolConfigs has key {
        all_pools: SmartVector<Object<LiquidityPool>>,
        is_paused: bool,
        fee_manager: address,
        pauser: address,
        pending_fee_manager: address,
        pending_pauser: address,
        stable_fee_bps: u64,
        volatile_fee_bps: u64,
    }

    #[resource_group_member(group = aptos_framework::object::ObjectGroup)]
    struct LiquidityPool has key {
        token_store_1: Object<FungibleStore>,
        token_store_2: Object<FungibleStore>,
        fees_store_1: Object<FungibleStore>,
        fees_store_2: Object<FungibleStore>,
        lp_token_refs: LPTokenRefs,
        swap_fee_bps: u64,
        is_stable: bool,
    }

    #[resource_group_member(group = aptos_framework::object::ObjectGroup)]
    struct FeesAccounting has key {
        total_fees_1: u128,
        total_fees_2: u128,
        total_fees_at_last_claim_1: SmartTable<address, u128>,
        total_fees_at_last_claim_2: SmartTable<address, u128>,
        claimable_1: SmartTable<address, u128>,
        claimable_2: SmartTable<address, u128>,
    }

    #[event]
    /// Event emitted when a pool is created.
    struct CreatePool has drop, store {
        pool: Object<LiquidityPool>,
        token_1: Object<Metadata>,
        token_2: Object<Metadata>,
        is_stable: bool,
    }

    #[event]
    /// Event emitted when a swap happens.
    struct Swap has drop, store {
        pool: address,
        from_token: Object<Metadata>,
        amount_in: u64,
    }

    public entry fun initialize() {
        if (is_initialized()) {
            return
        };

        coin_wrapper::initialize();
        let swap_signer = &package_manager::get_signer();
        move_to(swap_signer, LiquidityPoolConfigs {
            all_pools: smart_vector::new(),
            is_paused: false,
            fee_manager: @deployer,
            pauser: @deployer,
            pending_fee_manager: @0x0,
            pending_pauser: @0x0,
            stable_fee_bps: 10, // 0.1%
            volatile_fee_bps: 20, // 0.2%
        });
    }

    #[view]
    public fun is_initialized(): bool {
        exists<LiquidityPoolConfigs>(@swap)
    }

    #[view]
    public fun total_number_of_pools(): u64 acquires LiquidityPoolConfigs {
        smart_vector::length(&safe_liquidity_pool_configs().all_pools)
    }

    #[view]
    public fun all_pools(): vector<Object<LiquidityPool>> acquires LiquidityPoolConfigs {
        let all_pools = &safe_liquidity_pool_configs().all_pools;
        let results = vector[];
        let len = smart_vector::length(all_pools);
        let i = 0;
        while (i < len) {
            vector::push_back(&mut results, *smart_vector::borrow(all_pools, i));
            i = i + 1;
        };
        results
    }

    #[view]
    public fun liquidity_pool(
        token_1: Object<Metadata>,
        token_2: Object<Metadata>,
        is_stable: bool,
    ): Object<LiquidityPool> {
        object::address_to_object(liquidity_pool_address(token_1, token_2, is_stable))
    }

    #[view]
    public fun liquidity_pool_address_safe(
        token_1: Object<Metadata>,
        token_2: Object<Metadata>,
        is_stable: bool,
    ): (bool, address) {
        let pool_address = liquidity_pool_address(token_1, token_2, is_stable);
        (exists<LiquidityPool>(pool_address), pool_address)
    }

    #[view]
    public fun liquidity_pool_address(
        token_1: Object<Metadata>,
        token_2: Object<Metadata>,
        is_stable: bool,
    ): address {
        if (!is_sorted(token_1, token_2)) {
            return liquidity_pool_address(token_2, token_1, is_stable)
        };
        object::create_object_address(&@swap, get_pool_seeds(token_1, token_2, is_stable))
    }

    #[view]
    public fun lp_token_supply<T: key>(pool: Object<T>): u128 {
        option::destroy_some(fungible_asset::supply(pool))
    }

    #[view]
    public fun pool_reserves<T: key>(pool: Object<T>): (u64, u64) acquires LiquidityPool {
        let pool_data = liquidity_pool_data(&pool);
        (
            fungible_asset::balance(pool_data.token_store_1),
            fungible_asset::balance(pool_data.token_store_2),
        )
    }

    #[view]
    public fun supported_token_strings(pool: Object<LiquidityPool>): vector<String> acquires LiquidityPool {
        vector::map(supported_inner_assets(pool), |a| coin_wrapper::get_original(a))
    }

    #[view]
    public fun supported_coins(pool: Object<LiquidityPool>): vector<String> acquires LiquidityPool {
        let coins = vector[];
        vector::for_each(supported_inner_assets(pool), |a| {
            if (coin_wrapper::is_wrapper(a)) {
                vector::push_back(&mut coins, coin_wrapper::get_coin_type(a))
            };
        });
        coins
    }

    #[view]
    public fun supported_native_fungible_assets(
        pool: Object<LiquidityPool>,
    ): vector<Object<Metadata>> acquires LiquidityPool {
        vector::filter(supported_inner_assets(pool), |a| !coin_wrapper::is_wrapper(*a))
    }

    #[view]
    public fun supported_inner_assets(pool: Object<LiquidityPool>): vector<Object<Metadata>> acquires LiquidityPool {
        let pool_data = liquidity_pool_data(&pool);
        vector[
            fungible_asset::store_metadata(pool_data.token_store_1),
            fungible_asset::store_metadata(pool_data.token_store_2),
        ]
    }

    #[view]
    public fun is_sorted(token_1: Object<Metadata>, token_2: Object<Metadata>): bool {
        let token_1_addr = object::object_address(&token_1);
        let token_2_addr = object::object_address(&token_2);
        comparator::is_smaller_than(&comparator::compare(&token_1_addr, &token_2_addr))
    }

    #[view]
    public fun is_stable(pool: Object<LiquidityPool>): bool acquires LiquidityPool {
        liquidity_pool_data(&pool).is_stable
    }

    #[view]
    public fun swap_fee_bps(pool: Object<LiquidityPool>): u64 acquires LiquidityPool {
        liquidity_pool_data(&pool).swap_fee_bps
    }

    #[view]
    public fun min_liquidity(): u64 {
        MINIMUM_LIQUIDITY
    }

    #[view]
    public fun claimable_fees(lp: address, pool: Object<LiquidityPool>): (u128, u128) acquires FeesAccounting {
        let fees_accounting = safe_fees_accounting(&pool);
        (
            *smart_table::borrow_with_default(&fees_accounting.claimable_1, lp, &0),
            *smart_table::borrow_with_default(&fees_accounting.claimable_2, lp, &0),
        )
    }

    /// Creates a new liquidity pool.
    public fun create(
        token_1: Object<Metadata>,
        token_2: Object<Metadata>,
        is_stable: bool,
    ): Object<LiquidityPool> acquires LiquidityPoolConfigs {
        if (!is_sorted(token_1, token_2)) {
            return create(token_2, token_1, is_stable)
        };
        let configs = unchecked_mut_liquidity_pool_configs();

        // The liquidity pool will serve 3 separate roles:
        // 1. Represent the liquidity pool that LPs and users interact with to add/remove liquidity and swap tokens.
        // 2. Represent the metadata of the LP token.
        // 3. Store the min liquidity that will be locked into the pool when initial liquidity is added.
        let pool_constructor_ref = create_lp_token(token_1, token_2, is_stable);
        let pool_signer = &object::generate_signer(pool_constructor_ref);
        let lp_token = object::object_from_constructor_ref<Metadata>(pool_constructor_ref);
        fungible_asset::create_store(pool_constructor_ref, lp_token);
        move_to(pool_signer, LiquidityPool {
            token_store_1: create_token_store(pool_signer, token_1),
            token_store_2: create_token_store(pool_signer, token_2),
            fees_store_1: create_token_store(pool_signer, token_1),
            fees_store_2: create_token_store(pool_signer, token_2),
            lp_token_refs: create_lp_token_refs(pool_constructor_ref),
            swap_fee_bps: if (is_stable) { configs.stable_fee_bps } else { configs.volatile_fee_bps },
            is_stable,
        });
        move_to(pool_signer, FeesAccounting {
            total_fees_1: 0,
            total_fees_2: 0,
            total_fees_at_last_claim_1: smart_table::new(),
            total_fees_at_last_claim_2: smart_table::new(),
            claimable_1: smart_table::new(),
            claimable_2: smart_table::new(),
        });
        let pool = object::convert(lp_token);
        smart_vector::push_back(&mut configs.all_pools, pool);

        event::emit(CreatePool { pool, token_1, token_2, is_stable });
        pool
    }

    /////////////////////////////////////////////////// USERS /////////////////////////////////////////////////////////

    #[view]
    /// Return the amount of tokens received for a swap with the given amount in and the liquidity pool.
    public fun get_amount_out(
        pool: Object<LiquidityPool>,
        from: Object<Metadata>,
        amount_in: u64,
    ): (u64, u64) acquires LiquidityPool {
        let pool_data = liquidity_pool_data(&pool);
        let reserve_1 = (fungible_asset::balance(pool_data.token_store_1) as u256);
        let reserve_2 = (fungible_asset::balance(pool_data.token_store_2) as u256);
        let (reserve_in, reserve_out) = if (from == fungible_asset::store_metadata(pool_data.token_store_1)) {
            (reserve_1, reserve_2)
        } else {
            (reserve_2, reserve_1)
        };
        let fees_amount = math64::mul_div(amount_in, pool_data.swap_fee_bps, FEE_SCALE);
        let amount_in = ((amount_in - fees_amount) as u256);
        let amount_out = if (pool_data.is_stable) {
            let k = calculate_constant_k(pool_data);
            reserve_out - get_y(amount_in + reserve_in, k, reserve_out)
        } else {
            amount_in * reserve_out / (reserve_in + amount_in)
        };
        ((amount_out as u64), fees_amount)
    }

    /// Swaps `from` for the other token in the pool.
    /// This is friend-only as the returned fungible assets might be of an internal wrapper type. If this is not the
    /// case, this function can be made public.
    public(friend) fun swap(
        pool: Object<LiquidityPool>,
        from: FungibleAsset,
    ): FungibleAsset acquires FeesAccounting, LiquidityPool, LiquidityPoolConfigs {
        assert!(!safe_liquidity_pool_configs().is_paused, ESWAPS_ARE_PAUSED);
        // Calculate the amount of tokens to return to the user and the amount of fees to extract.
        let from_token = fungible_asset::metadata_from_asset(&from);
        let amount_in = fungible_asset::amount(&from);
        let (amount_out, fees_amount) = get_amount_out(pool, from_token, amount_in);
        let fees = fungible_asset::extract(&mut from, fees_amount);

        // Deposits and withdraws.
        let pool_data = liquidity_pool_data(&pool);
        let k_before = calculate_constant_k(pool_data);
        let fees_accounting = unchecked_mut_fees_accounting(&pool);
        let store_1 = pool_data.token_store_1;
        let store_2 = pool_data.token_store_2;
        let swap_signer = &package_manager::get_signer();
        let fees_amount = (fees_amount as u128);
        let out = if (from_token == fungible_asset::store_metadata(pool_data.token_store_1)) {
            // User's swapping token 1 for token 2.
            fungible_asset::deposit(store_1, from);
            fungible_asset::deposit(pool_data.fees_store_1, fees);
            fees_accounting.total_fees_1 = fees_accounting.total_fees_1 + fees_amount;
            fungible_asset::withdraw(swap_signer, store_2, amount_out)
        } else {
            // User's swapping token 2 for token 1.
            fungible_asset::deposit(store_2, from);
            fungible_asset::deposit(pool_data.fees_store_2, fees);
            fees_accounting.total_fees_2 = fees_accounting.total_fees_2 + fees_amount;
            fungible_asset::withdraw(swap_signer, store_1, amount_out)
        };

        let k_after = calculate_constant_k(pool_data);
        assert!(k_before <= k_after, EK_BEFORE_SWAP_GREATER_THAN_EK_AFTER_SWAP);

        event::emit(Swap { pool: object::object_address(&pool), from_token, amount_in }, );
        out
    }

    //////////////////////////////////////// Liquidity Providers (LPs) ///////////////////////////////////////////////

    /// Mint LP tokens for the given liquidity. Note that the LP would receive a smaller amount of LP tokens if the
    /// amounts of liquidity provided are not optimal (do not conform with the constant formula of the pool). Users
    /// should compute the optimal amounts before calling this function.
    public fun mint(
        lp: &signer,
        fungible_asset_1: FungibleAsset,
        fungible_asset_2: FungibleAsset,
        is_stable: bool,
    ) acquires FeesAccounting, LiquidityPool {
        let token_1 = fungible_asset::metadata_from_asset(&fungible_asset_1);
        let token_2 = fungible_asset::metadata_from_asset(&fungible_asset_2);
        if (!is_sorted(token_1, token_2)) {
            return mint(lp, fungible_asset_2, fungible_asset_1, is_stable)
        };
        // The LP store needs to exist before we can mint LP tokens.
        let pool = liquidity_pool(token_1, token_2, is_stable);
        let lp_store = ensure_lp_token_store(signer::address_of(lp), pool);
        let amount_1 = fungible_asset::amount(&fungible_asset_1);
        let amount_2 = fungible_asset::amount(&fungible_asset_2);
        assert!(amount_1 > 0 && amount_2 > 0, EZERO_AMOUNT);
        let pool_data = liquidity_pool_data(&pool);
        let store_1 = pool_data.token_store_1;
        let store_2 = pool_data.token_store_2;

        // Before depositing the added liquidity, compute the amount of LP tokens the LP will receive.
        let reserve_1 = fungible_asset::balance(store_1);
        let reserve_2 = fungible_asset::balance(store_2);
        let lp_token_supply = option::destroy_some(fungible_asset::supply(pool));
        let mint_ref = &pool_data.lp_token_refs.mint_ref;
        let liquidity_token_amount = if (lp_token_supply == 0) {
            let total_liquidity = (math128::sqrt((amount_1 as u128) * (amount_2 as u128)) as u64);
            // Permanently lock the first MINIMUM_LIQUIDITY tokens.
            fungible_asset::mint_to(mint_ref, pool, MINIMUM_LIQUIDITY);
            total_liquidity - MINIMUM_LIQUIDITY
        } else {
            // Only the smaller amount between the token 1 or token 2 is considered. Users should make sure to either
            // use the router module or calculate the optimal amounts to provide before calling this function.
            let token_1_liquidity = math64::mul_div(amount_1, (lp_token_supply as u64), reserve_1);
            let token_2_liquidity = math64::mul_div(amount_2, (lp_token_supply as u64), reserve_2);
            math64::min(token_1_liquidity, token_2_liquidity)
        };
        assert!(liquidity_token_amount > 0, EINSUFFICIENT_LIQUIDITY_MINTED);

        // Deposit the received liquidity into the pool.
        fungible_asset::deposit(store_1, fungible_asset_1);
        fungible_asset::deposit(store_2, fungible_asset_2);

        // We need to update the amount of rewards claimable by this LP token store if they already have a previous
        // balance. This ensures that their update balance would not lead to earning a larger portion of the fees
        // retroactively.
        update_claimable_fees(signer::address_of(lp), pool);

        // Mint the corresponding amount of LP tokens to the LP.
        let lp_tokens = fungible_asset::mint(mint_ref, liquidity_token_amount);
        fungible_asset::deposit_with_ref(&pool_data.lp_token_refs.transfer_ref, lp_store, lp_tokens);
    }

    /// Transfer a given amount of LP tokens from the sender to the receiver. This must be called for all transfers as
    /// fungible_asset::transfer or primary_fungible_store::transfer would not work for LP tokens.
    public entry fun transfer(
        from: &signer,
        lp_token: Object<LiquidityPool>,
        to: address,
        amount: u64,
    ) acquires FeesAccounting, LiquidityPool {
        assert!(amount > 0, EZERO_AMOUNT);
        let from_address = signer::address_of(from);
        let from_store = ensure_lp_token_store(from_address, lp_token);
        let to_store = ensure_lp_token_store(to, lp_token);

        // Update the claimable amounts for both the sender and receiver before transferring.
        update_claimable_fees(from_address, lp_token);
        update_claimable_fees(to, lp_token);

        let transfer_ref = &liquidity_pool_data(&lp_token).lp_token_refs.transfer_ref;
        fungible_asset::transfer_with_ref(transfer_ref, from_store, to_store, amount);
    }

    /// Burn the given amount of LP tokens and receive the underlying liquidity.
    /// This is friend-only as the returned fungible assets might be of an internal wrapper type. If this is not the
    /// case, this function can be made public.
    public(friend) fun burn(
        lp: &signer,
        token_1: Object<Metadata>,
        token_2: Object<Metadata>,
        is_stable: bool,
        amount: u64,
    ): (FungibleAsset, FungibleAsset) acquires FeesAccounting, LiquidityPool {
        assert!(amount > 0, EZERO_AMOUNT);
        let lp_address = signer::address_of(lp);
        let pool = liquidity_pool(token_1, token_2, is_stable);
        let store = ensure_lp_token_store(lp_address, pool);

        // We need to update the amount of rewards claimable by this LP token store if they already have a previous
        // balance. This ensures that they can get the unclaimed fees they're entitled to before burning.
        update_claimable_fees(lp_address, pool);

        // Burn the provided LP tokens.
        let lp_token_supply = option::destroy_some(fungible_asset::supply(pool));
        let pool_data = liquidity_pool_data(&pool);
        fungible_asset::burn_from(&pool_data.lp_token_refs.burn_ref, store, amount);

        // Calculate the amounts of tokens redeemed from the pool.
        let store_1 = pool_data.token_store_1;
        let store_2 = pool_data.token_store_2;
        let reserve_1 = fungible_asset::balance(store_1);
        let reserve_2 = fungible_asset::balance(store_2);
        let amount_to_redeem_1 = (math128::mul_div(
            (amount as u128),
            (reserve_1 as u128),
            lp_token_supply
        ) as u64);
        let amount_to_redeem_2 = (math128::mul_div(
            (amount as u128),
            (reserve_2 as u128),
            lp_token_supply
        ) as u64);
        assert!(amount_to_redeem_1 > 0 && amount_to_redeem_2 > 0, EINSUFFICIENT_LIQUIDITY_REDEEMED);

        // Withdraw and return the redeemed tokens.
        let swap_signer = &package_manager::get_signer();
        let redeemed_1 = fungible_asset::withdraw(swap_signer, store_1, amount_to_redeem_1);
        let redeemed_2 = fungible_asset::withdraw(swap_signer, store_2, amount_to_redeem_2);
        if (is_sorted(token_1, token_2)) {
            (redeemed_1, redeemed_2)
        } else {
            (redeemed_2, redeemed_1)
        }
    }

    /// Calculate and update the latest amount of fees claimable by the given LP.
    public entry fun update_claimable_fees(lp: address, pool: Object<LiquidityPool>) acquires FeesAccounting {
        let fees_accounting = unchecked_mut_fees_accounting(&pool);
        let current_total_fees_1 = fees_accounting.total_fees_1;
        let current_total_fees_2 = fees_accounting.total_fees_2;
        let lp_balance = (primary_fungible_store::balance(lp, pool) as u128);
        let lp_token_total_supply = lp_token_supply(pool);
        // Calculate and update the amount of fees this LP token store is entitled to, taking into account the last
        // time they claimed.
        if (lp_balance > 0) {
            let last_total_fees_1 = *smart_table::borrow(&fees_accounting.total_fees_at_last_claim_1, lp);
            let last_total_fees_2 = *smart_table::borrow(&fees_accounting.total_fees_at_last_claim_2, lp);
            let delta_1 = current_total_fees_1 - last_total_fees_1;
            let delta_2 = current_total_fees_2 - last_total_fees_2;
            let claimable_1 = math128::mul_div(delta_1, lp_balance, lp_token_total_supply);
            let claimable_2 = math128::mul_div(delta_2, lp_balance, lp_token_total_supply);
            if (claimable_1 > 0) {
                let old_claimable_1 = smart_table::borrow_mut_with_default(&mut fees_accounting.claimable_1, lp, 0);
                *old_claimable_1 = *old_claimable_1 + claimable_1;
            };
            if (claimable_2 > 0) {
                let old_claimable_2 = smart_table::borrow_mut_with_default(&mut fees_accounting.claimable_2, lp, 0);
                *old_claimable_2 = *old_claimable_2 + claimable_2;
            };
        };

        smart_table::upsert(&mut fees_accounting.total_fees_at_last_claim_1, lp, current_total_fees_1);
        smart_table::upsert(&mut fees_accounting.total_fees_at_last_claim_2, lp, current_total_fees_2);
    }

    /// Claim the fees that the given LP is entitled to.
    /// This is friend-only as the returned fungible assets might be of an internal wrapper type. If this is not the
    /// case, this function can be made public.
    public(friend) fun claim_fees(
        lp: &signer,
        pool: Object<LiquidityPool>,
    ): (FungibleAsset, FungibleAsset) acquires FeesAccounting, LiquidityPool {
        let lp_address = signer::address_of(lp);
        update_claimable_fees(lp_address, pool);

        let pool_data = liquidity_pool_data(&pool);
        let fees_accounting = unchecked_mut_fees_accounting(&pool);
        let claimable_1 = if (smart_table::contains(&fees_accounting.claimable_1, lp_address)) {
            smart_table::remove(&mut fees_accounting.claimable_1, lp_address)
        } else {
            0
        };
        let claimable_2 = if (smart_table::contains(&fees_accounting.claimable_2, lp_address)) {
            smart_table::remove(&mut fees_accounting.claimable_2, lp_address)
        } else {
            0
        };
        let swap_signer = &package_manager::get_signer();
        let fees_1 = if (claimable_1 > 0) {
            fungible_asset::withdraw(swap_signer, pool_data.fees_store_1, (claimable_1 as u64))
        } else {
            fungible_asset::zero(fungible_asset::store_metadata(pool_data.fees_store_1))
        };
        let fees_2 = if (claimable_2 > 0) {
            fungible_asset::withdraw(swap_signer, pool_data.fees_store_2, (claimable_2 as u64))
        } else {
            fungible_asset::zero(fungible_asset::store_metadata(pool_data.fees_store_2))
        };
        (fees_1, fees_2)
    }

    /////////////////////////////////////////////////// OPERATIONS /////////////////////////////////////////////////////

    public entry fun set_pauser(pauser: &signer, new_pauser: address) acquires LiquidityPoolConfigs {
        let pool_configs = pauser_only_mut_liquidity_pool_configs(pauser);
        pool_configs.pending_pauser = new_pauser;
    }

    public entry fun accept_pauser(new_pauser: &signer) acquires LiquidityPoolConfigs {
        let pool_configs = unchecked_mut_liquidity_pool_configs();
        assert!(signer::address_of(new_pauser) == pool_configs.pending_pauser, ENOT_AUTHORIZED);
        pool_configs.pauser = pool_configs.pending_pauser;
        pool_configs.pending_pauser = @0x0;
    }

    public entry fun set_pause(pauser: &signer, is_paused: bool) acquires LiquidityPoolConfigs {
        let pool_configs = pauser_only_mut_liquidity_pool_configs(pauser);
        pool_configs.is_paused = is_paused;
    }

    public entry fun set_fee_manager(fee_manager: &signer, new_fee_manager: address) acquires LiquidityPoolConfigs {
        let pool_configs = fee_manager_only_mut_liquidity_pool_configs(fee_manager);
        pool_configs.pending_fee_manager = new_fee_manager;
    }

    public entry fun accept_fee_manager(new_fee_manager: &signer) acquires LiquidityPoolConfigs {
        let pool_configs = unchecked_mut_liquidity_pool_configs();
        assert!(signer::address_of(new_fee_manager) == pool_configs.pending_fee_manager, ENOT_AUTHORIZED);
        pool_configs.fee_manager = pool_configs.pending_fee_manager;
        pool_configs.pending_fee_manager = @0x0;
    }

    public entry fun set_stable_fee(fee_manager: &signer, new_fee_bps: u64) acquires LiquidityPoolConfigs {
        let pool_configs = fee_manager_only_mut_liquidity_pool_configs(fee_manager);
        pool_configs.stable_fee_bps = new_fee_bps;
    }

    public entry fun set_volatile_fee(fee_manager: &signer, new_fee_bps: u64) acquires LiquidityPoolConfigs {
        let pool_configs = fee_manager_only_mut_liquidity_pool_configs(fee_manager);
        pool_configs.volatile_fee_bps = new_fee_bps;
    }

    inline fun create_lp_token(
        token_1: Object<Metadata>,
        token_2: Object<Metadata>,
        is_stable: bool,
    ): &ConstructorRef {
        let token_name = lp_token_name(token_1, token_2);
        let seeds = get_pool_seeds(token_1, token_2, is_stable);
        let lp_token_constructor_ref = &object::create_named_object(&package_manager::get_signer(), seeds);
        // We don't enable automatic primary store creation because we need LPs to call into this module for transfers
        // so the fees accounting can be updated correctly.
        primary_fungible_store::create_primary_store_enabled_fungible_asset(
            lp_token_constructor_ref,
            option::none(),
            token_name,
            string::utf8(b"LP"),
            LP_TOKEN_DECIMALS,
            string::utf8(b""),
            string::utf8(b"")
        );
        lp_token_constructor_ref
    }

    fun create_lp_token_refs(constructor_ref: &ConstructorRef): LPTokenRefs {
        LPTokenRefs {
            burn_ref: fungible_asset::generate_burn_ref(constructor_ref),
            mint_ref: fungible_asset::generate_mint_ref(constructor_ref),
            transfer_ref: fungible_asset::generate_transfer_ref(constructor_ref),
        }
    }

    fun ensure_lp_token_store<T: key>(lp: address, pool: Object<T>): Object<FungibleStore> acquires LiquidityPool {
        primary_fungible_store::ensure_primary_store_exists(lp, pool);
        let store = primary_fungible_store::primary_store(lp, pool);
        if (!fungible_asset::is_frozen(store)) {
            // LPs must call transfer here to transfer the LP tokens so claimable fees can be updated correctly.
            let transfer_ref = &liquidity_pool_data(&pool).lp_token_refs.transfer_ref;
            fungible_asset::set_frozen_flag(transfer_ref, store, true);
        };
        store
    }

    inline fun get_pool_seeds(token_1: Object<Metadata>, token_2: Object<Metadata>, is_stable: bool): vector<u8> {
        let seeds = vector[];
        vector::append(&mut seeds, bcs::to_bytes(&object::object_address(&token_1)));
        vector::append(&mut seeds, bcs::to_bytes(&object::object_address(&token_2)));
        vector::append(&mut seeds, bcs::to_bytes(&is_stable));
        seeds
    }

    inline fun create_token_store(pool_signer: &signer, token: Object<Metadata>): Object<FungibleStore> {
        let constructor_ref = &object::create_object_from_object(pool_signer);
        fungible_asset::create_store(constructor_ref, token)
    }

    inline fun lp_token_name(token_1: Object<Metadata>, token_2: Object<Metadata>): String {
        let token_symbol = string::utf8(b"LP-");
        string::append(&mut token_symbol, fungible_asset::symbol(token_1));
        string::append_utf8(&mut token_symbol, b"-");
        string::append(&mut token_symbol, fungible_asset::symbol(token_2));
        token_symbol
    }

    inline fun calculate_constant_k(pool: &LiquidityPool): u256 {
        let r1 = (fungible_asset::balance(pool.token_store_1) as u256);
        let r2 = (fungible_asset::balance(pool.token_store_2) as u256);
        if (pool.is_stable) {
            // k = x^3 * y + y^3 * x. This is a modified constant for stable pairs.
            r1 * r1 * r1 * r2 + r2 * r2 * r2 * r1
        } else {
            // k = x * y. This is standard constant product for volatile asset pairs.
            r1 * r2
        }
    }

    inline fun safe_fees_accounting<T: key>(pool: &Object<T>): &FeesAccounting acquires FeesAccounting {
        borrow_global<FeesAccounting>(object::object_address(pool))
    }

    inline fun liquidity_pool_data<T: key>(pool: &Object<T>): &LiquidityPool acquires LiquidityPool {
        borrow_global<LiquidityPool>(object::object_address(pool))
    }

    inline fun safe_liquidity_pool_configs(): &LiquidityPoolConfigs acquires LiquidityPoolConfigs {
        borrow_global<LiquidityPoolConfigs>(@swap)
    }

    inline fun pauser_only_mut_liquidity_pool_configs(
        pauser: &signer,
    ): &mut LiquidityPoolConfigs acquires LiquidityPoolConfigs {
        let pool_configs = unchecked_mut_liquidity_pool_configs();
        assert!(signer::address_of(pauser) == pool_configs.pauser, ENOT_AUTHORIZED);
        pool_configs
    }

    inline fun fee_manager_only_mut_liquidity_pool_configs(
        fee_manager: &signer,
    ): &mut LiquidityPoolConfigs acquires LiquidityPoolConfigs {
        let pool_configs = unchecked_mut_liquidity_pool_configs();
        assert!(signer::address_of(fee_manager) == pool_configs.fee_manager, ENOT_AUTHORIZED);
        pool_configs
    }

    inline fun unchecked_mut_liquidity_pool_data<T: key>(pool: &Object<T>): &mut LiquidityPool acquires LiquidityPool {
        borrow_global_mut<LiquidityPool>(object::object_address(pool))
    }

    inline fun unchecked_mut_fees_accounting<T: key>(pool: &Object<T>): &mut FeesAccounting acquires FeesAccounting {
        borrow_global_mut<FeesAccounting>(object::object_address(pool))
    }

    inline fun unchecked_mut_liquidity_pool_configs(): &mut LiquidityPoolConfigs acquires LiquidityPoolConfigs {
        borrow_global_mut<LiquidityPoolConfigs>(@swap)
    }

    inline fun f(x0: u256, y: u256): u256 {
        x0 * (y * y * y) + (x0 * x0 * x0) * y
    }

    inline fun d(x0: u256, y: u256): u256 {
        3 * x0 * (y * y) + (x0 * x0 * x0)
    }

    fun get_y(x0: u256, xy: u256, y: u256): u256 {
        let i = 0;
        while (i < 255) {
            let y_prev = y;
            let k = f(x0, y);
            if (k < xy) {
                let dy = (xy - k) / d(x0, y);
                y = y + dy;
            } else {
                let dy = (k - xy) / d(x0, y);
                y = y - dy;
            };
            if (y > y_prev) {
                if (y - y_prev <= 1) {
                    return y
                }
            } else {
                if (y_prev - y <= 1) {
                    return y
                }
            };
            i = i + 1;
        };
        y
    }

    #[test_only]
    friend swap::liquidity_pool_tests;
}
```

# router.move

This file implements the main interface for user interactions with the swap project, bridging between different token standards and the core liquidity pool functionality. Key aspects include:

1. Unified Interface:
   - Provides a single entry point for users to interact with liquidity pools, regardless of whether they're using coins or native fungible assets.
   - Implements functions for swapping, adding liquidity, and removing liquidity that handle both token standards seamlessly.

2. Token Standard Bridging:
   - Utilizes the coin_wrapper module to convert between coins and fungible assets as needed.
   - Enables operations involving any combination of coins and native fungible assets (e.g., coin-to-coin, coin-to-asset, asset-to-coin, asset-to-asset).

3. Liquidity Pool Operations:
   - Implements high-level functions for creating pools, swapping tokens, adding liquidity, and removing liquidity.
   - Calculates optimal amounts for liquidity provision to minimize slippage.

4. Slippage Protection:
   - Incorporates slippage checks in swap and liquidity operations to protect users from unfavorable price movements.

5. Entry Functions:
   - Defines numerous entry functions to allow direct calls from transactions for various operations.
   - Provides both entry functions and regular functions to support different use cases (direct user interactions vs. composability with other modules).

6. Error Handling:
   - Implements custom error codes and assertions to handle various failure scenarios, enhancing the robustness of the contract.

7. View Functions:
   - Offers view functions for querying expected swap outputs and optimal liquidity amounts.

8. Generic Implementation:
   - Uses generic type parameters extensively to support operations with different coin types.

9. Friend Modules:
   - Utilizes the `friend` keyword to interact with restricted functions in the liquidity_pool module.

10. Mathematical Calculations:
    - Implements complex calculations for determining optimal swap and liquidity provision amounts.

11. Event Emissions:
    - Indirectly emits events through calls to the liquidity_pool module for important state changes.

This module serves as the main user-facing interface for the swap project, abstracting away the complexities of different token standards and internal implementations. It demonstrates effective use of Aptos Move's type system, generics, and module interoperability features to create a seamless and flexible decentralized exchange interface. The router's design allows for easy integration with external systems and provides a user-friendly API for interacting with the swap functionality, showcasing how complex DeFi operations can be simplified and made accessible in the Aptos ecosystem.

```
/// This module provides an interface for liquidity_pool that supports both coins and native fungible assets.
///
/// A liquidity pool has two tokens and thus can have 3 different combinations: 2 native fungible assets, 1 coin and
/// 1 native fungible asset, or 2 coins. Each combination has separate functions for swap, add and remove liquidity.
/// The coins provided by the users are wrapped and coins are returned to users by unwrapping internal fungible asset
/// with coin_wrapper.
module swap::router {
    use aptos_framework::aptos_account;
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::fungible_asset::{Self, FungibleAsset, Metadata};
    use aptos_framework::object::{Self, Object};
    use aptos_framework::primary_fungible_store;
    use aptos_std::math128;

    use swap::coin_wrapper;
    use swap::liquidity_pool::{Self, LiquidityPool};

    /// Output is less than the desired minimum amount.
    const EINSUFFICIENT_OUTPUT_AMOUNT: u64 = 1;
    /// The liquidity pool is misconfigured and has 0 amount of one asset but non-zero amount of the other.
    const EINFINITY_POOL: u64 = 2;
    /// One or both tokens passed are not valid native fungible assets.
    const ENOT_NATIVE_FUNGIBLE_ASSETS: u64 = 3;

    /////////////////////////////////////////////////// PROTOCOL ///////////////////////////////////////////////////////
    public entry fun create_pool(
        token_1: Object<Metadata>,
        token_2: Object<Metadata>,
        is_stable: bool,
    ) {
        liquidity_pool::create(token_1, token_2, is_stable);
    }

    public entry fun create_pool_coin<CoinType>(
        token_2: Object<Metadata>,
        is_stable: bool,
    ) {
        let token_1 = coin_wrapper::create_fungible_asset<CoinType>();
        create_pool(token_1, token_2, is_stable);
    }

    public entry fun create_pool_both_coins<CoinType1, CoinType2>(is_stable: bool) {
        let token_2 = coin_wrapper::create_fungible_asset<CoinType2>();
        create_pool_coin<CoinType1>(token_2, is_stable);
    }

    /////////////////////////////////////////////////// USERS /////////////////////////////////////////////////////////

    #[view]
    /// Return the expected amount out for a given amount in of tokens to swap via the given liquidity pool.
    public fun get_amount_out(
        amount_in: u64,
        from_token: Object<Metadata>,
        to_token: Object<Metadata>,
    ): (u64, u64) {
        let (found, pool) = liquidity_pool::liquidity_pool_address_safe(from_token, to_token, true);
        if (!found) {
            pool = liquidity_pool::liquidity_pool_address(to_token, from_token, false);
        };
        let pool = object::address_to_object<LiquidityPool>(pool);
        liquidity_pool::get_amount_out(pool, from_token, amount_in)
    }

    /// Swap an amount of fungible assets for another fungible asset. User can specifies the minimum amount they
    /// expect to receive. If the actual amount received is less than the minimum amount, the transaction will fail.
    public entry fun swap_entry(
        user: &signer,
        amount_in: u64,
        amount_out_min: u64,
        from_token: Object<Metadata>,
        to_token: Object<Metadata>,
        is_stable: bool,
        recipient: address,
    ) {
        let in = primary_fungible_store::withdraw(user, from_token, amount_in);
        let out = swap(in, amount_out_min, to_token, is_stable);
        primary_fungible_store::deposit(recipient, out);
    }

    /// Similar to swap_entry but returns the fungible asset received for composability with other modules.
    public fun swap(
        in: FungibleAsset,
        amount_out_min: u64,
        to_token: Object<Metadata>,
        is_stable: bool,
    ): FungibleAsset {
        let from_token = fungible_asset::asset_metadata(&in);
        let pool = liquidity_pool::liquidity_pool(from_token, to_token, is_stable);
        let out = liquidity_pool::swap(pool, in);
        assert!(fungible_asset::amount(&out) >= amount_out_min, EINSUFFICIENT_OUTPUT_AMOUNT);
        out
    }

    /// Swap an amount of coins for fungible assets. User can specifies the minimum amount they expect to receive.
    public entry fun swap_coin_for_asset_entry<FromCoin>(
        user: &signer,
        amount_in: u64,
        amount_out_min: u64,
        to_token: Object<Metadata>,
        is_stable: bool,
        recipient: address,
    ) {
        let in = coin::withdraw<FromCoin>(user, amount_in);
        let out = swap_coin_for_asset<FromCoin>(in, amount_out_min, to_token, is_stable);
        primary_fungible_store::deposit(recipient, out);
    }

    /// Similar to swap_coin_for_asset_entry but returns the fungible asset received for composability with other
    /// modules.
    public fun swap_coin_for_asset<FromCoin>(
        in: Coin<FromCoin>,
        amount_out_min: u64,
        to_token: Object<Metadata>,
        is_stable: bool,
    ): FungibleAsset {
        swap(coin_wrapper::wrap(in), amount_out_min, to_token, is_stable)
    }

    /// Swap an amount of fungible assets for coins. User can specifies the minimum amount they expect to receive.
    public entry fun swap_asset_for_coin_entry<ToCoin>(
        user: &signer,
        amount_in: u64,
        amount_out_min: u64,
        from_token: Object<Metadata>,
        is_stable: bool,
        recipient: address,
    ) {
        let in = primary_fungible_store::withdraw(user, from_token, amount_in);
        let out = swap_asset_for_coin<ToCoin>(in, amount_out_min, is_stable);
        aptos_account::deposit_coins(recipient, out);
    }

    /// Similar to swap_asset_for_coin_entry but returns the coin received for composability with other modules.
    public fun swap_asset_for_coin<ToCoin>(
        in: FungibleAsset,
        amount_out_min: u64,
        is_stable: bool,
    ): Coin<ToCoin> {
        let to_token = coin_wrapper::get_wrapper<ToCoin>();
        let out = swap(in, amount_out_min, to_token, is_stable);
        coin_wrapper::unwrap<ToCoin>(out)
    }

    /// Swap an amount of coins for another coin. User can specifies the minimum amount they expect to receive.
    public entry fun swap_coin_for_coin_entry<FromCoin, ToCoin>(
        user: &signer,
        amount_in: u64,
        amount_out_min: u64,
        is_stable: bool,
        recipient: address,
    ) {
        let in = coin::withdraw<FromCoin>(user, amount_in);
        let out = swap_coin_for_coin<FromCoin, ToCoin>(in, amount_out_min, is_stable);
        coin::deposit(recipient, out);
    }

    /// Similar to swap_coin_for_coin_entry but returns the coin received for composability with other modules.
    public fun swap_coin_for_coin<FromCoin, ToCoin>(
        in: Coin<FromCoin>,
        amount_out_min: u64,
        is_stable: bool,
    ): Coin<ToCoin> {
        let in = coin_wrapper::wrap(in);
        swap_asset_for_coin<ToCoin>(in, amount_out_min, is_stable)
    }

    /////////////////////////////////////////////////// LPs ///////////////////////////////////////////////////////////

    #[view]
    /// Returns the optimal amounts of tokens to provide as liquidity given the desired amount of each token to add.
    /// The returned values are the amounts of token 1, token 2, and LP tokens received.
    public fun optimal_liquidity_amounts(
        token_1: Object<Metadata>,
        token_2: Object<Metadata>,
        is_stable: bool,
        amount_1_desired: u64,
        amount_2_desired: u64,
        amount_1_min: u64,
        amount_2_min: u64,
    ): (u64, u64, u64) {
        let pool = liquidity_pool::liquidity_pool(token_1, token_2, is_stable);
        let (reserves_1, reserves_2) = liquidity_pool::pool_reserves(pool);
        // Reverse the reserve numbers if token 1 and token 2 don't match the pool's token order.
        if (!liquidity_pool::is_sorted(token_1, token_2)) {
            (reserves_1, reserves_2) = (reserves_2, reserves_1);
        };
        let amount_1_desired = (amount_1_desired as u128);
        let amount_2_desired = (amount_2_desired as u128);
        let amount_1_min = (amount_1_min as u128);
        let amount_2_min = (amount_2_min as u128);
        let reserves_1 = (reserves_1 as u128);
        let reserves_2 = (reserves_2 as u128);
        let lp_token_total_supply = liquidity_pool::lp_token_supply(pool);
        let (amount_1, amount_2) = (amount_1_desired, amount_2_desired);
        let liquidity = if (lp_token_total_supply == 0) {
            math128::sqrt(amount_1 * amount_2) - (liquidity_pool::min_liquidity() as u128)
        } else if (reserves_1 > 0 && reserves_2 > 0) {
            let amount_2_optimal = math128::mul_div(amount_1_desired, reserves_2, reserves_1);
            if (amount_2 <= amount_2_desired) {
                assert!(amount_2_optimal >= amount_2_min, EINSUFFICIENT_OUTPUT_AMOUNT);
                amount_2 = amount_2_optimal;
            } else {
                amount_1 = math128::mul_div(amount_2_desired, reserves_1, reserves_2);
                assert!(amount_1 <= amount_1_desired && amount_1 >= amount_1_min, EINSUFFICIENT_OUTPUT_AMOUNT);
            };
            math128::min(
                amount_1 * lp_token_total_supply / reserves_1,
                amount_2 * lp_token_total_supply / reserves_2,
            )
        } else {
            abort EINFINITY_POOL
        };
        ((amount_1 as u64), (amount_2 as u64), (liquidity as u64))
    }

    /// Add liquidity to a pool. The user specifies the desired amount of each token to add and this will add the
    /// optimal amounts. If no optimal amounts can be found, this will fail.
    public entry fun add_liquidity_entry(
        lp: &signer,
        token_1: Object<Metadata>,
        token_2: Object<Metadata>,
        is_stable: bool,
        amount_1_desired: u64,
        amount_2_desired: u64,
        amount_1_min: u64,
        amount_2_min: u64,
    ) {
        let (optimal_amount_1, optimal_amount_2, _) = optimal_liquidity_amounts(
            token_1,
            token_2,
            is_stable,
            amount_1_desired,
            amount_2_desired,
            amount_1_min,
            amount_2_min,
        );
        let optimal_1 = primary_fungible_store::withdraw(lp, token_1, optimal_amount_1);
        let optimal_2 = primary_fungible_store::withdraw(lp, token_2, optimal_amount_2);
        add_liquidity(lp, optimal_1, optimal_2, is_stable);
    }

    /// Add two tokens as liquidity to a pool. The user should have computed the amounts to add themselves as this would
    /// not optimize the amounts.
    public inline fun add_liquidity(
        lp: &signer,
        token_1: FungibleAsset,
        token_2: FungibleAsset,
        is_stable: bool,
    ) {
        liquidity_pool::mint(lp, token_1, token_2, is_stable);
    }

    /// Add a coin and a token as liquidity to a pool. The user specifies the desired amount of each token to add and
    /// this will add the optimal amounts. If no optimal amounts can be found, this will fail.
    public entry fun add_liquidity_coin_entry<CoinType>(
        lp: &signer,
        token_2: Object<Metadata>,
        is_stable: bool,
        amount_1_desired: u64,
        amount_2_desired: u64,
        amount_1_min: u64,
        amount_2_min: u64,
    ) {
        let token_1 = coin_wrapper::get_wrapper<CoinType>();
        let (optimal_amount_1, optimal_amount_2, _) = optimal_liquidity_amounts(
            token_1,
            token_2,
            is_stable,
            amount_1_desired,
            amount_2_desired,
            amount_1_min,
            amount_2_min,
        );
        let optimal_1 = coin_wrapper::wrap(coin::withdraw<CoinType>(lp, optimal_amount_1));
        let optimal_2 = primary_fungible_store::withdraw(lp, token_2, optimal_amount_2);
        add_liquidity(lp, optimal_1, optimal_2, is_stable);
    }

    /// Add a coin and a token as liquidity to a pool. The user should have computed the amounts to add themselves as
    /// this would not optimize the amounts.
    public fun add_liquidity_coin<CoinType>(
        lp: &signer,
        token_1: Coin<CoinType>,
        token_2: FungibleAsset,
        is_stable: bool,
    ) {
        add_liquidity(lp, coin_wrapper::wrap(token_1), token_2, is_stable);
    }

    /// Add two coins as liquidity to a pool. The user specifies the desired amount of each token to add and
    /// this will add the optimal amounts. If no optimal amounts can be found, this will fail.
    public entry fun add_liquidity_both_coin_entry<CoinType1, CoinType2>(
        lp: &signer,
        is_stable: bool,
        amount_1_desired: u64,
        amount_2_desired: u64,
        amount_1_min: u64,
        amount_2_min: u64,
    ) {
        let token_1 = coin_wrapper::get_wrapper<CoinType1>();
        let token_2 = coin_wrapper::get_wrapper<CoinType2>();
        let (optimal_amount_1, optimal_amount_2, _) = optimal_liquidity_amounts(
            token_1,
            token_2,
            is_stable,
            amount_1_desired,
            amount_2_desired,
            amount_1_min,
            amount_2_min,
        );
        let optimal_1 = coin_wrapper::wrap(coin::withdraw<CoinType1>(lp, optimal_amount_1));
        let optimal_2 = coin_wrapper::wrap(coin::withdraw<CoinType2>(lp, optimal_amount_2));
        add_liquidity(lp, optimal_1, optimal_2, is_stable);
    }

    /// Add two coins as liquidity to a pool. The user should have computed the amounts to add themselves as this would
    /// not optimize the amounts.
    public fun add_liquidity_both_coins<CoinType1, CoinType2>(
        lp: &signer,
        token_1: Coin<CoinType1>,
        token_2: Coin<CoinType2>,
        is_stable: bool,
    ) {
        add_liquidity(lp, coin_wrapper::wrap(token_1), coin_wrapper::wrap(token_2), is_stable);
    }

    /// Remove an amount of liquidity from a pool. The user can specify the min amounts of each token they expect to
    /// receive to avoid slippage.
    public entry fun remove_liquidity_entry(
        lp: &signer,
        token_1: Object<Metadata>,
        token_2: Object<Metadata>,
        is_stable: bool,
        liquidity: u64,
        amount_1_min: u64,
        amount_2_min: u64,
        recipient: address,
    ) {
        let (amount_1, amount_2) = remove_liquidity(
            lp,
            token_1,
            token_2,
            is_stable,
            liquidity,
            amount_1_min,
            amount_2_min,
        );
        primary_fungible_store::deposit(recipient, amount_1);
        primary_fungible_store::deposit(recipient, amount_2);
    }

    /// Similiar to `remove_liquidity_entry` but returns the received fungible assets instead of depositing them.
    public fun remove_liquidity(
        lp: &signer,
        token_1: Object<Metadata>,
        token_2: Object<Metadata>,
        is_stable: bool,
        liquidity: u64,
        amount_1_min: u64,
        amount_2_min: u64,
    ): (FungibleAsset, FungibleAsset) {
        assert!(!coin_wrapper::is_wrapper(token_1) && !coin_wrapper::is_wrapper(token_2), ENOT_NATIVE_FUNGIBLE_ASSETS);
        remove_liquidity_internal(lp, token_1, token_2, is_stable, liquidity, amount_1_min, amount_2_min)
    }

    /// Remove liquidity from a pool consisting of a coin and a fungible asset. The user can specify the min amounts of
    /// each token they expect to receive to avoid slippage.
    public entry fun remove_liquidity_coin_entry<CoinType>(
        lp: &signer,
        token_2: Object<Metadata>,
        is_stable: bool,
        liquidity: u64,
        amount_1_min: u64,
        amount_2_min: u64,
        recipient: address,
    ) {
        let (amount_1, amount_2) =
            remove_liquidity_coin<CoinType>(lp, token_2, is_stable, liquidity, amount_1_min, amount_2_min);
        aptos_account::deposit_coins<CoinType>(recipient, amount_1);
        primary_fungible_store::deposit(recipient, amount_2);
    }

    /// Similiar to `remove_liquidity_coin_entry` but returns the received coins and assets instead of depositing them.
    public fun remove_liquidity_coin<CoinType>(
        lp: &signer,
        token_2: Object<Metadata>,
        is_stable: bool,
        liquidity: u64,
        amount_1_min: u64,
        amount_2_min: u64,
    ): (Coin<CoinType>, FungibleAsset) {
        let token_1 = coin_wrapper::get_wrapper<CoinType>();
        assert!(!coin_wrapper::is_wrapper(token_2), ENOT_NATIVE_FUNGIBLE_ASSETS);
        let (amount_1, amount_2) =
            remove_liquidity_internal(lp, token_1, token_2, is_stable, liquidity, amount_1_min, amount_2_min);
        (coin_wrapper::unwrap(amount_1), amount_2)
    }

    /// Remove liquidity from a pool consisting of two coins. The user can specify the min amounts of each token they
    /// expect to receive to avoid slippage.
    public entry fun remove_liquidity_both_coins_entry<CoinType1, CoinType2>(
        lp: &signer,
        is_stable: bool,
        liquidity: u64,
        amount_1_min: u64,
        amount_2_min: u64,
        recipient: address,
    ) {
        let (amount_1, amount_2) =
            remove_liquidity_both_coins<CoinType1, CoinType2>(lp, is_stable, liquidity, amount_1_min, amount_2_min);
        aptos_account::deposit_coins<CoinType1>(recipient, amount_1);
        aptos_account::deposit_coins<CoinType2>(recipient, amount_2);
    }

    /// Similiar to `remove_liquidity_both_coins_entry` but returns the received coins instead of depositing them.
    public fun remove_liquidity_both_coins<CoinType1, CoinType2>(
        lp: &signer,
        is_stable: bool,
        liquidity: u64,
        amount_1_min: u64,
        amount_2_min: u64,
    ): (Coin<CoinType1>, Coin<CoinType2>) {
        let token_1 = coin_wrapper::get_wrapper<CoinType1>();
        let token_2 = coin_wrapper::get_wrapper<CoinType2>();
        let (amount_1, amount_2) =
            remove_liquidity_internal(lp, token_1, token_2, is_stable, liquidity, amount_1_min, amount_2_min);
        (coin_wrapper::unwrap(amount_1), coin_wrapper::unwrap(amount_2))
    }

    inline fun remove_liquidity_internal(
        lp: &signer,
        token_1: Object<Metadata>,
        token_2: Object<Metadata>,
        is_stable: bool,
        liquidity: u64,
        amount_1_min: u64,
        amount_2_min: u64,
    ): (FungibleAsset, FungibleAsset) {
        let (redeemed_1, redeemed_2) = liquidity_pool::burn(lp, token_1, token_2, is_stable, liquidity);
        let amount_1 = fungible_asset::amount(&redeemed_1);
        let amount_2 = fungible_asset::amount(&redeemed_2);
        assert!(amount_1 >= amount_1_min && amount_2 >= amount_2_min, EINSUFFICIENT_OUTPUT_AMOUNT);
        (redeemed_1, redeemed_2)
    }
}
```

files in the ./sources/tests folder:

# test_helpers.move

This file provides essential utility functions and setup procedures for testing the swap project components. Key aspects include:

1. Test Environment Setup:
   - Implements a `set_up` function to initialize the testing environment, including setting up the package manager and liquidity pool modules.
   - Utilizes the `#[test_only]` attribute to ensure these functions are only available in the testing context.

2. Fungible Asset Creation:
   - Provides a utility function `create_fungible_asset_and_mint` to create and mint fungible assets for testing purposes.
   - Demonstrates the use of Aptos Move's object model and fungible asset standard in a testing context.

3. Coin Creation:
   - Implements a `create_coin_and_mint` function to create and mint coins of a specified type for testing.
   - Shows how to interact with Aptos Move's coin module in tests.

4. Modular Design:
   - Designed to be imported and used across various test files in the project, promoting code reuse and consistency in test setups.

5. Resource Account Simulation:
   - Utilizes the package manager's test initialization to simulate resource account behavior in tests.

6. Flexible Asset Generation:
   - Allows for the creation of both fungible assets and coins, supporting comprehensive testing of the swap project's multi-standard token handling.

7. Test Data Customization:
   - Provides parameters in utility functions to allow testers to customize asset properties like name and amount.

8. Aptos Framework Integration:
   - Demonstrates how to interact with core Aptos framework modules (like `aptos_framework::object` and `aptos_framework::fungible_asset`) in a testing context.

9. Error Handling Omission:
   - Deliberately omits extensive error handling to keep test setup code concise, assuming a controlled test environment.

This module plays a crucial role in the testing infrastructure of the swap project. It encapsulates common setup procedures and utility functions, enabling more concise and focused test cases in other test files. By providing a consistent way to create test assets and initialize the test environment, it ensures that all components of the swap project can be thoroughly and uniformly tested. The design of this helper module showcases best practices in creating a robust testing framework for complex Aptos Move projects, facilitating comprehensive unit and integration testing of DeFi applications.

```
#[test_only]
module swap::test_helpers {
    use aptos_framework::coin::{Self, Coin};
    use aptos_framework::fungible_asset::{Self, FungibleAsset};
    use aptos_framework::object;
    use aptos_framework::primary_fungible_store;
    use std::option;
    use std::string;
    use swap::package_manager;
    use swap::liquidity_pool;

    public fun set_up(deployer: &signer) {
        package_manager::initialize_for_test(deployer);
        liquidity_pool::initialize();
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

# coin_wrapper_tests.move

This file contains unit tests for the coin_wrapper module, which is crucial for the swap project's ability to handle both coins and fungible assets. Key aspects include:

1. End-to-End Testing:
   - Implements an e2e (end-to-end) test that covers the full lifecycle of wrapping a coin into a fungible asset and unwrapping it back.
   - Demonstrates the complete flow of creating, wrapping, unwrapping, and verifying coins and fungible assets.

2. Test Environment Setup:
   - Utilizes the test_helpers module to set up the testing environment, including initializing necessary modules and creating test accounts.

3. Fake Money Usage:
   - Leverages Aptos' FakeMoney coin type for testing, showcasing how to use mock coins in a test environment.

4. Assertion-based Verification:
   - Uses assert statements to verify the correctness of various operations, including amount preservation, metadata consistency, and successful unwrapping.

5. Metadata Verification:
   - Checks that the wrapped fungible asset correctly maintains the name, symbol, and decimals of the original coin.

6. Public Test Function:
   - Includes a public wrap function that can be used by other test modules, promoting code reuse in the testing suite.

7. Signer and Account Handling:
   - Demonstrates proper handling of test signers and accounts, crucial for testing permission-based operations in Aptos Move.

8. Error Checking:
   - While not explicitly shown, the comprehensive nature of the test implies checking for correct behavior in the absence of errors.

9. Test Attributes:
   - Uses the #[test] attribute to mark the main test function, integrating with Aptos Move's testing framework.

10. Module Initialization:
    - Indirectly tests the initialization of the coin_wrapper module through the setup process.

This test module plays a vital role in ensuring the reliability and correctness of the coin wrapper functionality, which is fundamental to the swap project's ability to operate across different token standards. By thoroughly testing the wrapping and unwrapping process, including metadata preservation and amount accuracy, it helps guarantee the integrity of token conversions within the swap ecosystem. The design of these tests demonstrates best practices in Aptos Move testing, including environment setup, end-to-end scenario coverage, and detailed state verification.

```
#[test_only]
module swap::coin_wrapper_tests {
    use aptos_framework::account;
    use aptos_framework::coin::{Self, Coin, FakeMoney};
    use aptos_framework::fungible_asset::{Self, FungibleAsset};
    use swap::coin_wrapper;
    use swap::test_helpers;
    use std::signer;

    #[test(user = @0x1, deployer = @0xcafe)]
    fun test_e2e(user: &signer, deployer: &signer) {
        test_helpers::set_up(deployer);
        account::create_account_for_test(signer::address_of(user));
        coin::create_fake_money(user, user, 1000);
        let coins = coin::withdraw<FakeMoney>(user, 1000);
        let fa = coin_wrapper::wrap(coins);
        let metadata = fungible_asset::asset_metadata(&fa);
        assert!(fungible_asset::amount(&fa) == 1000, 0);
        assert!(fungible_asset::name(metadata) == coin::name<FakeMoney>(), 0);
        assert!(fungible_asset::symbol(metadata) == coin::symbol<FakeMoney>(), 0);
        assert!(fungible_asset::decimals(metadata) == coin::decimals<FakeMoney>(), 0);
        let coins = coin_wrapper::unwrap<FakeMoney>(fa);
        assert!(coin::value(&coins) == 1000, 0);
        coin::deposit(signer::address_of(user), coins);
    }

    public fun wrap<CoinType>(coin: Coin<CoinType>): FungibleAsset {
        coin_wrapper::wrap(coin)
    }
}
```

# liquidity_pool_tests.move

This file contains extensive tests for the liquidity_pool module, which is the core component of the swap project. Key aspects include:

1. Comprehensive Testing:
   - Implements end-to-end (e2e) tests for both volatile and stable liquidity pools, covering the full lifecycle of pool operations.
   - Tests include pool creation, adding liquidity, swapping tokens, removing liquidity, and fee collection.

2. Volatile vs Stable Pools:
   - Separate test functions for volatile and stable pools, ensuring both pool types are thoroughly tested.
   - Demonstrates the different behaviors and calculations for each pool type.

3. Multi-user Scenarios:
   - Uses multiple test signers (lp_1, lp_2) to simulate different users interacting with the pools, testing various user perspectives and interactions.

4. Liquidity Provider (LP) Token Handling:
   - Tests minting, transferring, and burning of LP tokens.
   - Verifies correct distribution of assets when removing liquidity.

5. Fee Accrual and Collection:
   - Tests the fee collection mechanism, including accurate fee calculation and distribution to liquidity providers.

6. Swap Functionality:
   - Implements and verifies swap operations, including checks for correct output amounts and pool balance updates.

7. Reserve Verification:
   - Uses a custom verify_reserves function to ensure pool reserves are correctly updated after each operation.

8. Error Checking:
   - While not explicitly shown for every case, the comprehensive nature of the tests implies checking for correct behavior in various scenarios, including edge cases.

9. Test Helpers Usage:
   - Leverages utility functions (like create_pool and add_liquidity) to set up test scenarios efficiently.

10. Event Verification:
    - Indirectly tests event emissions through the operations that should trigger events in the liquidity_pool module.

11. Complex State Management:
    - Tests complex state changes, including transferring LP tokens between accounts and verifying correct fee distribution after ownership changes.

12. Mathematical Precision:
    - Implicitly tests the mathematical precision of pool operations, especially important for stable pools with more complex formulas.

This test module is crucial for ensuring the reliability, security, and correctness of the liquidity pool functionality, which is the heart of the swap project. By thoroughly testing various scenarios, including multi-user interactions, different pool types, and complex operations like fee accrual and distribution, it helps guarantee the integrity of the AMM (Automated Market Maker) system. The design of these tests demonstrates advanced practices in Aptos Move testing, including comprehensive scenario coverage, detailed state verification, and simulation of complex DeFi operations.

```
#[test_only]
module swap::liquidity_pool_tests {
    use aptos_framework::fungible_asset::{Self, FungibleAsset};
    use aptos_framework::object::Object;
    use aptos_framework::primary_fungible_store;
    use swap::liquidity_pool::{Self, LiquidityPool};
    use swap::test_helpers;
    use std::signer;
    use std::vector;

    #[test(lp_1 = @0xcafe, lp_2 = @0xdead)]
    fun test_e2e_volatile(lp_1: &signer, lp_2: &signer) {
        test_helpers::set_up(lp_1);
        let is_stable = false;
        let (pool, tokens_2, tokens_1) = create_pool(lp_1, is_stable);

        // Add liquidity to the pool
        add_liquidity(lp_1, &mut tokens_1, &mut tokens_2, 100000, 200000, is_stable);
        verify_reserves(pool, 100000, 200000);
        add_liquidity(lp_1, &mut tokens_1, &mut tokens_2, 100000, 100000, is_stable);
        verify_reserves(pool, 200000, 300000);

        // Swap tokens
        let (amount_out_1, fees_2) = swap_and_verify(lp_1, pool, &mut tokens_2, 10000);
        let (amount_out_2, fees_1) = swap_and_verify(lp_1, pool, &mut tokens_1, 10000);
        let expected_total_reserves_1 = ((200000 - amount_out_1 - fees_1 + 10000) as u128);
        let expected_total_reserves_2 = ((300000 - amount_out_2 - fees_2 + 10000) as u128);
        verify_reserves(pool, (expected_total_reserves_1 as u64), (expected_total_reserves_2 as u64));

        // Remove half of the liquidity. Should receive the proportional amounts of tokens back.
        let lp_tokens_to_withdraw = ((primary_fungible_store::balance(signer::address_of(lp_1), pool) / 2) as u128);
        let total_lp_supply = liquidity_pool::lp_token_supply(pool);
        let expected_liq_1 = expected_total_reserves_1 * lp_tokens_to_withdraw / total_lp_supply;
        let expected_liq_2 = expected_total_reserves_2 * lp_tokens_to_withdraw / total_lp_supply;
        let (liq_1, liq_2) = liquidity_pool::burn(
            lp_1,
            fungible_asset::asset_metadata(&tokens_1),
            fungible_asset::asset_metadata(&tokens_2),
            is_stable,
            (lp_tokens_to_withdraw as u64),
        );
        assert!(fungible_asset::amount(&liq_1) == (expected_liq_1 as u64), 0);
        assert!(fungible_asset::amount(&liq_2) == (expected_liq_2 as u64), 0);
        fungible_asset::merge(&mut tokens_1, liq_1);
        fungible_asset::merge(&mut tokens_2, liq_2);

        // There's a second account here - the lp_2 who tries to create the store for the LP token before hand.
        let lp_2_address = signer::address_of(lp_2);
        primary_fungible_store::ensure_primary_store_exists(lp_2_address, pool);
        assert!(primary_fungible_store::primary_store_exists(lp_2_address, pool), 0);
        assert!(!primary_fungible_store::is_frozen(lp_2_address, pool), 0);

        // Original LP transfers all LP tokens to the lp_2, which should also automatically locks their token store
        // to ensure they have to call liquidity_pool::transfer to do the actual transfer.
        let lp_1_address = signer::address_of(lp_1);
        let lp_balance = primary_fungible_store::balance(lp_1_address, pool);
        liquidity_pool::transfer(lp_1, pool, lp_2_address, lp_balance);
        assert!(primary_fungible_store::is_frozen(lp_2_address, pool), 0);
        assert!(primary_fungible_store::balance(lp_2_address, pool) == lp_balance, 0);

        // The original LP should still receive the fees for the swap that already happens, which is a bit less due to
        // the min liquidity.
        let (claimed_1, claimed_2) = liquidity_pool::claim_fees(lp_1, pool);
        assert!(fungible_asset::amount(&claimed_1) == fees_1 - 1, 0);
        assert!(fungible_asset::amount(&claimed_2) == fees_2 - 1, 0);
        primary_fungible_store::deposit(lp_1_address, claimed_1);
        primary_fungible_store::deposit(lp_1_address, claimed_2);

        // No more rewards to claim.
        let (remaining_1, remaining_2) = liquidity_pool::claimable_fees(lp_1_address, pool);
        assert!(remaining_1 == 0, 0);
        assert!(remaining_2 == 0, 0);

        // More swaps happen. Now the new LP should receive fees. Original LP shouldn't receive anymore.
        let (_, fees_1) = swap_and_verify(lp_1, pool, &mut tokens_1, 10000);
        let (claimed_1, claimed_2) = liquidity_pool::claim_fees(lp_2, pool);
        assert!(fungible_asset::amount(&claimed_1) == fees_1 - 1, 0);
        assert!(fungible_asset::amount(&claimed_2) == 0, 0);
        let (original_claimable_1, original_claimable_2) = liquidity_pool::claimable_fees(lp_1_address, pool);
        assert!(original_claimable_1 == 0 && original_claimable_2 == 0, 0);
        primary_fungible_store::deposit(lp_1_address, claimed_1);
        primary_fungible_store::deposit(lp_1_address, claimed_2);

        primary_fungible_store::deposit(lp_1_address, tokens_1);
        primary_fungible_store::deposit(lp_1_address, tokens_2);
    }

    #[test(lp_1 = @0xcafe)]
    fun test_e2e_stable(lp_1: &signer) {
        test_helpers::set_up(lp_1);
        let is_stable = true;
        let (pool, tokens_2, tokens_1) = create_pool(lp_1, is_stable);
        add_liquidity(lp_1, &mut tokens_1, &mut tokens_2, 100000, 200000, is_stable);
        swap_and_verify(lp_1, pool, &mut tokens_2, 10000);
        swap_and_verify(lp_1, pool, &mut tokens_1, 10000);

        let lp_1_address = signer::address_of(lp_1);
        primary_fungible_store::deposit(lp_1_address, tokens_1);
        primary_fungible_store::deposit(lp_1_address, tokens_2);
    }

    fun verify_reserves(pool: Object<LiquidityPool>, expected_1: u64, expected_2: u64) {
        let (reserves_1, reserves_2) = liquidity_pool::pool_reserves(pool);
        assert!(reserves_1 == expected_1, 0);
        assert!(reserves_2 == expected_2, 0);
    }

    public fun swap_and_verify(
        lp_1: &signer,
        pool: Object<LiquidityPool>,
        tokens: &mut FungibleAsset,
        amount_in: u64,
    ): (u64, u64) {
        let (reserves_1, reserves_2) = liquidity_pool::pool_reserves(pool);
        let pool_assets = liquidity_pool::supported_inner_assets(pool);
        if (fungible_asset::asset_metadata(tokens) != *vector::borrow(&pool_assets, 0)) {
            (reserves_1, reserves_2) = (reserves_2, reserves_1);
        };
        let out = liquidity_pool::swap(pool, fungible_asset::extract(tokens, amount_in));

        let fees_amount = liquidity_pool::swap_fee_bps(pool) * amount_in / 10000;
        amount_in = amount_in - fees_amount;
        let actual_amount = fungible_asset::amount(&out);
        if (!liquidity_pool::is_stable(pool)) {
            assert!(actual_amount == amount_in * reserves_2 / (reserves_1 + amount_in), 0);
        };
        primary_fungible_store::deposit(signer::address_of(lp_1), out);
        (actual_amount, fees_amount)
    }

    public fun add_liquidity(
        lp: &signer,
        tokens_1: &mut FungibleAsset,
        tokens_2: &mut FungibleAsset,
        amount_1: u64,
        amount_2: u64,
        is_stable: bool,
    ) {
        liquidity_pool::mint(
            lp,
            fungible_asset::extract(tokens_1, amount_1),
            fungible_asset::extract(tokens_2, amount_2),
            is_stable,
        );
    }

    public fun create_pool(lp_1: &signer, is_stable: bool): (Object<LiquidityPool>, FungibleAsset, FungibleAsset) {
        let tokens_1 = test_helpers::create_fungible_asset_and_mint(lp_1, b"test1", 10000000);
        let tokens_2 = test_helpers::create_fungible_asset_and_mint(lp_1, b"test2", 10000000);
        let pool = liquidity_pool::create(
            fungible_asset::asset_metadata(&tokens_1),
            fungible_asset::asset_metadata(&tokens_2),
            is_stable,
        );
        (pool, tokens_1, tokens_2)
    }
}
```

# package_manager_tests.move

This file contains unit tests for the package_manager module, which is fundamental to the security and permission management of the entire swap project. Key aspects include:

1. Resource Account Testing:
   - Verifies the correct initialization and functionality of the resource account, which is crucial for the project's permission model.

2. Signer Capability Verification:
   - Tests the ability to retrieve and use the signer capability associated with the resource account.

3. Address Management:
   - Checks the functionality of setting and retrieving addresses, a core feature of the package manager for tracking various components of the swap project.

4. Test Environment Setup:
   - Utilizes the initialize_for_test function to set up the testing environment, simulating the deployment of the package manager.

5. Permission Verification:
   - Implicitly tests the permission model by ensuring only authorized operations can be performed.

6. Error Handling:
   - While not explicitly shown, the nature of the tests implies checking for correct behavior in both successful and error scenarios.

7. Module Initialization:
   - Tests the proper initialization of the package_manager module in a test context.

8. Assertion-based Verification:
   - Uses assert statements to verify the correctness of operations, including address retrieval and signer capability functionality.

9. Test Attributes:
   - Employs the #[test] attribute to mark test functions, integrating with Aptos Move's testing framework.

10. Signer Handling:
    - Demonstrates proper creation and use of test signers, crucial for testing permission-based operations.

11. String Handling:
    - Tests the use of string keys for address management, verifying the package manager's ability to handle named addresses.

This test module is essential for ensuring the integrity and security of the swap project's foundation. By thoroughly testing the package manager's functionality, including resource account management, address tracking, and permission controls, it helps guarantee that the entire swap ecosystem operates on a secure and well-managed base. The design of these tests showcases best practices in Aptos Move testing for critical infrastructure components, ensuring that the core management and security features of the project are robust and reliable.

```
#[test_only]
module swap::package_manager_tests {
    use std::signer;
    use std::string;
    use swap::package_manager;

    #[test(deployer = @0xcafe)]
    public fun test_can_get_signer(deployer: &signer) {
        package_manager::initialize_for_test(deployer);
        let deployer_addr = signer::address_of(deployer);
        assert!(signer::address_of(&package_manager::get_signer()) == deployer_addr, 0);
    }

    #[test(deployer = @0xcafe)]
    public fun test_can_set_and_get_address(deployer: &signer) {
        package_manager::initialize_for_test(deployer);
        package_manager::add_address(string::utf8(b"test"), @0xdeadbeef);
        assert!(package_manager::get_address(string::utf8(b"test")) == @0xdeadbeef, 0);
    }
}
```

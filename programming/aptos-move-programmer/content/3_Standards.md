# Aptos Standards

This document provides a comprehensive overview of the key standards used in Aptos Move development. We'll cover the modern Digital Asset (DA) and Fungible Asset (FA) standards, as well as provide an overview of the legacy standards for context and migration considerations.

## Table of Contents
1. Digital Asset (DA) Standard
2. Fungible Asset (FA) Standard
3. Legacy Standards Overview

## 1. Digital Asset (DA) Standard

The Digital Asset (DA) standard is Aptos' modern implementation for Non-Fungible Tokens (NFTs). It leverages the Aptos Move Object model to provide a flexible and powerful way to represent unique digital assets on the blockchain.

### 1.1 Key Concepts

- **Objects**: The DA standard uses Move Objects to represent NFTs, allowing for greater flexibility and extensibility.
- **Collections**: Groups of NFTs with shared properties and metadata.
- **Tokens**: Individual digital assets within a collection.

### 1.2 Improvements Over Legacy Token Standard

1. **Token Extension**: Tokens can be easily extended due to their implementation as Move Objects.
2. **Direct NFT Transfer**: Recipients no longer need to "opt-in" on-chain to receive NFTs.
3. **NFT Composability**: NFTs can own other NFTs, enabling complex asset structures.

### 1.3 Implementation Guidelines

#### Creating a Collection

Collections are created using the `collection::create_fixed_collection` or `collection::create_unlimited_collection` functions:

```move
use aptos_token_objects::collection;
use std::string;
use std::option;

public entry fun create_collection(creator: &signer) {
    let description = string::utf8(b"My Collection Description");
    let name = string::utf8(b"My Collection");
    let uri = string::utf8(b"https://example.com/my-collection");
    let max_supply = 1000; // Use 0 for unlimited supply

    collection::create_fixed_collection(
        creator,
        description,
        max_supply,
        name,
        option::none(), // No royalty
        uri
    );
}
```

#### Creating a Token

Tokens are created using the `token::create_from_account` function:

```move
use aptos_token_objects::token;
use std::string;
use std::option;

public entry fun create_token(creator: &signer) {
    let collection = string::utf8(b"My Collection");
    let description = string::utf8(b"My Token Description");
    let name = string::utf8(b"My Token");
    let uri = string::utf8(b"https://example.com/my-token");
    let royalty = option::none();

    token::create_from_account(
        creator,
        collection,
        description,
        name,
        royalty,
        uri
    );
}
```

#### Token Transfer

Tokens can be transferred using the `object::transfer` function:

```move
use aptos_framework::object;

public entry fun transfer_token<T: key>(from: &signer, to: address, token: object::Object<T>) {
    object::transfer(from, token, to);
}
```

### 1.4 Best Practices

1. Use meaningful and unique names for collections and tokens.
2. Store extensive metadata off-chain and reference it via the `uri` field.
3. Implement custom logic by extending token objects with additional resources.
4. Use `create_named_token` for important tokens that shouldn't be deletable.

## 2. Fungible Asset (FA) Standard

The Fungible Asset (FA) standard provides a flexible and extensible way to create and manage fungible tokens on Aptos. It replaces the legacy Coin standard with a more powerful Object-based implementation.

### 2.1 Key Concepts

- **Metadata**: Represents details about the fungible asset (name, symbol, decimals, etc.).
- **FungibleStore**: Stores the balance of a particular fungible asset for an account.
- **Primary Store**: The default store for a fungible asset in an account.
- **Refs**: Special capabilities for minting, burning, and transferring assets.

### 2.2 Improvements Over Legacy Coin Standard

1. **Extensibility**: FA leverages Move Objects for greater customization.
2. **Multiple Stores**: Accounts can have multiple stores for the same asset type.
3. **Automatic Store Creation**: Recipients don't need to explicitly create stores.
4. **Rich Metadata**: More flexible and extensive metadata options.

### 2.3 Implementation Guidelines

#### Creating a Fungible Asset

```move
use aptos_framework::fungible_asset;
use aptos_framework::object;
use std::string;
use std::option;

public entry fun create_fungible_asset(creator: &signer) {
    let constructor_ref = object::create_named_object(creator, b"MyFA");

    fungible_asset::create_fungible_asset(
        &constructor_ref,
        option::none(), // No maximum supply
        string::utf8(b"My Fungible Asset"),
        string::utf8(b"MFA"),
        8, // Decimals
        string::utf8(b"https://example.com/my-fa-icon"),
        string::utf8(b"https://example.com/my-fa")
    );
}
```

#### Minting Fungible Assets

```move
use aptos_framework::fungible_asset;

public entry fun mint_fa(admin: &signer, amount: u64, recipient: address) acquires MintRef {
    let mint_ref = borrow_global<MintRef>(@my_address);
    let fa = fungible_asset::mint(&mint_ref.0, amount);
    fungible_asset::deposit(recipient, fa);
}
```

#### Transferring Fungible Assets

```move
use aptos_framework::fungible_asset;
use aptos_framework::object;

public entry fun transfer_fa(from: &signer, to: address, fa_address: address, amount: u64) {
    let fa = object::address_to_object<fungible_asset::Metadata>(fa_address);
    fungible_asset::transfer(from, fa, to, amount);
}
```

### 2.4 Best Practices

1. Use named objects for important fungible assets to ensure they're not deletable.
2. Carefully manage mint, burn, and transfer capabilities.
3. Consider using secondary stores for advanced use cases like escrow or vesting.
4. Implement proper access control for administrative functions.

## 3. Legacy Standards Overview

Understanding the legacy standards is crucial for maintaining existing projects and planning migrations to the new standards.

### 3.1 Legacy Coin Standard

The legacy Coin standard was the original implementation for fungible tokens on Aptos.

Key characteristics:
- Uses a `Coin<T>` struct to represent fungible tokens.
- Requires explicit `CoinStore<T>` registration for users to hold tokens.
- Uses capability-based access control for minting and burning.

Example of creating a coin type:

```move
struct MyCoin {}

public fun initialize(account: &signer) {
    aptos_framework::managed_coin::initialize<MyCoin>(
        account,
        b"My Coin",
        b"MC",
        6,
        false,
    );
}
```

### 3.2 Legacy Token Standard

The legacy Token standard was the original NFT implementation on Aptos.

Key characteristics:
- Uses a `TOKEN_DATA_ID` to identify token types.
- Supports semi-fungible tokens.
- Uses property maps for custom attributes.

Example of creating a token:

```move
public entry fun create_token(
    account: &signer,
    collection: String,
    name: String,
    description: String,
    supply: u64,
    uri: String,
) {
    token::create_token_script(
        account,
        collection,
        name,
        description,
        supply,
        1, // maximum
        uri,
        creator_address,
        0, // royalty_points_denominator
        0, // royalty_points_numerator
        vector<String>[],
        vector<vector<u8>>[],
        vector<String>[]
    );
}
```

### 3.3 Migration Considerations

When migrating from legacy standards to the new DA and FA standards, consider the following:

1. **Data Migration**: Develop a strategy to migrate existing token data to the new Object-based model.
2. **Smart Contract Updates**: Refactor smart contracts to use the new standard APIs.
3. **Indexer Updates**: Update any off-chain indexers to handle both legacy and new events/data structures.
4. **User Experience**: Plan for a smooth transition in user-facing applications.
5. **Testing**: Thoroughly test all migrations in a test environment before deploying to mainnet.

## References

- [Aptos Digital Asset Standard](https://aptos.dev/en/build/smart-contracts/digital-asset)
- [Aptos Fungible Asset Standard](https://aptos.dev/en/build/smart-contracts/fungible-asset)
- [Legacy Coin Standard](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-framework/sources/coin.move)
- [Legacy Token Standard](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-token/sources/token.move)
- [Aptos Standards Comparison](https://aptos.dev/en/build/guides/nfts/aptos-token-overview)

These resources provide more detailed information and the most up-to-date specifications for each standard.
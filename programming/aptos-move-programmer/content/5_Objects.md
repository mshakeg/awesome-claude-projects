## Move Objects Chapter from the Aptos Move Book

In Move, Objects group resources together so they can be treated as a single entity on chain.

Objects have their own address and can own resources similar to an account. They are useful for representing more complicated data types on-chain as Objects can be used in entry functions directly, and can be transferred as complete packages instead of one resource at a time.

Here’s an example of creating an Object and transferring it:

```move
module my_addr::object_playground {
  use std::signer;
  use std::string::{Self, String};
  use aptos_framework::object::{Self, ObjectCore};

  struct MyStruct1 {
    message: String,
  }

  struct MyStruct2 {
    message: String,
  }

  entry fun create_and_transfer(caller: &signer, destination: address) {
    // Create object
    let caller_address = signer::address_of(caller);
    let constructor_ref = object::create_object(caller_address);
    let object_signer = object::generate_signer(constructor_ref);

    // Set up the object by creating 2 resources in it
    move_to(&object_signer, MyStruct1 {
      message: string::utf8(b"hello")
    });
    move_to(&object_signer, MyStruct2 {
      message: string::utf8(b"world")
    });

    // Transfer to destination
    let object = object::object_from_constructor_ref<ObjectCore>(
      &constructor_ref
    );
    object::transfer(caller, object, destination);
  }
}
```

During construction, Objects can be configured to be transferrable, burnable, and extensible.

For example, you could use an Object to represent a soulbound NFT by making it only transferrable once, and have it own resources for an image link and metadata. Objects can also own other Objects, so you could implement your own NFT collection Object by transferring several of the soulbound NFTs to it.

### Creating and Configuring Objects

Creating an Object involves two steps:

Creating the ObjectCore resource group (which has an address you can use to refer to the Object later).
Customizing how the Object will behave using permissions called Refs.
Configuring an Object by generating Refs has to happen in the same transaction you create it. Later on it is impossible to change those settings.

#### Creating an Object

There are three types of Object you can create:

A normal Object. This type is deletable and has a random address. You can create it using: 0x1::object::create_object(owner_address: address). For example:
object_playground.move

```move
module my_addr::object_playground {
  use std::signer;
  use aptos_framework::object;

  entry fun create_my_object(caller: &signer) {
    let caller_address = signer::address_of(caller);
    let constructor_ref = object::create_object(caller_address);
    // ...
  }
}
```

A named Object. This type is not deletable and has a deterministic address. You can create it by using: 0x1::object::create_named_object(creator: &signer, seed: vector<u8>). For example:
object_playground.move

```move
module my_addr::object_playground {
  use std::signer;
  use aptos_framework::object;

  /// Seed for my named object, must be globally unique to the creating account
  const NAME: vector<u8> = b"MyAwesomeObject";

  entry fun create_my_object(caller: &signer) {
    let caller_address = signer::address_of(caller);
    let constructor_ref = object::create_named_object(caller, NAME);
    // ...
  }

  #[view]
  fun has_object(creator: address): bool {
    let object_address = object::create_object_address(&creator, NAME);
    object_exists<0x1::object::ObjectCore>(object_address)
  }
}
```

A sticky Object. This type is also not deletable and has a random address. You can create it by using 0x1::object::create_sticky_object(owner_address: address). For example:
object_playground.move

```move
module my_addr::object_playground {
  use std::signer;
  use aptos_framework::object;

  entry fun create_my_object(caller: &signer) {
    let caller_address = signer::address_of(caller);
    let constructor_ref = object::create_sticky_object(caller_address);
    // ...
  }
}
```

#### Customizing Object Features

Once you create your object, you will receive a ConstructorRef you can use to generate additional Refs. Refs can be used in future to enable / disable / execute certain Object functions such as transferring resources, transferring the object itself, or deleting the Object.

The following sections will walk through commonly used Refs and the features they enable.

Note: The ConstructorRef cannot be stored. It is destroyed at the end of the transaction used to create the Object, so any other Refs must be generated during Object creation.

#### Adding Resources
You can use the ConstructorRef with object::generate_signer to create a signer that allows you to transfer resources onto the Object. This uses move_to, the same function as for adding resources to an account.

Example.move

```move
module my_addr::object_playground {
  use std::signer;
  use aptos_framework::object;

  #[resource_group_member(group = aptos_framework::object::ObjectGroup)]
  struct MyStruct has key {
    num: u8
  }

  entry fun create_my_object(caller: &signer) {
    let caller_address = signer::address_of(caller);

    // Creates the object
    let constructor_ref = object::create_object(caller_address);

    // Retrieves a signer for the object
    let object_signer = object::generate_signer(&constructor_ref);

    // Moves the MyStruct resource into the object
    move_to(&object_signer, MyStruct { num: 0 });

    // ...
  }
}
```

#### Adding Extensibility (`ExtendRef`)

Sometimes you want an Object to be editable later on. In that case, you can generate an ExtendRef with object::generate_extend_ref. This ref can be used to generate a signer for the object.

You can control who has permission to use the ExtendRef via smart contract logic like in the below example.

Example.move

```
module my_addr::object_playground {
  use std::signer;
  use std::string::{self, String};
  use aptos_framework::object::{self, Object};

  /// Caller is not the owner of the object
  const E_NOT_OWNER: u64 = 1;
  /// Caller is not the publisher of the contract
  const E_NOT_PUBLISHER: u64 = 2;

  #[resource_group_member(group = aptos_framework::object::ObjectGroup)]
  struct MyStruct has key {
    num: u8
  }

  #[resource_group_member(group = aptos_framework::object::ObjectGroup)]
  struct Message has key {
    message: string::String
  }

  #[resource_group_member(group = aptos_framework::object::ObjectGroup)]
  struct ObjectController has key {
    extend_ref: object::ExtendRef,
  }

  entry fun create_my_object(caller: &signer) {
    let caller_address = signer::address_of(caller);

    // Creates the object
    let constructor_ref = object::create_object(caller_address);

    // Retrieves a signer for the object
    let object_signer = object::generate_signer(&constructor_ref);

    // Moves the MyStruct resource into the object
    move_to(&object_signer, MyStruct { num: 0 });

    // Creates an extend ref, and moves it to the object
    let extend_ref = object::generate_extend_ref(&constructor_ref);
    move_to(&object_signer, ObjectController { extend_ref });
    // ...
  }

  entry fun add_message(
    caller: &signer,
    object: Object<MyStruct>,
    message: String
  ) acquires ObjectController {
    let caller_address = signer::address_of(caller);
    // There are a couple ways to go about permissions

    // Allow only the owner of the object
    assert!(object::is_owner(object, caller_address), E_NOT_OWNER);
    // Allow only the publisher of the contract
    assert!(caller_address == @my_addr, E_NOT_PUBLISHER);
    // Or any other permission scheme you can think of, the possibilities are endless!

    // Use the extend ref to get a signer
    let object_address = object::object_address(object);
    let extend_ref = borrow_global<ObjectController>(object_address).extend_ref;
    let object_signer = object::generate_signer_for_extending(&extend_ref);

    // Extend the object to have a message
    move_to(object_signer, Message { message });
  }
}
```

#### Disabling / Toggling Transfers (`TransferRef`)

By default, all Objects are transferable. This can be changed via a TransferRef which you can generate with object::generate_transfer_ref.

The example below shows how you could generate and manage permissions for determining whether an Object is transferrable.

Example.move

```move
module my_addr::object_playground {
  use std::signer;
  use std::string::{self, String};
  use aptos_framework::object::{self, Object};

  /// Caller is not the publisher of the contract
  const E_NOT_PUBLISHER: u64 = 1;

  #[resource_group_member(group = aptos_framework::object::ObjectGroup)]
  struct ObjectController has key {
    transfer_ref: object::TransferRef,
  }

  entry fun create_my_object(
    caller: &signer,
    transferrable: bool,
    controllable: bool
  ) {
    let caller_address = signer::address_of(caller);

    // Creates the object
    let constructor_ref = object::create_object(caller_address);

    // Retrieves a signer for the object
    let object_signer = object::generate_signer(&constructor_ref);

    // Creates a transfer ref for controlling transfers
    let transfer_ref = object::generate_transfer_ref(&constructor_ref);

    // We now have a choice, we can make it so the object can be transferred
    // and we can decide if we want to allow it to change later.  By default, it
    // is transferrable
    if (!transferrable) {
      object::disable_ungated_transfer(&transfer_ref);
    };

    // If we want it to be controllable, we must store the transfer ref for later
    if (controllable) {
      move_to(&object_signer, ObjectController { transfer_ref });
    }
    // ...
  }

  /// In this example, we'll only let the publisher of the contract change the
  /// permissions of transferring
  entry fun toggle_transfer(
    caller: &signer,
    object: Object<ObjectController>
  ) acquires ObjectController {
    // Only let the publisher toggle transfers
    let caller_address = signer::address_of(caller);
    assert!(caller_address == @my_addr, E_NOT_PUBLISHER);

    // Retrieve the transfer ref
    let object_address =

 object::object_address(object);
    let transfer_ref = borrow_global<ObjectController>(
      object_address
    ).transfer_ref;

    // Toggle it based on its current state
    if (object::ungated_transfer_allowed(object)) {
      object::disable_ungated_transfer(&transfer_ref);
    } else {
      object::enable_ungated_transfer(&transfer_ref);
    }
  }
}
```

#### One-Time Transfers (`LinearTransferRef`)

Additionally, if the creator wants to control all transfers, a LinearTransferRef can be created from the TransferRef to provide a one time use transfer functionality. This can be used to create “soulbound” objects by having a one-time transfer from the Object creator to the recipient. The LinearTransferRef must be used by the owner of the Object.

Example.move

```
module my_addr::object_playground {
  use std::signer;
  use std::option;
  use std::string::{self, String};
  use aptos_framework::object::{self, Object};

  /// Caller is not the publisher of the contract
  const E_NOT_PUBLISHER: u64 = 1;

  #[resource_group_member(group = aptos_framework::object::ObjectGroup)]
  struct ObjectController has key {
    transfer_ref: object::TransferRef,
    linear_transfer_ref: option::Option<object::LinearTransferRef>,
  }

  entry fun create_my_object(
    caller: &signer,
    transferrable: bool,
    controllable: bool
  ) {
    let caller_address = signer::address_of(caller);

    // Creates the object
    let constructor_ref = object::create_object(caller_address);

    // Retrieves a signer for the object
    let object_signer = object::generate_signer(&constructor_ref);

    // Creates a transfer ref for controlling transfers
    let transfer_ref = object::generate_transfer_ref(&constructor_ref);

    // Disable ungated transfer
    object::disable_ungated_transfer(&transfer_ref);
    move_to(&object_signer, ObjectController {
      transfer_ref,
      linear_transfer_ref: option::none(),
    });
    // ...
  }

  /// In this example, we'll only let the publisher of the contract change the
  /// permissions of transferring
  entry fun allow_single_transfer(
    caller: &signer,
    object: Object<ObjectController>
  ) acquires ObjectController {
    // Only let the publisher toggle transfers
    let caller_address = signer::address_of(caller);
    assert!(caller_address == @my_addr, E_NOT_PUBLISHER);

    let object_address = object::object_address(object);

    // Retrieve the transfer ref
    let transfer_ref = borrow_global<ObjectController>(
      object_address
    ).transfer_ref;

    // Generate a one time use `LinearTransferRef`
    let linear_transfer_ref = object::generate_linear_transfer_ref(
      &transfer_ref
    );

    // Store it for later usage
    let object_controller = borrow_global_mut<ObjectController>(
      object_address
    );
    option::fill(
      &mut object_controller.linear_transfer_ref,
      linear_transfer_ref
    )
  }

  /// Now only owner can transfer exactly once
  entry fun transfer(
    caller: &signer,
    object: Object<ObjectController>,
    new_owner: address
  ) acquires ObjectController {
    let object_address = object::object_address(object);

    // Retrieve the linear_transfer ref, it is consumed so it must be extracted
    // from the resource
    let object_controller = borrow_global_mut<ObjectController>(
      object_address
    );
    let linear_transfer_ref = option::extract(
      &mut object_controller.linear_transfer_ref
    );

    object::transfer_with_ref(linear_transfer_ref, new_owner);
  }
}
```

#### Allowing Deletion of an Object (`DeleteRef`)

For Objects created with the default method (allowing deletion) you can generate a DeleteRef which can be used later. This can help remove clutter as well as receive a storage refund.

You cannot create a DeleteRef for a non-deletable Object.

Example.move

```move
module my_addr::object_playground {
  use std::signer;
  use std::option;
  use std::string::{self, String};
  use aptos_framework::object::{self, Object};

  /// Caller is not the owner of the object
  const E_NOT_OWNER: u64 = 1;

  #[resource_group_member(group = aptos_framework::object::ObjectGroup)]
  struct ObjectController has key {
    delete_ref: object::DeleteRef,
  }

  entry fun create_my_object(
    caller: &signer,
    transferrable: bool,
    controllable: bool
  ) {
    let caller_address = signer::address_of(caller);

    // Creates the object
    let constructor_ref = object::create_object(caller_address);

    // Retrieves a signer for the object
    let object_signer = object::generate_signer(&constructor_ref);

    // Creates and store the delete ref
    let delete_ref = object::generate_delete_ref(&constructor_ref);
    move_to(&object_signer, ObjectController {
      delete_ref
    });
    // ...
  }

  /// Now only let the owner delete the object
  entry fun delete(
    caller: &signer,
    object: Object<ObjectController>,
  ) {
    // Only let caller delete
    let caller_address = signer::address_of(caller);
    assert!(object::is_owner(&object, caller_address), E_NOT_OWNER);

    let object_address = object::object_address(object);

    // Retrieve the delete ref, it is consumed so it must be extracted
    // from the resource
    let ObjectController {
      delete_ref
    } = move_from<ObjectController>(
      object_address
    );

    // Delete the object forever!
    object::delete(delete_ref);
  }
}
```

#### Making an Object Immutable

An object can be made immutable by making the contract associated immutable, and removing any ability to extend or mutate the object. By default, contracts are not immutable, and objects can be extended with an ExtendRef, and resources can be mutated if the contract allows for it.

### Using Objects

Once you’ve created your Object, you can use it in Move entry functions, structs, transfer it, and modify it using any refs you generated during Object construction. Below are various ways to utilize, manage, and interact with Objects in Move.

#### Using an Object as an entry function argument

Objects in move functions have the type Object<T>, where T is the type of a resource owned by the Object. All Objects have an ObjectCore type which contains the metadata for the Object.

To use an Object parameter, users can pass in the Object address or a reference to the Object. At runtime the contract will verify that the Object exists at that address, and has a resource of type T before executing the function.

Example.move

module my_addr::object_playground {
  use aptos_framework::object::Object;

  struct MyAwesomeStruct has key {}

  /// This will fail if the object doesn't have MyAwesomeStruct stored
  entry fun do_something(object: Object<MyAwesomeStruct>) {
    // ...
  }

  /// All Objects have ObjectCore, so this will only fail if the
  /// address is not an object
  entry fun do_something_to_object_core(object: Object<ObjectCore>) {
    // ...
  }
}
To let the user of the entry function specify the type of resource, you can keep the generic type T like so:

Example.move

module my_addr::object_playground {
  use aptos_framework::object::Object;

  /// This will fail if the object doesn't have the generic `T` stored
  entry fun do_something<T>(object: Object<T>) {
    // ...
  }
}

#### Object types

You can refer to an Object by any type of resource that is owned by the Object. For convenience, you can convert an address to an Object, or convert an Object between types as long as the resources are available using address_to_object and convert<T> like so:

Example.move

module my_addr::object_playground {
  use aptos_framework::object::{self, Object, ObjectCore};

  struct MyAwesomeStruct has key {}

  fun convert_type(object: Object<ObjectCore>): Object<MyAwesomeStruct> {
    object::convert<MyAwesomeStruct>(object)
  }

  fun address_to_type(object_address: address): Object<MyAwesomeStruct> {
    object::address_to_object<MyAwesomeStruct>(object_address)
  }
}
Objects can be owned by any address, including Objects, Accounts, and Resource accounts. This allows composability between objects and complex relationships between them.

#### Using an Object as type of a field in struct

Objects can help represent complicated types by using them in structs. For example,

Example.move

module my_addr::object_playground {
  use aptos_framework::object::{Self, Object};
  use aptos_framework::fungible_asset::Metadata;
  use aptos_framework::primary_fungible_store;

  struct MyStruct has key {
    fungible_asset_object: Object<Metadata>
  }

  entry fun create_fungible_asset(creator: &signer) {
    let fa_obj_constructor_ref = &object::create_sticky_object(@my_addr);
    let fa_obj_signer = object::generate_signer(fa_obj_constructor_ref);
    let fa_obj_addr = signer::address_of(&fa_obj_signer);
    primary_fungible_store::create_primary_store_enabled_fungible_asset(
        fa_obj_constructor_ref,
        max_supply,
        name,
        symbol,
        decimals,
        icon_uri,
        project_uri
    );
    move_to(creator, MyStruct {
      fungible_asset_object: object::address_to_object(fa_obj_addr)
    });
  }
}

#### Looking up who owns an Object

When writing contracts for Objects, it is often important to verify ownership before modifying the Object. Because an Object can be owned by any address, verifying ownership needs to account for whether the owner is an Account, a Resource Account or another Object like so:

Example.move

module my_addr::object_playground {
  use std::signer;
  use aptos_framework::object::{self, Object};

  // Not authorized!
  const E_NOT_AUTHORIZED: u64 = 1;

  fun check_owner_is_caller<T>(caller: &signer, object: Object<T>) {
    assert!(
      object::is_owner(object, signer::address_of(caller)),
      E_NOT_AUTHORIZED
    );
  }

  fun check_is_owner_of_object<T>(addr: address, object: Object<T>) {
    assert!(object::owner(object) == addr, E_NOT_AUTHORIZED);
  }

  fun check_is_nested_owner_of_object<T, U>(
    caller: &signer,
    outside_object: Object<T>,
    inside_object: Object<U>
  ) {
    // Ownership expected
    // Caller account -> Outside object -> inside object

    // Check outside object owns inside object
    let outside_address = object::object_address(outside_object);
    assert!(object::owns(inside_object, outside_address), E_NOT_AUTHORIZED);

    // Check that the caller owns the outside object
    let caller_address = signer::address_of(caller);
    assert!(object::owns(outside_object, caller_address), E_NOT_AUTHORIZED);

    // Check that the caller owns the inside object (via the outside object)
    // This can skip the first two calls (and even more nested)
    assert!(object::owns(inside_object, caller_address), E_NOT_AUTHORIZED);
  }
}

#### Transfer of ownership

By default, all Objects are transferrable. Some Objects are configured to disable ungated_transfers when they are constructed (see Constructing Objects for more details).

You can transfer an Object like so:

Example.move

module my_addr::object_playground {
  use aptos_framework::object::{self, Object};

  /// Transfer to another address, this can be an object or account
  fun transfer<T>(owner: &signer, object: Object<T>, destination: address) {
    object::transfer(owner, object, destination);
  }

  /// Transfer to another object
  fun transfer_to_object<T, U>(
    owner: &signer,
    object: Object<T>,
    destination: Object<U>
  ) {
    object::transfer_to_object(owner, object, destination);
  }
}
If ungated_transfer is disabled, then all transfers need to use a special permission given by a TransferRef or LinearTransferRef.

#### Events

By default, Objects only have a TransferEvent which triggers whenever the Object is transferred.

Objects can be extended to have additional events.

You can use the following functions to create event handles for Objects:

example.move

module 0x1::object {
  /// Create a guid for the object, typically used for events
  public fun create_guid(object: &signer): guid::GUID {}

  /// Generate a new event handle.
  public fun new_event_handle<T: drop + store>(object: &signer): event::EventHandle<T> {}
}
Generated event handles can be transferred to the Object as long as you have the Object’s SignerRef. For example:

example.move

module 0x42::example {
  use aptos_framework::event;
  use aptos_framework::fungible_asset::FungibleAsset;
  use aptos_framework::object::{Self, Object};

  struct LiquidityPoolEventStore has key {
    create_events: event::EventHandle<CreateLiquidtyPoolEvent>,
  }

  struct CreateLiquidtyPoolEvent {
    token_a: address,
    token_b: address,
    reserves_a: u128,
    reserves_b: u128,
  }

  public entry fun create_liquidity_pool_with_events(
        token_a: Object<FungibleAsset>,
        token_b: Object<FungibleAsset>,
        reserves_a: u128,
        reserves_b: u128
  ) {
    let exchange_signer = &get_exchange_signer();
    let liquidity_pool_constructor_ref = &object::create_object_from_account(
      exchange_signer
    );
    let liquidity_pool_signer = &object::generate_signer(
      liquidity_pool_constructor_ref
    );
    let event_handle = object::new_event_handle<CreateLiquidtyPoolEvent>(
      liquidity_pool_signer
    );

    event::emit_event<CreateLiquidtyPoolEvent>(&mut event_handle, CreateLiquidtyPoolEvent {
      token_a: object::object_address(&token_a),
      token_b: object::object_address(&token_b),
      reserves_a,
      reserves_b,
    });

    move_to(liquidity_pool_signer, LiquidityPool {
      token_a,
      token_b,
      reserves_a,
      reserves_b
    });

    move_to(liquidity_pool_signer, LiquidityPoolEventStore {
      create_events: event_handle
    });
  }
}

#### Modifying Objects after creation

In general, Objects can only be modified with Refs generated during construction. See Creating and Configuring Objects for more details on what Refs are available, how to generate them, and how to use them. This is how you add additional resources to an Object, delete it, and extend it.

One exception to this is “burning” an Object, which sets a flag so Wallet providers know to not display that Object. This can be used to declutter when an Object is not actually deletable.

Example.move

module my_addr::object_playground {
  use aptos_framework::object::{self, Object};

  fun burn_object<T>(owner: &signer, object: Object<T>) {
    object::burn(owner, object);
    assert!(object::is_burnt(object) == true, 1);
  }
}
If you later decide that you want the Object to be displayed again, you can un-”burn” it.

Example.move

module my_addr::object_playground {
  use aptos_framework::object::{self, Object};

  fun unburn_object<T>(owner: &signer, object: object::Object<T>) {
    object::unburn(owner, object);
    assert!(object::is_burnt(object) == false, 1);
  }
}

#### Example contracts

Here are three real-world code snippets which use Objects:

Digital Asset Marketplace Example - https://github.com/aptos-labs/aptos-core/tree/main/aptos-move/move-examples/marketplace
Digital Assets Examples - https://github.com/aptos-labs/aptos-core/tree/main/aptos-move/move-examples/token_objects
Fungible Asset Examples - https://github.com/aptos-labs/aptos-core/tree/main/aptos-move/move-examples/fungible_asset

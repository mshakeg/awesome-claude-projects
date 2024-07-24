# Structuring a BTT Test Suite in Forge/Foundry

This guide outlines how to organize a comprehensive test suite using the Branching Tree Technique (BTT) in a Forge/Foundry project, based on the approach used in the Sablier v2-core codebase.

Note: If a contract name is prefixed with the project name (e.g., SablierV2Lockup), you can omit the prefix in any place where the contract name is being referenced. For example, use "lockup" for folder names or "Lockup" in abstract contract names that incorporate the contract name.

## Terminology

- **Common/Shared Abstract Contract**: Refers to abstract contracts like `BaseContract` in our example or `SablierV2Lockup` in Sablier v2-core. This concept applies to both production code in ./src and test code in ./test.
- **Deployable Contract**: Refers to normal non-abstract solidity contracts that can be deployed and that possibly inherit from common/shared abstract contracts, such as `ContractA`, `ContractB`, and `ContractC` in our example, or `SablierV2LockupDynamic`, `SablierV2LockupLinear`, and `SablierV2LockupTranched` in Sablier v2-core which all inherit the `SablierV2Lockup` abstract contract.
- **Shared Test**: A test contract(typically an abstract contract) containing functionality that's common across multiple contracts or functions. Use for reusable logic, modifiers, or helper functions that are common across multiple contracts or functions.
- **Concrete Test**: A test contract for specific implementations or test scenarios, usually inheriting from shared tests. Use for specific test cases or contract-specific behavior that doesn't need to be shared.

## Example Project Structure

The following structure represents an example project with an abstract contract called `BaseContract` and three inheriting contracts (`ContractA`, `ContractB`, `ContractC`). `BaseContract` implements `function baseFunction1() external` and `function baseFunction2() external` that would be shared by all three inheriting contracts. `ContractA` implements `function functionA() external`, `ContractB` implements `function functionB() external`, and `ContractC` implements `function functionC() external`.

The below structure focuses in more detail on the integration test category; however, the same principles can be applied to the other test categories (i.e., unit, fork, and invariant tests).

```
project/
├── src/
│   ├── base/
│   │   └── BaseContract.sol
│   ├── ContractA.sol
│   ├── ContractB.sol
│   └── ContractC.sol
└── test/
    ├── Base.t.sol
    ├── integration/
    │   ├── Integration.t.sol
    │   ├── shared/
    │   │   ├── base-contract/
    │   │   │   ├── BaseContract.t.sol
    │   │   │   ├── baseFunction1.t.sol
    │   │   │   ├── baseFunction2.t.sol
    │   │   │   ├── functionA.t.sol
    │   │   │   ├── functionB.t.sol
    │   │   │   └── functionC.t.sol
    │   │   ├── contract-a/
    │   │   │   └── ContractA.t.sol
    │   │   ├── contract-b/
    │   │   │   └── ContractB.t.sol
    │   │   └── contract-c/
    │   │       └── ContractC.t.sol
    │   └── concrete/
    │       ├── base-contract/
    │       │   ├── base-function1/
    │       │   │   ├── baseFunction1.tree
    │       │   │   └── baseFunction1.t.sol
    │       │   └── base-function2/
    │       │       ├── baseFunction2.tree
    │       │       └── baseFunction2.t.sol
    │       ├── contract-a/
    │       │   ├── constructor.t.sol
    │       │   ├── ContractA.t.sol
    │       │   └── function-a/
    │       │       ├── functionA.tree
    │       │       └── functionA.t.sol
    │       ├── contract-b/
    │       │   ├── constructor.t.sol
    │       │   ├── ContractB.t.sol
    │       │   └── function-b/
    │       │       ├── functionB.tree
    │       │       └── functionB.t.sol
    │       └── contract-c/
    │           ├── constructor.t.sol
    │           ├── ContractC.t.sol
    │           └── function-c/
    │               ├── functionC.tree
    │               └── functionC.t.sol
    ├── unit/
    ├── fork/
    └── invariant/
```

## Key Components

1. **Base Test Contract**: `Base.t.sol`
   - Inherits from Forge's `Test` contract and other common/shared testing abstract contracts
   - Sets up common test variables and utility functions
   - Deploys necessary contracts for testing common to all test categories (i.e., integration, unit, fork, and invariant tests)
   - Nested directly in the ./test folder
   - Common base to all integration, unit, fork, and invariant tests
   - Generally inherited by the first abstract contract in each test folder (e.g., Integration_Test inherits Base_Test)


2. **Integration Test Base**: `integration/Integration.t.sol`
   - Inherits from `Base.t.sol`
   - Adds integration-specific setup and utility functions
   - Will be inherited at some level by all integration test contracts

   Note: base category test contracts such as `Integration_Test` similarly exist for fork and invariant test categories, all inheriting from `Base_Test`. So `./test/invariant/` would have an `Invariant.t.sol` file that implements an abstract contract called `Invariant_Test` that inherits the `Base_Test` abstract contract and similarly for fork.  Unit tests typically don't follow this structure due to their isolated nature.

3. **Shared Test Contracts**:
   - Located in `test/integration/shared/`
   - Provide common functionality for each contract type
   - Define abstract contracts with functions that are ultimately/eventually inherited by concrete test contracts

4. **Concrete Test Contracts**:
   - Located in `test/integration/concrete/`
   - Implement specific tests for each contract and function
   - Inherit from shared test contracts
   - In order for contracts to be testable by foundry/forge they must be actual deployable contracts not abstract contracts, though they can inherit from other abstract contracts.

5. **Tree Files**:
   - Located alongside their corresponding test files
   - Define all possible execution paths for a function

## Detailed Structure

### 1. Base Test Contract (`Base.t.sol`)

```solidity
abstract contract Base_Test is Test, Assertions, Calculations, Utils {
    // Common variables and setup
    ContractA internal contractA;
    ContractB internal contractB;
    ContractC internal contractC;

    function setUp() public virtual {
        // Deploy contracts, set up test environment
    }

    // Utility functions
}
```

Note: Assertions, Calculations, and Utils are abstract contracts providing custom assertion functions, calculation helpers, and utility functions respectively. For example, Assertions might include a function that accepts 2 custom structs to compare:

```solidity
/// @dev Compares two {Lockup.Amounts} struct entities.
function assertEq(Lockup.Amounts memory a, Lockup.Amounts memory b) internal {
    assertEq(a.deposited, b.deposited, "amounts.deposited");
    assertEq(a.refunded, b.refunded, "amounts.refunded");
    assertEq(a.withdrawn, b.withdrawn, "amounts.withdrawn");
}
```

### 2. Integration Test Base (`integration/Integration.t.sol`)

```solidity
abstract contract Integration_Test is Base_Test {
    function setUp() public virtual override {
        Base_Test.setUp();
        // Additional integration-specific setup
    }

    // Integration-specific utility functions
}
```

### 3. Shared Test Contracts

#### Base Shared Test (`test/integration/shared/base-contract/BaseContract.t.sol`)

```solidity
abstract contract BaseContract_Integration_Shared_Test is Integration_Test {
    IBaseContract internal baseContract;

    function setUp() public virtual override {
        // Make the Sender the default caller in this test suite.
        resetPrank({ msgSender: users.sender });
    }

    // Common test helper functions specified as virtual
    // These functions are meant to be implemented in the concrete {Contract}_Integration_Shared_Test abstract contracts such as ContractA_Integration_Shared_Test, ContractB_Integration_Shared_Test, ContractC_Integration_Shared_Test)
    // This allows each contract to define its own implementation of these helpers based on its specific needs.
    function createDefaultStream() internal virtual returns (uint256 streamId);
    function createDefaultStreamNotCancelable() internal virtual returns (uint256 streamId);
    function createDefaultStreamNotTransferable() internal virtual returns (uint256 streamId);
    // More common test helper virtual functions...
}
```

#### Function-Specific Shared Test for BaseContract (`test/integration/shared/base-contract/baseFunction1.t.sol`)

```solidity
abstract contract BaseFunction1_Integration_Shared_Test is BaseContract_Integration_Shared_Test {
    uint256 internal defaultStreamId;

    function setUp() public virtual override {
        defaultStreamId = createDefaultStream();
        resetPrank({ msgSender: users.recipient });
    }

    // "given" and "when" modifiers corresponding to .tree file conditions
    modifier whenAmountIsZero() { _; }
    modifier whenAmountIsPositive() { _; }
    // More modifiers...
}

// NB: The `BaseFunction2_Integration_Shared_Test` abstract contract would be similarly implemented
```

#### Function-Specific Shared Test for FunctionA (`test/integration/shared/base-contract/functionA.t.sol`)

```solidity
abstract contract FunctionA_Integration_Shared_Test is BaseContract_Integration_Shared_Test {
    uint256 internal streamId;

    function setUp() public virtual override {
        streamId = baseContract.nextStreamId();
    }

    // "given" and "when" modifiers corresponding to .tree file conditions
    modifier whenConditionX() { _; }
    modifier whenConditionY() { _; }
    // More modifiers...
}
```

#### Contract-Specific Shared Test (`test/integration/shared/contract-a/ContractA.t.sol`)

```solidity
abstract contract ContractA_Integration_Shared_Test is BaseContract_Integration_Shared_Test {
    function setUp() public virtual override {
        BaseContract_Integration_Shared_Test.setUp();
        // ContractA-specific setup
    }

    // Implement common test helpers
    function createDefaultStream() internal override returns (uint256 streamId) {
        // ContractA-specific implementation
    }

    function createDefaultStreamNotCancelable() internal override returns (uint256 streamId) {
        // ContractA-specific implementation
    }

    function createDefaultStreamNotTransferable() internal override returns (uint256 streamId) {
        // ContractA-specific implementation
    }

    // More helper implementations...
}
// NB: ContractB_Integration_Shared_Test and ContractC_Integration_Shared_Test can similarly be implemented in their respective file path with their specific contract implementations.
```

### 4. Concrete Test Contracts

#### Function-Specific Concrete Test for BaseContract

`test/integration/concrete/base-contract/base-function1/baseFunction1.t.sol`

```solidity
// This abstract contract provides common test function scenario implementations for baseFunction1 across all contracts
abstract contract BaseFunction1_Integration_Concrete_Test is
    Integration_Test,
    BaseFunction1_Integration_Shared_Test
{
    function setUp() public virtual override(Integration_Test, BaseFunction1_Integration_Shared_Test) {
        // We don't call Integration_Test.setUp() here as it's called in the contract-specific concrete test, which in this example project would be the following named abstract contracts:
        // ContractA_Integration_Concrete_Test
        // ContractB_Integration_Concrete_Test
        // ContractC_Integration_Concrete_Test
        BaseFunction1_Integration_Shared_Test.setUp();
    }

    // Common test functions for baseFunction1
    function test_BaseFunction1_RevertWhen_AmountIsZero() public whenAmountIsZero {
        vm.expectRevert("Amount must be positive");
        baseContract.baseFunction1(0);
    }

    function test_BaseFunction1_UpdateBalance_WhenAmountIsPositive() public whenAmountIsPositive {
        uint256 initialBalance = baseContract.getBalance();
        baseContract.baseFunction1(100);
        assertEq(baseContract.getBalance(), initialBalance + 100);
    }
}
```

#### Contract-Specific Concrete Tests (`test/integration/concrete/contract-a/ContractA.t.sol`)

```solidity
abstract contract ContractA_Integration_Concrete_Test is
    Integration_Test,
    ContractA_Integration_Shared_Test
{
    function setUp() public virtual override(Integration_Test, ContractA_Integration_Shared_Test) {
        Integration_Test.setUp();
        ContractA_Integration_Shared_Test.setUp();

        // Cast the ContractA contract as IBaseContract.
        baseContract = IBaseContract(contractA);
    }
}

// This is a deployable test contract specific to ContractA for baseFunction1
// Given that this is a deployable test contract `forge test` will under the hood deploy this contract and call all of the test_{scenario} functions that it inherited from the BaseFunction1_Integration_Concrete_Test abstract contract
contract BaseFunction1_ContractA_Integration_Concrete_Test is
    ContractA_Integration_Concrete_Test,
    BaseFunction1_Integration_Concrete_Test
{
    function setUp()
        public
        virtual
        override(ContractA_Integration_Concrete_Test, BaseFunction1_Integration_Concrete_Test)
    {
        ContractA_Integration_Concrete_Test.setUp();
        BaseFunction1_Integration_Concrete_Test.setUp();
    }
}

// This is a deployable test contract specific to ContractA for baseFunction2
// Given that this is a deployable test contract `forge test` will under the hood deploy this contract and call all of the test_{scenario} functions that it inherited from the BaseFunction1_Integration_Concrete_Test abstract contract
contract BaseFunction2_ContractA_Integration_Concrete_Test is
    ContractA_Integration_Concrete_Test,
    BaseFunction2_Integration_Concrete_Test
{
    function setUp()
        public
        virtual
        override(ContractA_Integration_Concrete_Test, BaseFunction2_Integration_Concrete_Test)
    {
        ContractA_Integration_Concrete_Test.setUp();
        BaseFunction2_Integration_Concrete_Test.setUp();
    }
}

// Given that baseFunction1 and baseFunction2 are the only BaseContract functions in this example project the above 2 deployable contracts completes this file i.e.:
// test/integration/concrete/contract-a/ContractA.t.sol

// NB: the baseFunction1 and baseFunction2 functions can similarly be tested for ContractB and ContractC since like ContractA they inherit BaseContract.
// These would be implemented in the following 2 files respectively:
// test/integration/concrete/contract-b/ContractB.t.sol
// test/integration/concrete/contract-c/ContractC.t.sol
```

#### Function-Specific Concrete Test for ContractA (`test/integration/concrete/contract-a/function-a/functionA.t.sol`)

```solidity
contract FunctionA_ContractA_Integration_Concrete_Test is
    ContractA_Integration_Concrete_Test,
    FunctionA_Integration_Shared_Test
{
    function setUp() public override(ContractA_Integration_Concrete_Test, FunctionA_Integration_Shared_Test) {
        ContractA_Integration_Concrete_Test.setUp();
        FunctionA_Integration_Shared_Test.setUp();

        // any additional set up that needs to be done can be done next:
        streamId = contractA.nextStreamId();
    }

    function test_FunctionA_WhenConditionX_AndConditionY() public
        whenConditionX
        whenConditionY
    {
        // Test implementation
    }

    // More test scenario functions...
}
```

## Tree Files

Tree files define the structure of possible execution paths for a function. Each branch typically corresponds to a modifier in the test implementation.

### Example Tree File for BaseContract baseFunction1 (`test/integration/concrete/base-contract/base-function1/baseFunction1.tree`)

```
baseFunction1.t.sol
├── when amount is zero
│   └── it should revert
└── when amount is positive
    └── it should update balance
```

This tree file structure directly corresponds to the test functions in the concrete test contract:

```solidity
function test_BaseFunction1_RevertWhen_AmountIsZero() public whenAmountIsZero { ... }
function test_BaseFunction1_UpdateBalance_WhenAmountIsPositive() public whenAmountIsPositive { ... }
```

#### Example Tree File for ContractA functionA (`test/integration/concrete/contract-a/function-a/functionA.tree`)

```
functionA.t.sol
├── when condition X
│   ├── when condition Y
│   │   └── it should do P
│   └── when condition Z
│       └── it should do Q
└── when condition W
    └── it should do R
```

Note: Typically, each condition in the .tree file maps to a single modifier in the test contracts.

## Brief Example for ContractB

We've already detailed how the test suite would be organized for ContractA, so we won't be as detailed in showing how the structure would be repeated for ContractB as it is very similar:

```solidity
// test/integration/shared/contract-b/ContractB.t.sol
abstract contract ContractB_Integration_Shared_Test is BaseContract_Integration_Shared_Test {
    function setUp() public virtual override {
        BaseContract_Integration_Shared_Test.setUp();
        // ContractB-specific setup
    }

    // Implement common test helpers
    function createDefaultStream() internal override returns (uint256 streamId) {
        // ContractB-specific implementation
    }

    // More helper implementations...
}

// test/integration/concrete/contract-b/ContractB.t.sol
abstract contract ContractB_Integration_Concrete_Test is
    Integration_Test,
    ContractB_Integration_Shared_Test
{
    function setUp() public virtual override(Integration_Test, ContractB_Integration_Shared_Test) {
        Integration_Test.setUp();
        ContractB_Integration_Shared_Test.setUp();

        baseContract = IBaseContract(contractB);
    }
}

contract BaseFunction1_ContractB_Integration_Concrete_Test is
    ContractB_Integration_Concrete_Test,
    BaseFunction1_Integration_Concrete_Test
{
    function setUp()
        public
        virtual
        override(ContractB_Integration_Concrete_Test, BaseFunction1_Integration_Concrete_Test)
    {
        ContractB_Integration_Concrete_Test.setUp();
        BaseFunction1_Integration_Concrete_Test.setUp();
    }
}

// BaseFunction2_ContractB_Integration_Concrete_Test would then be implemented similarly to BaseFunction1_ContractB_Integration_Concrete_Test in the same file(i.e. `test/integration/concrete/contract-b/ContractB.t.sol`) and would then complete this file given that in this example project baseFunction1 and baseFunction2 are the only external/public BaseContract functions
```

## Best Practices

1. **Modular Design**: Use inheritance to maximize code reuse and maintain a clear structure.
2. **Consistent Naming**: Follow a consistent naming convention for test contracts and functions.
3. **Mapping to Tree Files**: Ensure each test function corresponds to a branch in the .tree file.
4. **Use of Modifiers**: Utilize modifiers to set up test conditions matching the .tree structure.
5. **Separation of Concerns**: Keep shared functionality in shared tests and specific implementations in concrete tests.
6. **Comprehensive Coverage**: Ensure all branches in .tree files are covered by test functions.
7. **Clear Documentation**: Comment test functions to explain what scenario they're testing.
8. **Explicit setUp() Calls**: When overriding setUp(), explicitly call parent setUp() functions to ensure all necessary setup is performed.
9. **Test Function Naming Convention**: Use the format `test_[FunctionName]_[Condition1]_[Condition2]...` For revert cases, use `test_RevertWhen_[Condition]`.

## Running Tests

To run all tests:

```bash
forge test
```

To run specific test contracts or functions:

```bash
forge test --match-contract BaseFunction1_ContractA_Integration_Concrete_Test
forge test --match-test test_BaseFunction1_RevertWhen_AmountIsZero
```

## Integration with Other Test Types

While this structure focuses on integration tests, it can be adapted for other test types:

- **Unit Tests**: Focus on testing individual functions in isolation. They may not require the full inheritance structure but can still use shared helper functions.
- **Fork Tests**: Use a similar structure but with setup that forks from a specific network state. Include additional logic in the setUp() functions to handle forking.
- **Invariant Tests**: Focus on properties that should always hold true across various state changes. They may require a different structure but can still utilize shared helper functions and modifiers.

## Handling Complex Inheritance

In scenarios with multiple levels of inheritance:

1. Use virtual functions for overridable behavior.
2. Be explicit about which parent functions you're calling in overrides.
3. Consider using interfaces to define expected behavior.

## Common Pitfalls and Challenges

1. **Overcomplication**: Don't create shared contracts for every small piece of functionality. Balance reusability with simplicity.
2. **Inheritance Conflicts**: Be aware of the order of inheritance and how it affects function overriding.
3. **Maintainability**: As the project grows, maintain clear documentation and consider refactoring when the structure becomes too complex.
4. **Test Isolation**: Ensure that tests are isolated and don't depend on the state from other tests.

## Conclusion

This structure allows for a highly organized and maintainable test suite that fully implements the Branching Tree Technique. It provides clear separation of concerns, maximizes code reuse, and ensures comprehensive test coverage across all possible execution paths.

The approach demonstrated here, inspired by the Sablier v2-core codebase, offers a robust framework for testing complex smart contract systems. By using a combination of shared abstract contracts and concrete test implementations, it allows for thorough testing of both shared functionalities (like those in BaseContract) and specific functionalities of individual contracts (like ContractA, ContractB, and ContractC).

This structure is particularly beneficial for projects with multiple related contracts that share common functionalities. It allows for efficient testing of these shared functions across all implementing contracts while still providing the flexibility to test contract-specific features.

Remember that while this structure provides a comprehensive framework, it should be adapted as necessary to fit the specific needs and complexities of your particular project. The key is to maintain a clear, organized, and thorough testing strategy that covers all possible execution paths of your smart contracts.
# Implementing BTT Tests

## Test Contract Structure

1. Create a base test abstract contract inheriting from your testing framework (e.g., forge-std/Test.sol)
2. Implement a setUp() function to deploy the contract under test
3. Create separate test contracts for each function, inheriting from the base test abstract contract
4. Use multiple levels of inheritance to handle complex .tree structures

## Naming Conventions

Use the following naming conventions for test functions:

- General format: test_[Condition(s)]_[ExpectedOutcome]

- For revert cases:
    - test_RevertGiven_[StateCondition]: Used when a state condition causes a revert
    - test_RevertWhen_[ParameterCondition]: Used when a parameter condition causes a revert

- For successful cases:
    - test_[FunctionName]_[Condition(s)]

- For complex scenarios:
    - Combine multiple conditions using underscores, e.g., test_Withdraw_CallerRecipient2

- For testing specific behaviors or outcomes:
    - Use descriptive names that indicate the behavior being tested, e.g., test_Withdraw_EndTimeNotInTheFuture

Examples from the Sablier v2-core:

```solidity
function test_RevertWhen_DelegateCalled() external { ... }
function test_RevertGiven_Null() external whenNotDelegateCalled { ... }
function test_RevertGiven_StreamDepleted() external whenNotDelegateCalled givenNotNull { ... }
function test_RevertWhen_ToZeroAddress() external whenNotDelegateCalled givenNotNull givenStreamNotDepleted { ... }
function test_Withdraw_CallerApprovedOperator() external { ... }
function test_Withdraw_EndTimeNotInTheFuture() external { ... }
function test_Withdraw_StreamHasBeenCanceled() external { ... }
function test_Withdraw() external { ... }
```

These naming conventions help in clearly identifying the purpose of each test, the conditions being tested, and the expected outcomes. They also align closely with the structure of the .tree files, making it easier to map between the test implementation and the test structure documentation.

## Using Modifiers

Modifiers play a crucial role in BTT implementation. In complex scenarios, it's common to use empty modifiers extensively. These empty modifiers directly correspond to the branches in the .tree file, even when they don't contain specific logic. This approach ensures a clear mapping between the .tree structure and the test implementation.

### Example of Empty Modifiers

If the same modifiers are going to be used to test other contracts with the same or similar external functions you can group them together in a shared abstract contract similar to CreateWithDurations_Integration_Shared_Test shown below that is inherited by those various test contracts for testing the same or similar external function

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.8.22 <0.9.0;

import { Lockup_Integration_Shared_Test } from "./Lockup.t.sol";

abstract contract CreateWithDurations_Integration_Shared_Test is Lockup_Integration_Shared_Test {
    uint256 internal streamId;

    function setUp() public virtual override {
        streamId = lockup.nextStreamId();
    }

    modifier whenCliffDurationCalculationDoesNotOverflow() {
        _;
    }

    modifier whenDurationsNotZero() {
        _;
    }

    modifier whenNotDelegateCalled() {
        _;
    }

    modifier whenSegmentCountNotTooHigh() {
        _;
    }

    modifier whenTimestampsCalculationsDoNotOverflow() {
        _;
    }

    modifier whenTotalDurationCalculationDoesNotOverflow() {
        _;
    }

    modifier whenTrancheCountNotTooHigh() {
        _;
    }
}
```

### Using Modifiers in Test Functions

Apply modifiers to relevant test functions, stacking multiple modifiers to reflect the .tree structure as shown below in `CreateWithDurations_LockupDynamic_Integration_Concrete_Test` test contract for testing the `SablierV2LockupDynamic` contract's `createWithDurations` external function.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.8.22 <0.9.0;

import { ud2x18 } from "@prb/math/src/UD2x18.sol";

import { ISablierV2LockupDynamic } from "src/interfaces/ISablierV2LockupDynamic.sol";
import { Errors } from "src/libraries/Errors.sol";
import { Lockup, LockupDynamic } from "src/types/DataTypes.sol";

import { CreateWithDurations_Integration_Shared_Test } from "../../../shared/lockup/createWithDurations.t.sol";
import { LockupDynamic_Integration_Concrete_Test } from "../LockupDynamic.t.sol";

contract CreateWithDurations_LockupDynamic_Integration_Concrete_Test is
    LockupDynamic_Integration_Concrete_Test,
    CreateWithDurations_Integration_Shared_Test
{
    function setUp()
        public
        virtual
        override(LockupDynamic_Integration_Concrete_Test, CreateWithDurations_Integration_Shared_Test)
    {
        LockupDynamic_Integration_Concrete_Test.setUp();
        CreateWithDurations_Integration_Shared_Test.setUp();
        streamId = lockupDynamic.nextStreamId();
    }

    /// @dev it should revert.
    function test_RevertWhen_DelegateCalled() external {
        // test implementation...
    }

    /// @dev it should revert.
    function test_RevertWhen_SegmentCountTooHigh() external whenNotDelegateCalled {
        // test implementation...
    }

    function test_RevertWhen_DurationsZero() external whenNotDelegateCalled whenSegmentCountNotTooHigh {
        // test implementation...
    }

    function test_RevertWhen_TimestampsCalculationsOverflows_StartTimeNotLessThanFirstSegmentTimestamp()
        external
        whenNotDelegateCalled
        whenSegmentCountNotTooHigh
        whenDurationsNotZero
    {
        // test implementation...
    }

    function test_RevertWhen_TimestampsCalculationsOverflows_SegmentTimestampsNotOrdered()
        external
        whenNotDelegateCalled
        whenSegmentCountNotTooHigh
        whenDurationsNotZero
    {
        // test implementation...
    }

    function test_CreateWithDurations()
        external
        whenNotDelegateCalled
        whenSegmentCountNotTooHigh
        whenDurationsNotZero
        whenTimestampsCalculationsDoNotOverflow
    {
        // test implementation...
    }
}
```

This approach allows for a clear, one-to-one mapping between the .tree file structure and the test implementation, enhancing readability and maintainability.

## Best Practices

1. Ensure each test function corresponds to a complete path in the .tree file
2. Use assertions to verify expected outcomes for each branch
3. Test both happy paths and error cases thoroughly
4. Leverage VM operations (like vm.expectRevert, vm.warp) for testing complex conditions
5. Use helper functions to set up complex test scenarios
6. Maintain consistency between .tree files and test implementations
7. Regularly review and update tests as the contract evolves
8. Use empty modifiers to represent all conditions from the .tree file, even if they don't require specific setup
9. Consider grouping related empty modifiers in a shared abstract contract for reuse across multiple test files
10. Implement thorough setUp() functions to ensure correct initial state for each test

Remember to balance thoroughness with maintainability in your test implementations. The use of modifiers including empty modifiers allows for a clear representation of all conditions from the .tree file in your test code, improving readability and alignment with the BTT structure.
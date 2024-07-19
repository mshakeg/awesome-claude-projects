# Implementing BTT Tests

## Test Contract Structure

1. Create a base test abstract contract inheriting from your testing framework (e.g., forge-std/Test.sol)
2. Implement a setUp() function to deploy the contract under test
3. Create separate test contracts for each function, inheriting from the base test abstract contract

## Naming Conventions

Based on the patterns observed in the Sablier v2-core codebase, the following naming conventions are used for test functions:

- General format: test_[Condition(s)]_[ExpectedOutcome]

- For revert cases:
    - test_RevertGiven_[StateCondition]: Used when a state condition causes a revert
    - test_RevertWhen_[ParameterCondition]: Used when a parameter condition causes a revert

- For successful cases:
    - test_[FunctionName]: Used for straightforward, happy-path scenarios
    - test_[FunctionName]_[Condition(s)]: Used when testing specific conditions in successful scenarios

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

```solidity
abstract contract Withdraw_Integration_Shared_Test is Lockup_Integration_Shared_Test {
    modifier whenNotDelegateCalled() {
        _;
    }

    modifier givenNotNull() {
        _;
    }

    modifier givenStreamNotDepleted() {
        _;
    }

    modifier whenToNonZeroAddress() {
        _;
    }

    // ... many more empty modifiers ...
}
```

### Using Modifiers in Test Functions

Apply modifiers to relevant test functions, stacking multiple modifiers to reflect the .tree structure:

```solidity
function test_Withdraw()
    external
    whenNotDelegateCalled
    givenNotNull
    givenStreamNotDepleted
    whenToNonZeroAddress
    whenWithdrawAmountNotZero
    whenNoOverdraw
    whenWithdrawalAddressIsRecipient
    whenCallerSender
    givenEndTimeInTheFuture
    whenStreamHasNotBeenCanceled
    givenRecipientAllowedToHook
    whenRecipientNotReverting
    whenRecipientReturnsSelector
    whenRecipientNotReentrant
{
    // Test implementation
}
```

This approach allows for a clear, one-to-one mapping between the .tree file structure and the test implementation, enhancing readability and maintainability.

## Best Practices

1. Ensure each test function corresponds to a branch in the .tree file
2. Use assertions to verify expected outcomes
3. Test both happy paths and error cases
4. Leverage VM operations (like vm.expectRevert) for testing error conditions
5. Use helper functions to set up complex test scenarios
6. Maintain consistency between .tree files and test implementations
7. Regularly review and update tests as the contract evolves
8. Use empty modifiers to represent all conditions from the .tree file, even if they don't require specific setup
9. Consider grouping related empty modifiers in a shared abstract contract for reuse across multiple test files

## Example Implementation

```solidity
abstract contract Withdraw_Integration_Shared_Test is Lockup_Integration_Shared_Test {
    uint256 internal defaultStreamId;

    function setUp() public virtual override {
        defaultStreamId = createDefaultStream();
        resetPrank({ msgSender: users.recipient });
    }

    // Active modifier
    modifier givenEndTimeInTheFuture() {
        vm.warp({ newTimestamp: defaults.WARP_26_PERCENT() });
        _;
    }

    // Empty modifiers
    modifier givenNotNull() {
        _;
    }

    modifier whenNoOverdraw() {
        _;
    }

    // ... other modifiers ...
}

contract Withdraw_Integration_Concrete_Test is Withdraw_Integration_Shared_Test {
    function test_Withdraw()
        external
        whenNotDelegateCalled
        givenNotNull
        givenStreamNotDepleted
        whenToNonZeroAddress
        whenWithdrawAmountNotZero
        whenNoOverdraw
        whenWithdrawalAddressIsRecipient
        whenCallerSender
        givenEndTimeInTheFuture
        whenStreamHasNotBeenCanceled
        givenRecipientAllowedToHook
        whenRecipientNotReverting
        whenRecipientReturnsSelector
        whenRecipientNotReentrant
    {
        // Test implementation
    }
}
```

Remember to balance thoroughness with maintainability in your test implementations. The use of empty modifiers allows for a clear representation of all conditions from the .tree file in your test code, improving readability and alignment with the BTT structure.
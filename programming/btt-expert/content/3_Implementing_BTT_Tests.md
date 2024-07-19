# Implementing BTT Tests

## Test Contract Structure

1. Create a base test abstract contract inheriting from your testing framework (e.g., forge-std/Test.sol)
2. Implement a setUp() function to deploy the contract under test
3. Create separate test contracts for each function, inheriting from the base test abstract contract

## Naming Conventions

- Test functions: test_[Condition]_[ExpectedOutcome]
- For revert cases: test_RevertWhen_[Condition]
- For successful cases: test_[FunctionName]

## Using Modifiers

- Create modifiers for reusable "given" or "when" conditions
- Apply modifiers to relevant test functions
- Stack multiple modifiers when necessary
- Use modifiers declaratively to enhance test readability

## Example Implementation

```solidity
contract CreateWithDurations_LockupDynamic_Integration_Concrete_Test is
    LockupDynamic_Integration_Concrete_Test,
    CreateWithDurations_Integration_Shared_Test
{
    function test_RevertWhen_DelegateCalled() external {
        // Test implementation
    }

    function test_RevertWhen_SegmentCountTooHigh() external whenNotDelegateCalled {
        // Test implementation
    }

    function test_CreateWithDurations()
        external
        whenNotDelegateCalled
        whenSegmentCountNotTooHigh
        whenDurationsNotZero
        whenTimestampsCalculationsDoNotOverflow
    {
        // Test implementation
    }
}
```

## Best Practices

1. Ensure each test function corresponds to a branch in the .tree file
2. Use assertions to verify expected outcomes
3. Test both happy paths and error cases
4. Leverage VM operations (like vm.expectRevert) for testing error conditions
5. Use helper functions to set up complex test scenarios
6. Maintain consistency between .tree files and test implementations
7. Regularly review and update tests as the contract evolves

Remember to balance thoroughness with maintainability in your test implementations.
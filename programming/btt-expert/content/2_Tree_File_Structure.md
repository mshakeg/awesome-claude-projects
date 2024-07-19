# .tree File Structure and Best Practices

## Basic Structure

The .tree file structure follows a hierarchical pattern that represents all possible execution paths of a function, distinguishing between state conditions and parameter conditions:

```
functionName.t.sol
├── given [state condition]
│   ├── when [parameter condition]
│   │   └── it should [expected outcome]
│   └── when [different parameter condition]
│       └── it should [different expected outcome]
└── given [different state condition]
    └── when [parameter condition]
        └── it should [expected outcome]
```

## Complex Example

In real-world applications, .tree files can be much more complex. Here's an example based on a withdraw function taken from the sablier v2-core codebase:

```
withdraw.t.sol
├── when delegate called
│   └── it should revert
└── when not delegate called
    ├── given the ID references a null stream
    │   └── it should revert
    └── given the ID does not reference a null stream
        ├── given the stream's status is "DEPLETED"
        │   └── it should revert
        └── given the stream's status is not "DEPLETED"
            ├── when the provided address is zero
            │   └── it should revert
            └── when the provided address is not zero
                ├── when the withdraw amount is zero
                │   └── it should revert
                └── when the withdraw amount is not zero
                    ├── when the withdraw amount overdraws
                    │   └── it should revert
                    └── when the withdraw amount does not overdraw
                        ├── when the withdrawal address is not the stream recipient
                        │   ├── when the caller is unknown
                        │   │   └── it should revert
                        │   ├── when the caller is the sender
                        │   │   └── it should revert
                        │   ├── when the caller is a former recipient
                        │   │   └── it should revert
                        │   ├── when the caller is an approved third party
                        │   │   ├── it should make the withdrawal
                        │   │   └── it should update the withdrawn amount
                        │   └── when the caller is the recipient
                        │       ├── it should make the withdrawal
                        │       ├── it should update the withdrawn amount
                        │       ├── it should emit a {MetadataUpdate} event
                        │       └── it should emit a {WithdrawFromLockupStream} event
                        └── when the withdrawal address is the stream recipient
                            ├── when the caller is unknown
                            │   ├── it should make the withdrawal
                            │   └── it should update the withdrawn amount
                            ├── when the caller is the recipient
                            │   ├── it should make the withdrawal
                            │   └── it should update the withdrawn amount
                            └── when the caller is the sender
                                ├── given the end time is not in the future
                                │   ├── it should make the withdrawal
                                │   ├── it should mark the stream as depleted
                                │   └── it should make the stream not cancelable
                                └── given the end time is in the future
                                    ├── given the stream has been canceled
                                    │   ├── it should make the withdrawal
                                    │   ├── it should mark the stream as depleted
                                    │   ├── it should update the withdrawn amount
                                    │   ├── it should make Sablier run the recipient hook
                                    │   ├── it should emit a {MetadataUpdate} event
                                    │   └── it should emit a {WithdrawFromLockupStream} event
                                    └── given the stream has not been canceled
                                        ├── given the recipient is not allowed to hook
                                        │   ├── it should make the withdrawal
                                        │   ├── it should update the withdrawn amount
                                        │   └── it should not make Sablier run the recipient hook
                                        └── given the recipient is allowed to hook
                                            ├── when the recipient reverts
                                            │   └── it should revert the entire transaction
                                            └── when the recipient does not revert
                                                ├── when the recipient hook does not return a valid selector
                                                │   └── it should revert
                                                └── when the recipient hook returns a valid selector
                                                    ├── when there is reentrancy
                                                    │   ├── it should make multiple withdrawals
                                                    │   ├── it should update the withdrawn amounts
                                                    │   └── it should make Sablier run the recipient hook
                                                    └── when there is no reentrancy
                                                        ├── it should make the withdrawal
                                                        ├── it should update the withdrawn amount
                                                        ├── it should make Sablier run the recipient hook
                                                        ├── it should emit a {MetadataUpdate} event
                                                        └── it should emit a {WithdrawFromLockupStream} event
```

## Best Practices

1. Start with the function name as the root, typically ending with `.t.sol`, e.g. `withdraw.t.sol`
2. Use "given" nodes to represent contract states, including complex derived/computed/composed states.
3. Use "when" nodes to represent function parameters or input conditions.
4. Use "when" for transaction-specific inputs like msg.sender, msg.value, and msg.data, as these are considered input parameters, not state conditions.
5. End branches with "it should" nodes describing expected outcomes.
6. Keep .tree files focused on external behavior.
7. Use clear, descriptive language for each node.
8. Include both happy path and error case scenarios.
9. Consider edge cases and boundary conditions.

## Tips for Complex Functions

- Break down complex conditions into multiple levels of "when" statements.
- Group similar conditions together for better organization.
- Use comments to clarify complex scenarios if necessary.

## Granularity Guidelines

1. Create new branches for distinct scenarios that significantly change the function's execution path.
2. Separate independent conditions into different branches.
3. Group closely related, complex checks within a single branch if they always occur together.
4. Consider making reusable conditions separate branches.
5. Balance granularity with readability.
6. Ensure each branch represents an independent test case where possible.
7. Give each distinct error case its own branch.
8. Reflect the logical flow of the contract function in the .tree structure.
9. Create separate branches for important edge cases.
10. Each branch should lead to a meaningful and testable outcome.

## Mapping to Test Implementation

Each branch in the .tree file typically corresponds to a modifier in the test implementation. This is true even for branches that don't require specific setup logic - these are often represented by empty modifiers. This approach ensures a clear, one-to-one mapping between the .tree structure and the test code, enhancing readability and maintainability.

Remember to keep your .tree files up-to-date as your contract evolves, and ensure that all branches are covered in your test implementation.
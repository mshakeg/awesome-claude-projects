# .tree File Structure and Best Practices

## Basic Structure

The .tree file structure follows a hierarchical pattern that represents all possible execution paths of a function, distinguishing between state conditions and parameter conditions:

```
functionName.t.sol
├── when [most general input condition]
│   ├── given [most general state condition]
│   │   ├── when [more specific input condition]
│   │   │   ├── given [more specific state condition]
│   │   │   │   └── it should [expected outcome]
│   │   │   └── given [alternative specific state condition]
│   │   │       └── it should [different expected outcome]
│   │   └── when [alternative specific input condition]
│   │       └── it should [expected outcome]
│   └── given [alternative general state condition]
│       └── it should [expected outcome]
└── when [alternative general input condition]
    └── it should [expected outcome]
```

## Complex Example

In real-world applications, .tree files can be much more complex. Here's an expanded example based on a `withdraw` function for a streaming payment contract taken from `@sablier/v2-core`:

## Example: Comprehensive .tree File

Here's an expanded example based on a complex withdraw function:

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
                  │      ├── it should make the withdrawal
                  │      ├── it should update the withdrawn amount
                  │      ├── it should emit a {MetadataUpdate} event
                  │      └── it should emit a {WithdrawFromLockupStream} event
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

This example demonstrates the level of detail and granularity expected in a comprehensive .tree file for a complex smart contract function.


## Best Practices

1. Start with the most general conditions (e.g., delegate call checks) and progressively add more specific conditions.
2. Use "when" for input parameters and "given" for state conditions consistently.
3. Consider all possible types of callers and their permissions.
4. Include time-based conditions where relevant.
5. Consider all possible states of the contract and related entities (e.g., stream states).
6. Go into detail about complex operations like hook executions, including potential reentrancy scenarios.
7. Use clear, descriptive language for each node.
8. Include both happy path and error case scenarios.
9. Consider edge cases and boundary conditions exhaustively.
10. Ensure each distinct error case has its own branch.
11. Explicitly enumerate all possible contract states and their effects on function behavior.
12. Create a comprehensive taxonomy of potential callers and their actions.
13. Represent state transitions within the tree structure.
14. Show event emissions for different scenarios.
15. Include cross-function dependencies where relevant.
16. Represent the effects of modifiers and access control mechanisms.
17. Consider different numeric scenarios for inputs and state variables.

## Tips for Complex Functions

- Break down complex conditions into multiple levels of "when" and "given" statements.
- Group similar conditions together for better organization.
- Use comments to clarify complex scenarios if necessary.
- Consider creating separate sub-trees for highly complex operations and referencing them in the main tree.

## Granularity Guidelines

1. Create new branches for distinct scenarios that significantly change the function's execution path.
2. Separate independent conditions into different branches.
3. Group closely related, complex checks within a single branch if they always occur together.
4. Consider making reusable conditions separate branches.
5. Balance granularity with readability.
6. Ensure each branch represents an independent test case where possible.
7. Reflect the logical flow of the contract function in the .tree structure.
8. Create separate branches for important edge cases.
9. Each branch should lead to a meaningful and testable outcome.
10. Don't hesitate to create deeply nested structures if they accurately represent the function's logic.
11. Break down scenarios into the most granular cases possible, even if they seem redundant at first.
12. Create separate branches for distinct caller types, contract states, and input conditions.

## Handling Complex Scenarios

1. Reentrancy: Create separate branches for reentrancy scenarios, considering multiple withdrawal attempts.
2. Time-dependent logic: Use "given" statements to represent different time states (e.g., "given the end time is in the future").
3. External calls: Consider all possible outcomes of external calls, including failures and unexpected return values.
4. State changes: Reflect how different actions might change the contract's state and affect subsequent operations.
5. Event emissions: Explicitly show which events are emitted in different scenarios.
6. Error conditions: Include all possible error states and specific error messages or codes.
7. Numeric edge cases: Represent different numeric scenarios (e.g., zero, non-zero, maximum value) for inputs and state variables.
8. Gas considerations: Include branches for out-of-gas conditions or gas-dependent behavior when relevant.

## Mapping to Test Implementation

Each branch in the .tree file typically corresponds to a modifier in the test implementation. This is true even for branches that don't require specific setup logic - these are often represented by empty modifiers. This approach ensures a clear, one-to-one mapping between the .tree structure and the test code, enhancing readability and maintainability.

Remember to keep your .tree files up-to-date as your contract evolves, and ensure that all branches are covered in your test implementation. Regularly review and validate your .tree files to maintain their accuracy and comprehensiveness.
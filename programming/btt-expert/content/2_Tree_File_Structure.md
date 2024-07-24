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

In real-world applications, .tree files can be much more complex. Here's an example based on a `createWithDurations` external function for the `SablierV2LockupDynamic` contract taken from the sablier v2-core codebase:

```
createWithDurations.t.sol
├── when delegate called
│  └── it should revert
└── when not delegate called
   ├── when the segment count is too high
   │  └── it should revert
   └── when the segment count is not too high
       ├── when at least one of the durations at index one or greater is zero
       │  └── it should revert
       └── when none of the durations is zero
          ├── when the segment timestamp calculations overflow uint256
          │  ├── when the start time is not less than the first segment timestamp
          │  │  └── it should revert
          │  └── when the segment timestamps are not ordered
          │     └── it should revert
          └── when the segment timestamp calculations do not overflow uint256
             ├── it should create the stream
             ├── it should bump the next stream ID
             ├── it should mint the NFT
             ├── it should emit a {MetadataUpdate} event
             ├── it should perform the ERC-20 transfers
             └── it should emit a {CreateLockupDynamicStream} event
```

## Best Practices

1. Start with the function name as the root, typically ending with `.t.sol`, e.g. `createWithDurations.t.sol`
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
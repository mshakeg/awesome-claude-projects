# NatSpec Guidelines for Solidity

NatSpec (Natural Specification) is a documentation format for Solidity that uses special comments to generate human-readable and machine-readable documentation.

## Key Components

1. @title: Describes the contract's purpose
2. @author: Lists the contract's authors
3. @notice: Explains what the contract or function does (for users)
4. @dev: Provides additional details (for developers)
5. @param: Describes function parameters
6. @return: Describes function return values
7. @inheritdoc: Copies documentation from the base function

## Example

```solidity
/// @title Token Contract
/// @author Jane Doe
/// @notice This contract manages a simple token system
/// @dev Implements ERC20 standard
contract Token {
    /// @notice Transfer tokens to a specified address
    /// @dev Emits a Transfer event
    /// @param _to The address to transfer to
    /// @param _value The amount to be transferred
    /// @return success True if the transfer was successful
    function transfer(address _to, uint256 _value) public returns (bool success) {
        // Function implementation
    }
}
```

## Inline Comments

While NatSpec is primarily used for function and contract-level documentation, inline comments are crucial for explaining complex logic within functions. When adding inline comments:

- Use // for single-line comments
- Explain the "why" behind complex operations
- Break down multi-step processes
- Highlight any non-obvious implications of the code

Example:

```solidity
function complexOperation(uint256 x) public {
    // Perform initial validation
    require(x > 0, "Input must be positive");

    // Apply a complex calculation
    // This formula accounts for diminishing returns as x increases
    uint256 result = (x * x) / (x + 10);

    // Ensure result doesn't exceed maximum allowed value
    if (result > MAX_VALUE) {
        result = MAX_VALUE;
    }

    // Update state and emit event
    totalResult += result;
    emit ResultCalculated(msg.sender, result);
}
```

## Special Comment Types

### @dev

The @dev tag is used to provide additional details for developers. Use it for:

1. Explaining complex algorithms or non-obvious implementation details
2. Highlighting potential pitfalls or areas that require careful attention
3. Providing context about design decisions
4. Mentioning pending improvements or known limitations

### NOTE

"NOTE" is used for inline comments or in function-level documentation to draw attention to important information. Use it for:

1. Highlighting important considerations that don't fit into other NatSpec tags
2. Providing additional context crucial for understanding the code
3. Warning about potential edge cases or non-obvious behavior

### NB (Nota Bene)

"NB" is used to emphasize critically important information. Use it for:

1. Pointing out crucial information that must not be overlooked
2. Highlighting potential security risks or critical assumptions
3. Noting exceptional cases that deviate from the general pattern

## Documenting Different Function Types

### Internal and Private Functions

Use full NatSpec comments for internal and private functions, explaining purpose, parameters, return values, and implementation details.

### Public and External Functions with Interfaces

1. Place main NatSpec documentation in the interface
2. Use @inheritdoc in the implementing contract
3. Add additional comments in the implementing contract for specific details or caveats
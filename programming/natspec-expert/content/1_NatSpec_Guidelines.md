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
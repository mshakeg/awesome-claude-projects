# Advanced Documentation Techniques for Solidity

## Code Examples in Documentation

For complex functions or usage patterns, include code examples in the documentation:

```solidity
/// @notice Swaps tokens using the best available route
/// @dev This function will revert if slippage exceeds the specified amount
/// @param tokenIn The address of the input token
/// @param tokenOut The address of the output token
/// @param amountIn The amount of input tokens to swap
/// @param minAmountOut The minimum amount of output tokens to receive
/// @return amountOut The amount of output tokens received
/// @return path The path used for the swap
/// @example
/// ```
/// IERC20(DAI).approve(address(this), 1000e18);
/// (uint256 amountOut, address[] memory path) = router.swapExactTokensForTokens(
///     DAI,
///     USDC,
///     1000e18,
///     990e6
/// );
/// ```
function swapExactTokensForTokens(
    address tokenIn,
    address tokenOut,
    uint256 amountIn,
    uint256 minAmountOut
) external returns (uint256 amountOut, address[] memory path) {
    // Implementation
}
```

## TODO and FIXME Comments

Use TODO and FIXME comments to mark areas that need future attention, but use them sparingly and try to resolve them quickly:

```solidity
function complexCalculation(uint256 x, uint256 y) internal pure returns (uint256) {
    // TODO: Optimize this calculation for gas efficiency
    return (x * y) / (x + y) + (x ^ y);
}

// FIXME: This function is vulnerable to front-running attacks
function revealRandomNumber(uint256 guess) external {
    // Implementation
}
```

## Documenting Design Patterns

When using common design patterns, document them explicitly:

```solidity
/// @dev This contract uses the Ownable pattern for access control
/// @custom:security Uses OpenZeppelin's Ownable for secure ownership management
contract MyContract is Ownable {
    // Contract implementation
}
```

## Cross-referencing

Use cross-references to link related functions or concepts:

```solidity
/// @notice Allows a user to stake tokens
/// @dev This function is closely related to the `unstake` function
/// See {unstake} for the reversal of this operation
function stake(uint256 amount) external {
    // Implementation
}

/// @notice Allows a user to unstake tokens
/// @dev This function is closely related to the `stake` function
/// See {stake} for the initial staking operation
function unstake(uint256 amount) external {
    // Implementation
}
```

## Documenting State Transitions

For complex state machines or multi-step processes, document the state transitions:

```solidity
/// @notice Initiates the proposal process
/// @dev State transition: None -> Pending
function createProposal() external {
    // Implementation
}

/// @notice Executes an approved proposal
/// @dev State transition: Approved -> Executed
function executeProposal(uint256 proposalId) external {
    // Implementation
}
```

## Visual Aids

Consider using ASCII art or markdown tables in comments for complex data structures or algorithms:

```solidity
/// @dev Tree structure for merkle proof:
///
///         ROOT
///        /    \
///      A      B
///     / \    / \
///    C   D  E   F
///
/// Where A = hash(C || D), B = hash(E || F), ROOT = hash(A || B)
function verifyMerkleProof(bytes32[] memory proof, bytes32 leaf) public view returns (bool) {
    // Implementation
}
```

These advanced techniques will help create more comprehensive and user-friendly documentation for Solidity smart contracts.
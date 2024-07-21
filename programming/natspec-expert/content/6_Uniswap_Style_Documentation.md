# Uniswap-Style Documentation for Solidity Projects

When documenting complex Solidity projects, following the conventions used by established projects like Uniswap V3 can greatly enhance code readability and maintainability. This guide outlines the key aspects of Uniswap's documentation style and structure.

## Project Structure

Organize your Solidity contracts and interfaces in a structure similar to Uniswap V3:

```
contracts/
├── interfaces/
│   ├── pool/
│   │   ├── IExamplePoolActions.sol
│   │   ├── IExamplePoolDerivedState.sol
│   │   ├── IExamplePoolEvents.sol
│   │   ├── IExamplePoolImmutables.sol
│   │   ├── IExamplePoolOwnerActions.sol
│   │   └── IExamplePoolState.sol
│   └── IExamplePool.sol
├── libraries/
│   └── ... (various library contracts)
└── ExamplePool.sol
```

## Interface Segregation

Separate interfaces into distinct files based on functionality:

1. `IExamplePoolActions.sol`: Public actions that can be performed on the pool
2. `IExamplePoolDerivedState.sol`: Computed state variables
3. `IExamplePoolEvents.sol`: All events emitted by the pool
4. `IExamplePoolImmutables.sol`: Immutable state variables
5. `IExamplePoolOwnerActions.sol`: Owner-specific actions
6. `IExamplePoolState.sol`: Mutable state variables

Create a main interface file (e.g., `IExamplePool.sol`) that combines all these interfaces:

```solidity
/// @title The interface for the Example Pool
/// @notice Combines all the interfaces for the Example pool
interface IExamplePool is
    IExamplePoolImmutables,
    IExamplePoolState,
    IExamplePoolDerivedState,
    IExamplePoolActions,
    IExamplePoolOwnerActions,
    IExamplePoolEvents
{

}
```

## NatSpec Comments

Use comprehensive NatSpec comments for all public and external functions, state variables, and events. Include the following tags:

- @title: For contract-level documentation
- @notice: Explain what the function does in simple terms
- @dev: Provide additional details for developers
- @param: Describe each parameter
- @return: Describe return values

Example:

```solidity
/// @notice Swap tokenA for tokenB, or tokenB for tokenA
/// @dev The caller of this method receives a callback in the form of IExampleSwapCallback#exampleSwapCallback
/// @param recipient The address to receive the output of the swap
/// @param zeroForOne The direction of the swap, true for tokenA to tokenB, false for tokenB to tokenA
/// @param amountSpecified The amount of the swap, which implicitly configures the swap as exact input (positive), or exact output (negative)
/// @param sqrtPriceLimitX96 The Q64.96 sqrt price limit. If zero for one, the price cannot be less than this
/// value after the swap. If one for zero, the price cannot be greater than this value after the swap
/// @param data Any data to be passed through to the callback
/// @return amount0 The delta of the balance of token0 of the pool, exact when negative, minimum when positive
/// @return amount1 The delta of the balance of token1 of the pool, exact when negative, minimum when positive
function swap(
    address recipient,
    bool zeroForOne,
    int256 amountSpecified,
    uint160 sqrtPriceLimitX96,
    bytes calldata data
) external returns (int256 amount0, int256 amount1);
```

## Main Contract Documentation

In the main contract file:

1. Implement the combined interface
2. Use `@inheritdoc` to inherit documentation from interfaces
3. Add implementation-specific comments where necessary

Example:

```solidity
contract ExamplePool is IExamplePool, ReentrancyGuard {
    /// @inheritdoc IExamplePoolState
    uint256 public override feeGrowthGlobal0X128;

    /// @inheritdoc IExamplePoolActions
    function swap(
        address recipient,
        bool zeroForOne,
        int256 amountSpecified,
        uint160 sqrtPriceLimitX96,
        bytes calldata data
    ) external override nonReentrant returns (int256 amount0, int256 amount1) {
        // Implementation-specific comments
        // ...
    }
}
```

## Event Documentation

Provide detailed documentation for events, including all parameters:

```solidity
/// @notice Emitted when liquidity is added to a position
/// @param sender The address that initiated the liquidity addition
/// @param owner The owner of the position and recipient of any minted liquidity
/// @param tickLower The lower tick of the position
/// @param tickUpper The upper tick of the position
/// @param amount The amount of liquidity added to the position's range
/// @param amount0 The amount of tokenA added as liquidity
/// @param amount1 The amount of tokenB added as liquidity
event LiquidityAdded(
    address indexed sender,
    address indexed owner,
    int24 indexed tickLower,
    int24 indexed tickUpper,
    uint128 amount,
    uint256 amount0,
    uint256 amount1
);
```

## Inline Comments

Use inline comments sparingly and only when necessary to explain complex logic or non-obvious implementation details.

Example:

```solidity
// Continue swapping as long as there's remaining amount and we haven't reached the price limit
while (state.amountSpecifiedRemaining != 0 && state.sqrtPriceX96 != sqrtPriceLimitX96) {
    // ... (implementation details)
}
```

By adopting these Uniswap-inspired documentation conventions, you can create a well-structured, easy-to-understand codebase that follows industry best practices for complex Solidity projects.
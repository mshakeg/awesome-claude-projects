# Handling External Dependencies in BTT

When applying the Branching Tree Technique (BTT) to projects with external dependencies, it's important to strike a balance between comprehensive testing and practical implementation. This guide outlines best practices for handling external dependencies in your BTT structure and tests.

## General Principles

1. Focus on how your contract interacts with external dependencies.
2. Test your contract's behavior, not the external dependency itself.
3. Include error handling and edge cases in your .tree structure.
4. Consider both unit tests and integration tests for critical interactions.

## Approaches to Handling External Dependencies

There are two main approaches to handling external dependencies in your tests: using contract fixtures and using mocks. Each has its advantages and use cases.

### 1. Using Contract Fixtures

This approach involves deploying actual instances of the external contracts in your test environment.

#### Advantages:
- Provides a realistic testing environment
- Ensures compatibility with the actual contract implementations
- Allows testing of complex interactions that might be difficult to mock

#### Disadvantages:
- Can be slower to set up and run
- May be more complex to manipulate for specific test scenarios
- Might not easily allow testing edge cases or error conditions

#### Example: Using Uniswap V3 Fixtures

```solidity
import "@uniswap/v3-core/contracts/interfaces/IUniswapV3Factory.sol";
import "@uniswap/v3-core/contracts/interfaces/IUniswapV3Pool.sol";

contract UniswapV3Fixture is Test {
    IUniswapV3Factory public factory;
    IUniswapV3Pool public pool;
    ERC20Mock public token0;
    ERC20Mock public token1;

    function setUp() public virtual {
        // Deploy or use existing deployments of Uniswap contracts
        factory = IUniswapV3Factory(FACTORY_ADDRESS);
        token0 = new ERC20Mock("Token0", "TK0", 18);
        token1 = new ERC20Mock("Token1", "TK1", 18);

        // Create a pool
        pool = IUniswapV3Pool(factory.createPool(address(token0), address(token1), 3000));

        // Initialize the pool and add some liquidity
        // This setup is to ensure the pool is in a usable state
    }
}

contract YourContractFunction_Test is UniswapV3Fixture {
    YourContract public yourContract;

    function setUp() public override {
        super.setUp();
        yourContract = new YourContract(address(factory));
    }

    function testValidSwapInteraction() public {
        uint256 amountIn = 1000;
        token0.approve(address(yourContract), amountIn);

        uint256 balanceBefore = token1.balanceOf(address(this));
        yourContract.performSwap(address(pool), true, amountIn);
        uint256 balanceAfter = token1.balanceOf(address(this));

        assertGt(balanceAfter, balanceBefore, "Your contract should have increased token1 balance");
        // Assert your contract's state changes, event emissions, etc.
    }

    // Additional tests...
}
```

### 2. Using Mocks

This approach involves creating simplified versions of the external contracts that mimic their behavior.

#### Advantages:
- Allows fine-grained control over the "external" contract's behavior
- Can be faster to run, especially for large test suites
- Easier to set up specific scenarios, including edge cases and error conditions

#### Disadvantages:
- May not accurately represent all nuances of the actual external contract
- Risk of tests passing against the mock but failing against the real implementation
- Requires maintenance to keep in sync with changes in the external contracts

#### Example: Mocking Uniswap V3 Interaction

```solidity
contract MockUniswapV3Pool {
    struct Slot0 {
        uint160 sqrtPriceX96;
        int24 tick;
    }
    Slot0 public slot0;

    function setSlot0(uint160 _sqrtPriceX96, int24 _tick) external {
        slot0 = Slot0(_sqrtPriceX96, _tick);
    }

    function swap(
        address recipient,
        bool zeroForOne,
        int256 amountSpecified,
        uint160 sqrtPriceLimitX96,
        bytes calldata data
    ) external returns (int256 amount0, int256 amount1) {
        // Implement simplified swap logic
        amount0 = zeroForOne ? -amountSpecified : amountSpecified;
        amount1 = zeroForOne ? amountSpecified : -amountSpecified;
    }
}

contract YourContractFunction_Test is Test {
    YourContract public yourContract;
    MockUniswapV3Pool public mockPool;

    function setUp() public {
        mockPool = new MockUniswapV3Pool();
        yourContract = new YourContract(address(mockPool));
    }

    function test_YourContractSwap() public {
        mockPool.setSlot0(2**96, 0); // Set an initial price

        int256 amountSpecified = 1000;
        (int256 amount0, int256 amount1) = yourContract.performSwap(true, amountSpecified);

        assertEq(amount0, -amountSpecified, "Incorrect amount0");
        assertEq(amount1, amountSpecified, "Incorrect amount1");
    }

    // Additional tests...
}
```

## Choosing the Right Approach

- Use contract fixtures when:
  - You need to test complex interactions with the external system
  - The external system's exact behavior is crucial to your contract's functionality
  - You want to ensure compatibility with the latest version of the external contract

- Use mocks when:
  - You need to test specific edge cases or error conditions
  - The external dependency is simple or you only interact with it in limited ways
  - You want faster test execution, especially in large test suites

Often, a combination of both approaches can provide the most comprehensive testing strategy.

## BTT Structure for External Dependency Interactions

Regardless of whether you use fixtures or mocks, your .tree files should focus on how your contract interacts with the external dependency:

```
yourContractExternalInteraction.t.sol
├── given external dependency is in a normal state
│   ├── when your contract performs a valid operation
│   │   └── it should update internal state correctly
│   │   └── it should emit correct events
│   └── when your contract performs an invalid operation
│       └── it should handle the error appropriately
├── given external dependency is in an edge case state
│   └── when your contract interacts with it
│       └── it should handle the situation gracefully
└── given external dependency is unreachable
    └── when your contract attempts to interact
        └── it should handle the error and possibly revert
```

## Best Practices

1. Focus on testing your contract's behavior, not the external dependency itself.
2. Include both happy path and error case scenarios in your tests.
3. Test how your contract handles unexpected responses or states from the external dependency.
4. Regularly update your mocks or fixtures to match any changes in the external dependency.
5. Consider using integration tests on a forked mainnet to complement your unit tests with mocks or fixtures.
6. Document any assumptions made about the external dependency's behavior in your tests.

Remember, whether using contract fixtures or mocks, the goal is to ensure your contract behaves correctly when interacting with external dependencies under various conditions.
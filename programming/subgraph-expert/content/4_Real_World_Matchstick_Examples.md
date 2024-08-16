# Comprehensive Guide to Real-World Matchstick Examples

This guide provides comprehensive examples of using Matchstick to test subgraphs, based on real-world scenarios from the Uniswap V3 subgraph.

## Table of Contents
1. Setting Up the Test Environment
2. Basic Event Testing
3. Complex Event Testing: Swap Events
4. Testing Derived Calculations
5. Testing Time-Based Aggregations
6. Error Handling and Edge Cases
7. Best Practices for Subgraph Testing

## 1. Setting Up the Test Environment

Before diving into specific tests, it's crucial to set up a proper testing environment.

```typescript
import { assert, clearStore, test, describe, beforeEach } from 'matchstick-as/assembly/index'
import { Address, BigInt, ethereum } from '@graphprotocol/graph-ts'
import { handlePoolCreated, handleInitialize, handleSwap } from '../src/mappings/pool'
import { createPoolCreatedEvent, createInitializeEvent, createSwapEvent } from './event-creators'
import { USDC_WETH_03_MAINNET_POOL, TEST_CONFIG } from './constants'

describe('Uniswap V3 Subgraph Tests', () => {
  beforeEach(() => {
    clearStore()
    // Set up any global test conditions
  })

  // Test cases will be added here
})
```

### Setting Up Test Constants and Fixtures

```typescript
// constants.ts
import { Address, BigDecimal, BigInt, ethereum } from '@graphprotocol/graph-ts'
import { newMockEvent } from 'matchstick-as'

export const FACTORY_ADDRESS = '0x1F98431c8aD98523631AE4a59f267346ea31F984'
export const USDC_WETH_03_MAINNET_POOL = '0x8ad599c3a0ff1de082011efddc58f1908eb6e6d8'

export const TEST_CONFIG: SubgraphConfig = {
  factoryAddress: FACTORY_ADDRESS,
  stablecoinWrappedNativePoolAddress: USDC_WETH_03_MAINNET_POOL,
  // ... other configuration options
}

export class TokenFixture {
  address: string
  symbol: string
  name: string
  totalSupply: string
  decimals: string
  balanceOf: string
}

export const USDC_MAINNET_FIXTURE: TokenFixture = {
  address: '0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48',
  symbol: 'USDC',
  name: 'USD Coin',
  totalSupply: '300',
  decimals: '6',
  balanceOf: '1000',
}

// ... other fixtures and helper functions
```

## 2. Basic Event Testing

Start with testing simpler events like pool creation and initialization.

### Testing Pool Creation

```typescript
test('Pool is created correctly', () => {
  const poolCreatedEvent = createPoolCreatedEvent(
    USDC_WETH_03_MAINNET_POOL,
    Address.fromString('0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48'), // USDC
    Address.fromString('0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2'), // WETH
    3000, // fee
    BigInt.fromI32(60) // tickSpacing
  )

  handlePoolCreated(poolCreatedEvent)

  assert.entityCount('Pool', 1)
  assert.fieldEquals('Pool', USDC_WETH_03_MAINNET_POOL, 'token0', '0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48')
  assert.fieldEquals('Pool', USDC_WETH_03_MAINNET_POOL, 'token1', '0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2')
  assert.fieldEquals('Pool', USDC_WETH_03_MAINNET_POOL, 'feeTier', '3000')
})
```

### Testing Pool Initialization

```typescript
test('Pool is initialized correctly', () => {
  // Assume pool is already created
  const initializeEvent = createInitializeEvent(
    USDC_WETH_03_MAINNET_POOL,
    BigInt.fromString('1111111111111111'), // sqrtPriceX96
    5000 // tick
  )

  handleInitialize(initializeEvent)

  assert.fieldEquals('Pool', USDC_WETH_03_MAINNET_POOL, 'sqrtPrice', '1111111111111111')
  assert.fieldEquals('Pool', USDC_WETH_03_MAINNET_POOL, 'tick', '5000')
})
```

## 3. Complex Event Testing: Swap Events

Swap events are crucial for DEX subgraphs and require thorough testing.

### Setting Up Swap Event Tests

```typescript
import { Address, BigDecimal, BigInt, ethereum } from '@graphprotocol/graph-ts'
import { beforeAll, describe, test } from 'matchstick-as'
import { handleSwapHelper } from '../src/mappings/pool/swap'
import { Bundle, Token } from '../src/types/schema'
import { Swap } from '../src/types/templates/Pool/Pool'
import { assertObjectMatches, MOCK_EVENT, TEST_CONFIG, USDC_WETH_03_MAINNET_POOL } from './constants'

// Define a SwapFixture class to hold test data
class SwapFixture {
  sender: Address
  recipient: Address
  amount0: BigInt
  amount1: BigInt
  sqrtPriceX96: BigInt
  liquidity: BigInt
  tick: i32
}

// Create a SwapFixture instance with real transaction data
const SWAP_FIXTURE: SwapFixture = {
  sender: Address.fromString('0x6F1cDbBb4d53d226CF4B917bF768B94acbAB6168'),
  recipient: Address.fromString('0x6F1cDbBb4d53d226CF4B917bF768B94acbAB6168'),
  amount0: BigInt.fromString('-77505140556'),
  amount1: BigInt.fromString('20824112148200096620'),
  sqrtPriceX96: BigInt.fromString('1296814378469562426931209291431936'),
  liquidity: BigInt.fromString('8433670604946078834'),
  tick: 194071,
}

// Create a Swap event using the fixture data
const SWAP_EVENT = new Swap(
  Address.fromString(USDC_WETH_03_MAINNET_POOL),
  MOCK_EVENT.logIndex,
  MOCK_EVENT.transactionLogIndex,
  MOCK_EVENT.logType,
  MOCK_EVENT.block,
  MOCK_EVENT.transaction,
  [
    new ethereum.EventParam('sender', ethereum.Value.fromAddress(SWAP_FIXTURE.sender)),
    new ethereum.EventParam('recipient', ethereum.Value.fromAddress(SWAP_FIXTURE.recipient)),
    new ethereum.EventParam('amount0', ethereum.Value.fromSignedBigInt(SWAP_FIXTURE.amount0)),
    new ethereum.EventParam('amount1', ethereum.Value.fromSignedBigInt(SWAP_FIXTURE.amount1)),
    new ethereum.EventParam('sqrtPriceX96', ethereum.Value.fromUnsignedBigInt(SWAP_FIXTURE.sqrtPriceX96)),
    new ethereum.EventParam('liquidity', ethereum.Value.fromUnsignedBigInt(SWAP_FIXTURE.liquidity)),
    new ethereum.EventParam('tick', ethereum.Value.fromI32(SWAP_FIXTURE.tick)),
  ],
  MOCK_EVENT.receipt
)
```

### Testing Swap Event Handling

```typescript
describe('handleSwap', () => {
  beforeAll(() => {
    // Set up initial state, create necessary entities
    // This includes creating Token entities, setting up the Bundle, etc.
  })

  test('Swap event updates all relevant entities', () => {
    handleSwapHelper(SWAP_EVENT, TEST_CONFIG)

    // Test Factory updates
    assertObjectMatches('Factory', TEST_CONFIG.factoryAddress, [
      ['txCount', '1'],
      ['totalVolumeETH', /* expected value */],
      ['totalVolumeUSD', /* expected value */],
      ['totalFeesUSD', /* expected value */],
      // ... other factory fields
    ])

    // Test Pool updates
    assertObjectMatches('Pool', USDC_WETH_03_MAINNET_POOL, [
      ['volumeToken0', /* expected value */],
      ['volumeToken1', /* expected value */],
      ['volumeUSD', /* expected value */],
      ['feesUSD', /* expected value */],
      ['liquidity', SWAP_FIXTURE.liquidity.toString()],
      ['tick', SWAP_FIXTURE.tick.toString()],
      ['sqrtPrice', SWAP_FIXTURE.sqrtPriceX96.toString()],
      // ... other pool fields
    ])

    // Test Token updates
    assertObjectMatches('Token', USDC_MAINNET_FIXTURE.address, [
      ['volume', /* expected value */],
      ['volumeUSD', /* expected value */],
      ['feesUSD', /* expected value */],
      ['txCount', '1'],
      ['totalValueLocked', /* expected value */],
      // ... other token fields
    ])

    // Test Swap entity creation
    assertObjectMatches('Swap', MOCK_EVENT.transaction.hash.toHexString() + '-' + MOCK_EVENT.logIndex.toString(), [
      ['transaction', MOCK_EVENT.transaction.hash.toHexString()],
      ['timestamp', MOCK_EVENT.block.timestamp.toString()],
      ['pool', USDC_WETH_03_MAINNET_POOL],
      ['token0', USDC_MAINNET_FIXTURE.address],
      ['token1', WETH_MAINNET_FIXTURE.address],
      ['sender', SWAP_FIXTURE.sender.toHexString()],
      ['recipient', SWAP_FIXTURE.recipient.toHexString()],
      ['amount0', SWAP_FIXTURE.amount0.toString()],
      ['amount1', SWAP_FIXTURE.amount1.toString()],
      ['amountUSD', /* expected value */],
      // ... other swap fields
    ])

    // Test time-based aggregations
    testTimeBucketUpdates()
  })
})
```

## 4. Testing Derived Calculations

Test complex calculations like token prices and TVL.

```typescript
test('Derived ETH and USD prices are calculated correctly', () => {
  // Set up initial state with known prices
  // Perform a swap that should affect prices

  handleSwap(swapEvent)

  // Test token derivedETH values
  assert.fieldEquals('Token', USDC_ADDRESS, 'derivedETH', /* expected value */)
  assert.fieldEquals('Token', WETH_ADDRESS, 'derivedETH', /* expected value */)

  // Test USD prices in various entities
  assert.fieldEquals('Pool', USDC_WETH_03_MAINNET_POOL, 'totalValueLockedUSD', /* expected value */)
  // Add more assertions for USD values in different entities
})
```

## 5. Testing Time-Based Aggregations

Ensure that day and hour data are correctly updated across multiple events.

```typescript
function testTimeBucketUpdates(): void {
  const dayId = MOCK_EVENT.block.timestamp.toI32() / 86400
  const hourId = MOCK_EVENT.block.timestamp.toI32() / 3600

  // Test UniswapDayData updates
  assertObjectMatches('UniswapDayData', dayId.toString(), [
    ['date', dayId.toString()],
    ['volumeETH', /* expected value */],
    ['volumeUSD', /* expected value */],
    ['feesUSD', /* expected value */],
  ])

  // Test PoolDayData updates
  assertObjectMatches('PoolDayData', USDC_WETH_03_MAINNET_POOL + '-' + dayId.toString(), [
    ['date', dayId.toString()],
    ['pool', USDC_WETH_03_MAINNET_POOL],
    ['volumeUSD', /* expected value */],
    ['volumeToken0', /* expected value */],
    ['volumeToken1', /* expected value */],
    ['feesUSD', /* expected value */],
    ['tvlUSD', /* expected value */],
  ])

  // Test PoolHourData updates
  assertObjectMatches('PoolHourData', USDC_WETH_03_MAINNET_POOL + '-' + hourId.toString(), [
    ['periodStartUnix', (hourId * 3600).toString()],
    ['pool', USDC_WETH_03_MAINNET_POOL],
    ['volumeUSD', /* expected value */],
    ['volumeToken0', /* expected value */],
    ['volumeToken1', /* expected value */],
    ['feesUSD', /* expected value */],
    ['tvlUSD', /* expected value */],
  ])

  // Test TokenDayData and TokenHourData updates
  // ... (similar to PoolDayData and PoolHourData)
}

test('Multiple swaps in the same block update aggregations correctly', () => {
  const swap1 = createSwapEvent(/* params for first swap */)
  const swap2 = createSwapEvent(/* params for second swap */)

  handleSwap(swap1)
  handleSwap(swap2)

  const dayId = swap1.block.timestamp.toI32() / 86400
  assert.fieldEquals('UniswapDayData', dayId.toString(), 'volumeUSD', /* expected combined volume */)
  assert.fieldEquals('UniswapDayData', dayId.toString(), 'feesUSD', /* expected combined fees */)

  // Add similar assertions for PoolDayData, TokenDayData, etc.
})
```

## 6. Error Handling and Edge Cases

Test how the subgraph handles unusual or erroneous input.

```typescript
test('Swap with zero amounts does not affect volume calculations', () => {
  const zeroSwap = createSwapEvent(
    USDC_WETH_03_MAINNET_POOL,
    Address.fromString('0x1234...'), // sender
    Address.fromString('0x5678...'), // recipient
    BigInt.fromI32(0), // amount0
    BigInt.fromI32(0), // amount1
    BigInt.fromString('1000000000000000000'), // sqrtPriceX96
    BigInt.fromString('1000000'), // liquidity
    5000 // tick
  )

  // Capture state before swap
  const volumeBefore = /* get current volume */

  handleSwap(zeroSwap)

  // Assert that volume and fee calculations remain unchanged
  assert.fieldEquals('Factory', TEST_CONFIG.factoryAddress, 'totalVolumeUSD', volumeBefore)
  // Add more assertions for unchanged values
})

test('Handling of invalid pool addresses', () => {
  const invalidSwap = createSwapEvent(
    '0x0000000000000000000000000000000000000000', // invalid pool address
    Address.fromString('0x1234...'), // sender
    Address.fromString('0x5678...'), // recipient
    BigInt.fromI32(100), // amount0
    BigInt.fromI32(100), // amount1
    BigInt.fromString('1000000000000000000'), // sqrtPriceX96
    BigInt.fromString('1000000'), // liquidity
    5000 // tick
  )

  // This should not throw an error, but should not update any entities
  handleSwap(invalidSwap)

  assert.entityCount('Swap', 0)
  // Add assertions to ensure no entities were updated
})

test('Swap crossing price thresholds updates derived prices correctly', () => {
  // Create a swap that causes a significant price change
  const largeSwap = createSwapEvent(/* params causing large price change */)

  handleSwapHelper(largeSwap, TEST_CONFIG)

  // Assert that token derivedETH values are updated correctly
  // Check that this affects USD prices in various entities
})

## 7. Best Practices for Subgraph Testing

1. **Isolate Tests**: Use `clearStore()` before each test to ensure a clean state.
2. **Use Real Data**: Base your test cases on real transaction data when possible.
3. **Test Edge Cases**: Include tests for zero values, extremely large values, and invalid inputs.
4. **Comprehensive Coverage**: Test all event handlers and important helper functions.
5. **Verify Derived Values**: Always check that calculated fields (e.g., USD prices) are updated correctly.
6. **Time-Based Testing**: Ensure that day and hour rollover scenarios are covered in your tests.
7. **Mocking External Calls**: Use `createMockedFunction` for any contract calls your subgraph makes.
8. **Consistent Test Data**: Use fixtures or constants for frequently used test data.
9. **Descriptive Test Names**: Write clear test descriptions that explain the scenario being tested.
10. **Continuous Integration**: Integrate Matchstick tests into your CI/CD pipeline.

## Additional Testing Scenarios

### Testing Mint and Burn Events

```typescript
// handleMint.test.ts and handleBurn.test.ts
import { assert, clearStore, test } from 'matchstick-as/assembly/index'
import { handleMint, handleBurn } from '../src/mappings/pool'
import { createMintEvent, createBurnEvent, USDC_WETH_03_MAINNET_POOL, TEST_CONFIG } from './constants'

test('Mint event updates pool and position entities', () => {
  clearStore()

  const amount = BigInt.fromString('1000000')
  const amount0 = BigInt.fromString('500000')
  const amount1 = BigInt.fromString('2000000')

  const mintEvent = createMintEvent(USDC_WETH_03_MAINNET_POOL, amount, amount0, amount1)

  handleMint(mintEvent)

  assert.fieldEquals('Pool', USDC_WETH_03_MAINNET_POOL, 'liquidity', amount.toString())
  // Add more assertions for updated fields and entities

  clearStore()
})

test('Burn event updates pool and position entities', () => {
  clearStore()

  // Similar structure to the Mint test, but for Burn events

  clearStore()
})
```

### Testing Helper Functions and Derived Fields

```typescript
// utils.test.ts
import { assert, test } from 'matchstick-as/assembly/index'
import { convertTokenToDecimal, findNativePerToken } from '../src/utils'
import { USDC_MAINNET_FIXTURE, WETH_MAINNET_FIXTURE, TEST_CONFIG } from './constants'

test('convertTokenToDecimal handles different decimal places', () => {
  const amount = BigInt.fromString('1000000')
  const decimals = BigInt.fromString('6')

  const result = convertTokenToDecimal(amount, decimals)

  assert.equals(ethereum.Value.fromString('1'), ethereum.Value.fromBigDecimal(result))
})

test('findNativePerToken calculates correct price for stablecoins', () => {
  const token = createAndStoreTestToken(USDC_MAINNET_FIXTURE)
  const ethPerToken = findNativePerToken(
    token,
    TEST_CONFIG.wrappedNativeAddress,
    TEST_CONFIG.stablecoinAddresses,
    TEST_CONFIG.minimumNativeLocked
  )
  const expectedStablecoinPrice = BigDecimal.fromString('1').div(TEST_ETH_PRICE_USD)
  assert.equals(ethereum.Value.fromBigDecimal(expectedStablecoinPrice), ethereum.Value.fromBigDecimal(ethPerToken))
})

test('getNativePriceInUSD returns correct price when stablecoin is token0', () => {
  createAndStoreTestPool(USDC_WETH_03_MAINNET_POOL_FIXTURE)
  const pool = Pool.load(USDC_WETH_03_MAINNET_POOL)!
  pool.token0Price = BigDecimal.fromString('1')
  pool.save()

  const ethPriceUSD = getNativePriceInUSD(USDC_WETH_03_MAINNET_POOL, true)
  assert.equals(ethereum.Value.fromString('1'), ethereum.Value.fromBigDecimal(ethPriceUSD))
})
```

### Testing Complex Scenarios

```typescript
// derivedETH.test.ts
import { assert, clearStore, test, describe, beforeEach } from 'matchstick-as/assembly/index'
import { handlePoolCreated, handleInitialize, handleSwap } from '../src/mappings/pool'
import { createPoolCreatedEvent, createInitializeEvent, createSwapEvent, USDC_WETH_03_MAINNET_POOL, TEST_CONFIG } from './constants'
import { Token } from '../src/types/schema'

describe('Derived ETH calculations', () => {
  beforeEach(() => {
    clearStore()
    // Set up initial state, create necessary entities
  })

  test('Token derivedETH is updated after pool creation and initialization', () => {
    const poolCreatedEvent = createPoolCreatedEvent(USDC_WETH_03_MAINNET_POOL, /* other params */)
    handlePoolCreated(poolCreatedEvent)

    const initializeEvent = createInitializeEvent(USDC_WETH_03_MAINNET_POOL, /* other params */)
    handleInitialize(initializeEvent)

    const usdcToken = Token.load(USDC_MAINNET_FIXTURE.address)!
    assert.bigDecimalEquals(usdcToken.derivedETH, /* expected derived ETH value */)
  })

  test('Token derivedETH is updated after a swap', () => {
    // Assume pool is already created and initialized
    const swapEvent = createSwapEvent(USDC_WETH_03_MAINNET_POOL, /* other params */)
    handleSwap(swapEvent)

    const usdcToken = Token.load(USDC_MAINNET_FIXTURE.address)!
    const wethToken = Token.load(WETH_MAINNET_FIXTURE.address)!

    assert.bigDecimalEquals(usdcToken.derivedETH, /* expected USDC derived ETH value */)
    assert.bigDecimalEquals(wethToken.derivedETH, /* expected WETH derived ETH value */)
  })
})
```

## Conclusion

This comprehensive guide to real-world Matchstick examples demonstrates how to thoroughly test a complex subgraph like Uniswap V3. By following these examples and best practices, subgraph developers can create robust, maintainable test suites that ensure the reliability and accuracy of their subgraphs.

Key takeaways:
1. Use realistic test data and fixtures based on actual blockchain events.
2. Test all aspects of your subgraph, including event handlers, derived calculations, and time-based aggregations.
3. Pay special attention to edge cases and error handling scenarios.
4. Organize your tests logically and use descriptive names for better maintainability.
5. Leverage Matchstick's powerful assertion and mocking capabilities to create comprehensive tests.

By implementing these testing strategies, you can significantly improve the quality and reliability of your subgraph, making it easier to maintain and update as your project evolves.
# Matchstick Best Practices

## Structuring Tests

1. Organize tests by entity or event handler:
   ```typescript
   describe("Pool entity", () => {
     // Tests related to Pool entity
   })

   describe("handleMint", () => {
     // Tests for handleMint event
   })
   ```

2. Use `beforeEach` to set up common test conditions:
   ```typescript
   beforeEach(() => {
     clearStore()
     // Set up any necessary entities or mock data
   })
   ```

3. Test both positive and negative scenarios:
   ```typescript
   test("handleMint creates a new position", () => {
     // Test successful mint
   })

   test("handleMint fails with invalid input", () => {
     // Test mint with invalid data
   })
   ```

## Mocking and Fixtures

1. Create reusable fixtures for common entities and events:
   ```typescript
   export const USDC_WETH_03_MAINNET_POOL_FIXTURE: PoolFixture = {
     address: "0x8ad599c3a0ff1de082011efddc58f1908eb6e6d8",
     token0: USDC_MAINNET_FIXTURE,
     token1: WETH_MAINNET_FIXTURE,
     feeTier: "3000",
     tickSpacing: "60",
     liquidity: "100",
   }
   ```

2. Use helper functions to create mock events:
   ```typescript
   export function createNewGravatarEvent(id: i32, ownerAddress: string, displayName: string, imageUrl: string): NewGravatar {
     // Create and return a mock NewGravatar event
   }
   ```

## Assertions

1. Use specific assertions when possible:
   ```typescript
   assert.fieldEquals("Pool", poolAddress, "feeTier", "3000")
   assert.bigIntEquals(expectedLiquidity, actualLiquidity)
   ```

2. Create custom assertion helpers for complex checks:
   ```typescript
   export const assertObjectMatches = (entityType: string, id: string, obj: string[][]): void => {
     for (let i = 0; i < obj.length; i++) {
       assert.fieldEquals(entityType, id, obj[i][0], obj[i][1])
     }
   }
   ```

## Handling Complex Scenarios

1. Test interactions between multiple entities:
   ```typescript
   test("Mint affects Pool and Position entities", () => {
     // Setup Pool and Position entities
     // Perform mint operation
     // Assert changes in both Pool and Position entities
   })
   ```

2. Test derived fields and complex calculations:
   ```typescript
   test("Token derivedETH is calculated correctly", () => {
     // Setup necessary entities and mock data
     // Trigger calculation of derivedETH
     // Assert the correct calculation of derivedETH
   })
   ```

## Continuous Integration

1. Include Matchstick tests in your CI pipeline:
   ```yaml
   test:
     runs-on: ubuntu-latest
     steps:
       - uses: actions/checkout@v2
       - name: Install dependencies
         run: yarn install
       - name: Run tests
         run: yarn test
   ```

2. Use test coverage as a quality gate:
   ```yaml
   - name: Check test coverage
     run: yarn test:coverage
   - name: Fail if coverage is below threshold
     run: |
       COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
       if (( $(echo "$COVERAGE < 80" | bc -l) )); then
         echo "Test coverage is below 80%"
         exit 1
       fi
   ```

By following these best practices, you can create more robust, maintainable, and effective tests for your subgraphs using Matchstick.
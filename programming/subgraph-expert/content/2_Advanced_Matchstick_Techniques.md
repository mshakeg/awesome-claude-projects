# Advanced Matchstick Techniques

## Mocking

Matchstick allows mocking of various Graph Node functionalities:

### Event Creation

```typescript
import { newMockEvent } from "matchstick-as/assembly/index"
import { PoolCreated } from "../generated/Factory/Factory"

export function createPoolCreatedEvent(token0: Address, token1: Address, fee: i32, tickSpacing: i32, pool: Address): PoolCreated {
  let mockEvent = newMockEvent()
  let newPoolCreatedEvent = new PoolCreated(
    mockEvent.address,
    mockEvent.logIndex,
    mockEvent.transactionLogIndex,
    mockEvent.logType,
    mockEvent.block,
    mockEvent.transaction,
    mockEvent.parameters
  )

  newPoolCreatedEvent.parameters = new Array()
  // Add parameters here

  return newPoolCreatedEvent
}
```

### Contract Calls

```typescript
import { createMockedFunction } from "matchstick-as/assembly/index"

createMockedFunction(Address.fromString("0x1234..."), "functionName", "functionName(uint256):(string)")
  .withArgs([ethereum.Value.fromUnsignedBigInt(BigInt.fromI32(123))])
  .returns([ethereum.Value.fromString("mocked result")])
```

### IPFS Files

```typescript
import { mockIpfsFile } from "matchstick-as/assembly/index"

mockIpfsFile("QmHash", "tests/ipfs/file.json")
```

## Testing Dynamic Data Sources

Matchstick provides functions for testing dynamic data sources:

```typescript
import { assert, dataSourceMock } from "matchstick-as/assembly/index"

assert.dataSourceCount("TemplateName", expectedCount)
assert.dataSourceExists("TemplateName", "0x1234...")
dataSourceMock.setReturnValues("0x1234...", "network", context)
dataSourceMock.resetValues()
```

## Test Coverage

To generate a test coverage report:

1. Export your handlers in test files
2. Run: `graph test -- -c`

This will show which handlers are tested and provide a coverage percentage.

## Debugging

Matchstick provides logging capabilities for debugging:

```typescript
import { log } from "matchstick-as/assembly/log"

log.debug("Debug message", [])
log.info("Info message", [])
log.warning("Warning message", [])
log.error("Error message", [])
log.critical("Critical error")
```

These advanced techniques allow for more comprehensive and robust testing of subgraphs. Use them to create thorough test suites that cover various scenarios and edge cases in your subgraph's behavior.
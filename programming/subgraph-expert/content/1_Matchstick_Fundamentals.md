# Matchstick Fundamentals

## Introduction to Matchstick

Matchstick is a unit testing framework developed by LimeChain for testing subgraphs in The Graph protocol. It allows developers to test their mapping logic in a sandboxed environment, ensuring higher quality and reliability of deployed subgraphs.

## Setting Up Matchstick

1. Install Matchstick as a dev dependency:
   ```bash
   yarn add --dev matchstick-as
   ```

2. Ensure you have PostgreSQL installed:
   - MacOS: `brew install postgresql`
   - Linux: `sudo apt install postgresql`
   - WSL: Use Node.js v18.1.0 or later

## Basic Test Structure

Matchstick uses a describe-test pattern similar to other testing frameworks:

```typescript
import { describe, test } from "matchstick-as/assembly/index"

describe("handleNewGravatar()", () => {
  test("Should create a new Gravatar entity", () => {
    // Test logic here
  })
})
```

## Key Testing Functions

- `describe()`: Defines a test group
- `test()`: Defines a specific test case
- `beforeAll()`, `afterAll()`: Run code before/after all tests
- `beforeEach()`, `afterEach()`: Run code before/after each test

## Assertions

Matchstick provides various assertion functions:

```typescript
import { assert } from "matchstick-as/assembly/index"

assert.fieldEquals("Entity", "id", "fieldName", "expectedValue")
assert.equals(expected, actual)
assert.notInStore("Entity", "id")
assert.addressEquals(address1, address2)
```

## Running Tests

Execute tests using the Graph CLI:

```bash
graph test [options] <datasource>
```

Options include:
- `-c, --coverage`: Run tests in coverage mode
- `-d, --docker`: Run tests in a Docker container

This guide covers the basics of setting up and using Matchstick for subgraph testing. For more advanced techniques and real-world examples, refer to the other documents in this series.

## Important References

https://thegraph.com/docs/en/developing/unit-testing-framework/ - Developer documentation on unit testing with Matchstick.

https://github.com/LimeChain/matchstick - The Matchstick Github Code Repo by LimeChain

https://github.com/LimeChain/demo-subgraph - Demo Subgraph (The Graph) showcasing unit testing with Matchstick.
# Minimalistic BTT Example: Foo Contract

This example demonstrates the application of the Branching Tree Technique (BTT) to a simple smart contract using Foundry/Forge. It showcases the use of .tree files, test structure, and the use of modifiers for DRY (Don't Repeat Yourself) principle.

## Contract: ./src/Foo.sol

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.8.21;

contract Foo {
    uint256 public value;
    uint256 public bundled;

    function setValue(uint256 newValue) external {
        value = newValue;
    }

    function when(bool x) public pure returns (bool) {
        if (!x) {
            revert("x must not be false");
        } else {
            return x;
        }
    }

    function given() public view returns (uint256) {
        if (value < 100) {
            revert("value must not be less than 100");
        } else {
            return value;
        }
    }

    event NewBundled(uint256 newBundled);

    function bundledIts(uint256 x) external returns (uint256) {
        bundled = x;
        emit NewBundled(x);
        return x;
    }

    function givenWhen(bool x) external view returns (uint256, bool) {
        uint256 result0 = given();
        bool result1 = when(x);
        return (result0, result1);
    }

    function whenGivenWhen(bool x) external view returns (uint256, bool) {
        if (msg.sender != address(0xcafe)) {
            revert("the caller must be 0xcafe");
        }
        uint256 result0 = given();
        bool result1 = when(x);
        return (result0, result1);
    }
}
```

## Base Test Abstract Contract: ./test/Base.t.sol

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.8.21 <0.9.0;

import "forge-std/Test.sol";
import { Foo } from "../src/Foo.sol";

abstract contract Base_Test is Test {
    Foo internal foo;

    /// @dev A function invoked before each test case is run.
    function setUp() public virtual {
        foo = new Foo();
    }
}
```

## Example 1: Testing the 'when' function

### ./test/when/when.tree
```
when.t.sol
├── when x is false
│   └── it should revert
└── when x is true
    └── it should return x
```

### ./test/when/when.t.sol
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.8.21;

import { Base_Test } from "../Base.t.sol";

contract When_Test is Base_Test {
    function test_RevertWhen_XIsFalse() external {
        vm.expectRevert("x must not be false");
        foo.when({ x: false });
    }

    function test_WhenXIsTrue() external {
        bool returnedValue = foo.when({ x: true });
        assertTrue(returnedValue);
    }
}
```

## Example 2: Testing the 'given' function

### ./test/given/given.tree
```
given.t.sol
├── given value is less than 100
│  └── it should revert
└── given value is not less than 100
   └── it should return x
```

### ./test/given/given.t.sol
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.8.21;

import { Base_Test } from "../Base.t.sol";

contract Given_Test is Base_Test {
    function test_RevertGiven_ValueIsLessThan100() external {
        foo.setValue({ newValue: 42 });
        vm.expectRevert("value must not be less than 100");
        foo.given();
    }

    function test_GivenValueIsNotLessThan100() external {
        foo.setValue({ newValue: 1337 });
        uint256 returnedValue = foo.given();
        assertEq(returnedValue, 1337);
    }
}
```

## Example 3: Testing the 'bundledIts' function

### ./test/bundled-its/bundledIts.tree
```
bundledIts.t.sol
├── it should update storage
├── it should emit an event
└── it should return x
```

### ./test/bundled-its/bundledIts.t.sol
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.8.21;

import { Base_Test } from "../Base.t.sol";

contract BundledIts_Test is Base_Test {
    event NewBundled(uint256 newBundled);

    function test_ShouldUpdateStorage() external {
        foo.bundledIts({ x: 42 });
        uint256 bundledValue = foo.bundled();
        assertEq(bundledValue, 42);
    }

    function test_ShouldEmitAnEvent() external {
        vm.expectEmit({ emitter: address(foo) });
        emit NewBundled({ newBundled: 42 });
        foo.bundledIts({ x: 42 });
    }

    function test_ShouldReturnX() external {
        uint256 returnedValue = foo.bundledIts({ x: 42 });
        assertEq(returnedValue, 42);
    }
}
```

## Example 4: Testing the 'givenWhen' function

### ./test/given-when/givenWhen.tree
```
givenWhen.t.sol
├── given value is less than 100
│  └── it should revert
└── given value is not less than 100
   ├── when x is false
   │  └── it should revert
   └── when x is true
      └── it should return x
```

### ./test/given-when/givenWhen.t.sol
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.8.21;

import { Base_Test } from "../Base.t.sol";

contract GivenWhen_Test is Base_Test {
    function test_RevertGiven_ValueIsLessThan100() external {
        foo.setValue({ newValue: 42 });
        vm.expectRevert("value must not be less than 100");
        foo.givenWhen({ x: true });
    }

    // Using a modifier to set up the "given" condition
    modifier givenValueIsNotLessThan100() {
        foo.setValue({ newValue: 1337 });
        _;
    }

    function test_RevertWhen_XIsFalse() external givenValueIsNotLessThan100 {
        vm.expectRevert("x must not be false");
        foo.givenWhen({ x: false });
    }

    function test_WhenXIsTrue() external givenValueIsNotLessThan100 {
        (uint256 returnedValue0, bool returnedValue1) = foo.givenWhen({ x: true });
        assertEq(returnedValue0, 1337);
        assertTrue(returnedValue1);
    }
}
```

## Example 5: Testing the 'whenGivenWhen' function

### ./test/when-given-when/whenGivenWhen.tree
```
whenGivenWhen.t.sol
├── when the caller is unknown
│  └── it should revert
└── when the caller is known
   ├── given value is less than 100
   │  └── it should revert
   └── given value is not less than 100
      ├── when x is false
      │  └── it should revert
      └── when x is true
         └── it should return x
```

### ./test/when-given-when/whenGivenWhen.t.sol
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.8.21;

import { Base_Test } from "../Base.t.sol";

contract WhenGivenWhen_Test is Base_Test {
    function test_RevertWhen_TheCallerIsUnknown() external {
        vm.startPrank({ msgSender: address(0xbeef) });
        vm.expectRevert("the caller must be 0xcafe");
        foo.whenGivenWhen({ x: true });
    }

    // Modifier to set up the "when" condition for the caller
    modifier whenTheCallerIsKnown() {
        vm.startPrank({ msgSender: address(0xcafe) });
        _;
    }

    function test_RevertGiven_ValueIsLessThan100() external whenTheCallerIsKnown {
        foo.setValue({ newValue: 42 });
        vm.expectRevert("value must not be less than 100");
        foo.whenGivenWhen({ x: true });
    }

    // Modifier to set up the "given" condition for the value
    modifier givenValueIsNotLessThan100() {
        foo.setValue({ newValue: 1337 });
        _;
    }

    function test_RevertWhen_XIsFalse() external whenTheCallerIsKnown givenValueIsNotLessThan100 {
        vm.expectRevert("x must not be false");
        foo.whenGivenWhen({ x: false });
    }

    function test_WhenXIsTrue() external whenTheCallerIsKnown givenValueIsNotLessThan100 {
        (uint256 returnedValue0, bool returnedValue1) = foo.whenGivenWhen({ x: true });
        assertEq(returnedValue0, 1337);
        assertTrue(returnedValue1);
    }
}
```

This example demonstrates:
1. Creating .tree files to visualize test scenarios
2. Structuring test contracts based on .tree files
3. Using modifiers to set up test conditions and apply DRY principle
4. Testing different execution paths including success and revert cases
5. Using Forge's testing utilities like `vm.expectRevert` and `vm.startPrank`

The BTT approach helps ensure comprehensive test coverage and maintains a clear relationship between the test structure and the contract's functionality.
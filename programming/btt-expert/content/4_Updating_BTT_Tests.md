# Updating BTT Tests

## When to Update

1. When the contract's external/public functions change
2. When new edge cases or execution paths are identified
3. During regular code reviews and maintenance

## Update Process

1. Review the changes in the contract code
2. Update the corresponding .tree file to reflect new or changed execution paths
3. Modify the test contract to align with the updated .tree file
4. Add new test functions for new branches
5. Adjust existing test functions as needed
6. Update modifiers if necessary
7. Run the updated tests to ensure all new and existing cases pass

## Tips for Efficient Updates

1. Use version control to track changes in .tree files and test contracts
2. Comment on significant changes for future reference
3. Consider using automated tools to assist in identifying changes that require test updates
4. Regularly review the entire BTT structure to ensure it remains coherent and comprehensive

## Example Update Scenario

Original function:
```solidity
function transfer(address to, uint256 amount) public returns (bool) {
    require(balanceOf[msg.sender] >= amount, "Insufficient balance");
    balanceOf[msg.sender] -= amount;
    balanceOf[to] += amount;
    return true;
}
```

Updated function:
```solidity
function transfer(address to, uint256 amount) public returns (bool) {
    require(to != address(0), "Invalid recipient");
    require(balanceOf[msg.sender] >= amount, "Insufficient balance");
    balanceOf[msg.sender] -= amount;
    balanceOf[to] += amount;
    emit Transfer(msg.sender, to, amount);
    return true;
}
```

In this case, you would:
1. Add a new branch to the .tree file for the "Invalid recipient" check
2. Create a new test function `test_RevertWhen_RecipientIsZeroAddress()`
3. Update the existing test functions to check for the Transfer event emission

Remember to thoroughly test all new and modified code paths after updates.
# Solidity Documentation Best Practices

## General Guidelines

1. Be concise yet informative
2. Use consistent terminology throughout the codebase
3. Document all public and external functions
4. Explain complex algorithms or business logic
5. Keep comments up-to-date with code changes

## Code-Level Documentation

- Use inline comments for complex code sections
- Explain the "why" behind important decisions
- Document any assumptions or preconditions

## Contract-Level Documentation

- Provide an overview of the contract's purpose and functionality
- List any dependencies or inherited contracts
- Explain the contract's role within the larger system

## Security Considerations

- Document any potential security risks or considerations
- Explain access control mechanisms
- Note any critical state changes or value transfers

## Example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

/// @title Voting Contract
/// @notice Manages a simple voting system
/// @dev Implements basic voting functionality with time-based restrictions
contract Voting {
    // State variables documentation...

    /// @notice Cast a vote for a candidate
    /// @dev Checks if voting is open and voter hasn't voted before
    /// @param _candidateId The ID of the candidate being voted for
    /// @return success True if the vote was cast successfully
    function vote(uint256 _candidateId) public returns (bool success) {
        // Function implementation
        // ...

        // Critical state change
        // Update vote count and mark voter as having voted
        // ...

        return true;
    }

    // More functions...
}
```
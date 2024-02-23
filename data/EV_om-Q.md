## [L-01] Redundant Length Check in setWinningTickets

### Context
https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L291

### Description
In the `setWinningTickets` function, there is a length check on the `_winningTickets` array to ensure it matches the expected number of winners (`numWinners`). This check is redundant and results in unnecessary gas consumption because `_winningTickets` is always initialized with a length equal to `numWinners`:

```solidity
uint256[] memory _winningTickets = new uint256[](numWinners);
for (uint256 i = 0; i < numWinners; i++) {
    _winningTickets[i] = revealRandomNumber(sequenceNumbers[i], userRandoms[i], providerRandoms[i]);
    emit SetWinningTicket(_round, _prizeIndex, _winningTickets[i], i);
}
winningTickets[_round][_prizeIndex] = _winningTickets;
require(_winningTickets.length == numWinners, "WTL");
```

### Recommendation 
It is recommended to remove the length check on the `_winningTickets` array to simplify the code and optimize gas usage. This change will not introduce any additional issues as the initialization of `_winningTickets` inherently guarantees its length matches `numWinners`.

## [L-02] Code Duplication in ThrusterYield Contract

### Context
https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-clmm/contracts/ThrusterGas.sol#L21-L37
https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-cfmm/contracts/ThrusterYield.sol#L44-L55

### Description
The `ThrusterYield` contract contains code that is identical to what is implemented in the `ThrusterGas` contract. Specifically, the functionality for claiming gas is replicated in both contracts. This redundancy violates the DRY (Don't Repeat Yourself) principle, making the codebase harder to maintain and more susceptible to bugs if updates are required in the future.

### Recommendation 
The `ThrusterYield` contract could inherit from the `ThrusterGas` contract, eliminating the need to replicate the gas claiming functionality. This approach not only reduces the overall codebase size but also simplifies future maintenance and updates.
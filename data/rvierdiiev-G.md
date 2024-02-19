## G-01. Remove `requestedRandomNumber` field.
### Description
`ThrusterTreasure.requestedRandomNumber` variable is used to [store who have requested random](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L229). This was copied from Pyth entrophy example and in that example user were able to request random. But in ThrusterTreasure contract only owner can request random. Thus this functionality is not needed and can be removed.

## G-02. Remove unneeded check.
### Description
`ThrusterTreasure.claimPrizesForRound` function has check that [`round.ticketEnd > round.ticketStart`](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L106). This situation is not possible, as round for user stores [amount of his tickets](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L95C76-L95C90) and those tickets [always bigger than 0](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L89).
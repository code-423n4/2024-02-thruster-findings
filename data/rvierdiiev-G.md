## G-01. Remove `requestedRandomNumber` field.
### Description
`ThrusterTreasure.requestedRandomNumber` variable is used to [store who have requested random](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L229). This was copied from Pyth entrophy example and in that example user were able to request random. But in ThrusterTreasure contract only owner can request random. Thus this functionality is not needed and can be removed.

## G-02. Remove unneeded check.
### Description
`ThrusterTreasure.claimPrizesForRound` function has check that [`round.ticketEnd > round.ticketStart`](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L106). This situation is not possible, as round for user stores [amount of his tickets](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L95C76-L95C90) and those tickets [always bigger than 0](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L89).

## G-03. Do not configure claimable native yield for ThrusterPool
### Description
ThrusterPool contract is uniswap v3 like pool, which is not supposed to receive native eth. Thus there is [no reason to set claimable native yield for it](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-clmm/contracts/ThrusterPool.sol#L130) and no reason [to claim it](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-clmm/contracts/ThrusterPool.sol#L810).

There is similar thing is uniswap v2 like pool.
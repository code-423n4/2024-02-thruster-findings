### Gas-01 Emitting two duplicated events might be unnecessary and wasteful.
**Instances(2)**
**Total Gas Saved(2250)**

Both ThrusterPool.sol and ThrusterPair.sol's swap function emit duplicated events - once locally from the pool/pair contract, once from the factory contract. Emit the same event twice is wasteful and might also be unnecessary for the use case.

According to readme, the protocol would prefer to have swap event centrally emitted from factory contracts for tracking. This is fine, then it could be argued that the local swap emit from the pool/pair contract is unnecessary. If the individual pool deployer needs to track swap events, they can subscribe to the factory contract's event which contains the origin pool address as an indexed field.

```solidity
//thruster-protocol/thruster-cfmm/contracts/ThrusterPair.sol
    function swap(
        uint256 amount0Out,
        uint256 amount1Out,
        address to,
        bytes calldata data
    ) external lock {
...
        emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
        IThrusterFactory(factory).emitSwap(
            msg.sender,
            amount0In,
            amount1In,
            amount0Out,
            amount1Out,
            to
        );
```
(https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-cfmm/contracts/ThrusterPair.sol#L222-L223)
```solidity
//thruster-protocol/thruster-clmm/contracts/ThrusterPool.sol
    function swap(
        address recipient,
        bool zeroForOne,
        int256 amountSpecified,
        uint160 sqrtPriceLimitX96,
        bytes calldata data
    )
        external
        override
        noDelegateCall
        returns (int256 amount0, int256 amount1)
    {
...
        emit Swap(
            msg.sender,
            recipient,
            amount0,
            amount1,
            state.sqrtPriceX96,
            state.liquidity,
            state.tick
        );
        IThrusterPoolFactory(factory).emitSwap(
            msg.sender,
            recipient,
            amount0,
            amount1,
            state.sqrtPriceX96,
            state.liquidity,
            state.tick
        );
        slot0.unlocked = true;
```
(https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-clmm/contracts/ThrusterPool.sol#L756-L759)

Note for each emit log, there is at least 375 static gas cost + 375 gas per topic + other dynamic cost.

Recommendations:
Save users's runtime gas cost by only emitting swap event in the factory contract.


### Gas-02 Wasteful runtime gas cost due to maximum for-loop iterations
**Instances(1)**
**Total Gas Saved(Various)**

In ThrusterTreasure.sol::claimPrizesForRound(), for-loop will always run maximum iterations(`maxPrizeCount`) regardless of the actual maximum prize index for a given round.

For example, when `maxPrizeCount` is set to 4 and there is only one prizeIdx set for a round, the for-loop maxPrizeCount_ will still run 4 times. This will cost extra gas for users. 

Each for-loop iteration has an overhead + any extra cost in checking the for-loop round + for-loop body gas cost. 

```solidity
//thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol
    function claimPrizesForRound(uint256 roundToClaim) external {
...
|>      for (uint256 i = 0; i < maxPrizeCount_; i++) {
            Prize memory prize = prizes[roundToClaim][i];
            uint256[] memory winningTicketsRoundPrize = winningTickets[
                roundToClaim
            ][i];
            for (uint256 j = 0; j < winningTicketsRoundPrize.length; j++) {
                //@audit-info ? can winningTicket be 0?
                uint256 winningTicket = winningTicketsRoundPrize[j];
                if (
                    round.ticketStart <= winningTicket &&
                    round.ticketEnd > winningTicket
                ) {
                    _claimPrize(prize, msg.sender, winningTicket);
                }
            }
```
(https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L108-L117)

Recommendations:
Consider for each round, set a round-specific max prize index to limit the for-loop iterations instead of using a global max number.

  



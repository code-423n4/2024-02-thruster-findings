### Low-01 Incorrect comment -  Fee should be between 1/6 and 1/2 of the growth in sqrt(k)

**Instances(1)**

ThrusterPair.sol allows fee/yieldCut to be adjusted between 1/6 and 1/2 (15% to 50%) of growth in sqrt(k). And the yieldCut is calculated in `_mintYieldCut()`. 

However, `_mintYieldCut()` has incorrect comments which state that the fee is always 1/6th. 

```solidity
//thruster-protocol/thruster-cfmm/contracts/ThrusterPair.sol
      //@audit : Incorrect comment - fee should be between 1/6 th and 1/2 of the growth in sqrt(k)
|>    // if yieldCut is on, mint liquidity equivalent to 1/6th of the growth in sqrt(k)
    function _mintYieldCut(
        uint112 _reserve0,
        uint112 _reserve1
    ) private returns (bool yieldCutOn) {
```
(https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-cfmm/contracts/ThrusterPair.sol#L120)

Recommendations:
Change the comment to reflect the range `1/6 and 1/2`. 

### Low-02 Unnecessary inheritance of IThrusterERC20 in TrusterPair.sol

**Instances(1)**

Based on readme, a design decision is made to flatten ERC20 function implementations in TrustedPair.sol. As a result, ERC20 methods are included in both TrusterPair.sol and ITrustedPair.sol. 

There is no need to add to TrustedPair.sol's inheritance chain with addition of IThrusterERC20.sol which is a subset of ITrusterPair.sol

```solidity
//thruster-protocol/thruster-cfmm/contracts/ThrusterPair.sol

contract ThrusterPair is IThrusterPair, IThrusterERC20, ThrusterYield {
...
//thruster-protocol/thruster-cfmm/interfaces/IThrusterPair.sol
interface IThrusterPair {
    event Approval(address indexed owner, address indexed spender, uint256 value);
    event Transfer(address indexed from, address indexed to, uint256 value);
...
    function name() external pure returns (string memory);
    function symbol() external pure returns (string memory);
    function decimals() external pure returns (uint8);
    function totalSupply() external view returns (uint256);
    function balanceOf(address owner) external view returns (uint256);
    function allowance(address owner, address spender) external view returns (uint256);
...
```
(https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-cfmm/contracts/ThrusterPair.sol#L15)

Recommendations:
Remove IThrusterERC20 inheritance.

### Low-03 ThrusterPair may have an outdated manager address due to a vulnerable manager update mechanism. 
**Instances(1)**

A manager role in ThrusterPair(inheriting ThrusterYield) can claim Blast rewards generated from the contract. This manager address should be the same as `yieldTo` address in ThrusterPoolFactory.sol.

For each pool/pair, the manager role can be updated in `ThrusterYield::setManager()` manually by the current manager/`yieldTo` address. Each pair contract will have its isolated `manager` address in storage. 
However, this is not an efficient mechanism to sync the manager address across all deployed pools/pairs with ThrusterFactory. Here's one challenging scenario:

Too many pairs to update for the manager or too costly. Due to pool deployment being permissionless, there could be too many pairs for all the manager addresses to be updated efficiently and affordably over time. When not all pair contracts are updated at the same time, some pair contracts will have an outdated manager address. 

In addition, if the outdated manager address is compromised, the old manager address can also potentially hijack the Blast rewards of those pair contracts. 

```solidity
//thruster-protocol/thruster-cfmm/contracts/ThrusterYield.sol

    function setManager(address _manager) external onlyManager {
        manager = _manager;
    }

```
The protocol calling each pair contracts to update the `manager` address is not an ideal update mechanism. It opens up risk for outdated manager addresses to persist in some pair contracts, and also compromised old manager accounts to hijack pair Blast rewards.

Recommendations:
In ThrusterYield.sol, instead of storing an isolated `manager` address, in `onlyManager()`, simply call ThrusterFactory for the most current `yieldTo` address.

### Low-04 `claimPrizesForRound()` doesn't satisfy check-effect-interaction patterns
**Instances(1)**

In ThrusterTreasure.sol, `claimPrizesForRound()` is a public function that allows users/winners to claim prizes. But this function doesn't satisfy the `check-effect-interaction` pattern and might become vulnerable to reentrancy if the external interactions allow callback functions.

Notably, `claimPrizesForRound()` will first call external interactions `_claimPrize()` which involves external token transfers in a for-loop. And it only updates users round info `entered[msg.sender][roundToClaim]` at the end of the function call. This means, if user can re-enter `claimPrizesForRound()`, user's `entered[msg.sender][roundToClaim]` will not be cleared, potentially allowing users to `_claimPrize()` multiple times.

```solidity
//thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol
    function claimPrizesForRound(uint256 roundToClaim) external {
...
        for (uint256 i = 0; i < maxPrizeCount_; i++) {
                         //@audit interactions
...            |>        _claimPrize(prize, msg.sender, winningTicket);
...
    }
          //@audit effects
|>        entered[msg.sender][roundToClaim] = Round(0, 0, roundToClaim); // Clear user's tickets for the round
        emit CheckedPrizesForRound(msg.sender, roundToClaim);
    }
```
(https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L114-L118)

Recommendations:
Update `entered[msg.sender][roundToClaim]` before calling `_claimPrize()`.


### Low-05 Unnecessary code in ThrusterTreasure::requestRandomNumberMany and ThrusterTreasure::requestRandomNumber
**Instances(2)**

In ThrusterTreasure.sol, both `requestRandomNumberMany()` and `requestRandomNumber()` contains a line of unnecessary code `requestedRandomNumber[sequenceNumber] = msg.sender;`.

Due to `requestRandomNumberMany()` and `requestRandomNumber()` only allows owner to call and request random number from Etropy, `msg.sender` will always be owner. Also `sequenceNumber` is a return intermediate value from Etropy, and will only be meaningful when paired with the corresponding random numbers and commitment values that are generated off-chain. So storing a single `sequenceNumber` with always the same owner address only costs more gas with no meaningful impact.

In addition, the storage mapping `requestedRandomNumber[sequenceNumber]` can also be removed since it is never read on-chain and will not be helpful for the off-chain process either because it doesn't show the pairing of `userCommitment` and `sequenceNumber`. Simply `emit RandomNumberRequest` will be sufficient for the off-chain process. 

```solidity
//thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol

    function requestRandomNumber(
        bytes32 userCommitment
    ) external payable onlyOwner returns (uint64) {
...
        //@audit unnecessary, can remove. 
|>      requestedRandomNumber[sequenceNumber] = msg.sender;
        emit RandomNumberRequest(sequenceNumber, userCommitment);
       
...
  }

    function requestRandomNumberMany(
        bytes32[] calldata userCommitments
    ) external payable onlyOwner returns (uint64[] memory seqNums) {
...
            //@audit unnecessary, can remove. 
|>          requestedRandomNumber[sequenceNumber] = msg.sender;
            emit RandomNumberRequest(sequenceNumber, userCommitments[i]);
...
```
(https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L243)

Recommendations:
Remove `requestedRandomNumber[sequenceNumber] = msg.sender;`

### Low-06 Unnecessary implementations for ETH rewards - Pools can’t receive ETH in balance and will never be eligible for ETH rewards. 
**Instances(2)**

Three contracts have implementations for ETH Blast rewards but these contracts don't implement `receive()` or `payable` functions to ever receive eth after deployment. Contracts that do not hold ETH will not be eligible for ETH blast rewards. So no need for these ETH blast reward implementations in these contracts.

```solidity
//thruster-protocol/thruster-clmm/contracts/ThrusterPool.sol
constructor() {
        int24 _tickSpacing;
        BLAST.configureClaimableGas();
        //@audit Pool will not in normal circumstance be able to receive ETH in balance and will not be eligible for ETH rewards.
|>      BLAST.configureClaimableYield();
        USDB.configure(YieldMode.CLAIMABLE);
        WETHB.configure(YieldMode.CLAIMABLE);
...
```
(https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-clmm/contracts/ThrusterPool.sol#L130)
```solidity
//thruster-protocol/thruster-cfmm/contracts/ThrusterYield.sol
    constructor(address _manager) public {
        BLAST.configureClaimableGas();
|>      BLAST.configureClaimableYield();
        USDB.configure(IERC20Rebasing.YieldMode.CLAIMABLE);
        WETHB.configure(IERC20Rebasing.YieldMode.CLAIMABLE);
        manager = _manager;
    }
...
```
(https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-cfmm/contracts/ThrusterYield.sol#L26)

Recommendations:
Remove both the ETH rewards configuration and the claiming code for ETH rewards for contracts not able to receive ETH.

### Low-07 A user can withdraw prizes immediately after winning tickets drawn, bypassing the intended 'next draw period' delay stated by Doc
**Instances(1)**

In Thruster [Doc](https://docs.thruster.finance/docs/products/thruster-treasure/overview), there is a mention of the timing of winner claim prizes which states `then a user will be subsequently prompted to claim their prize once the next draw period starts`. 

There are no on-chain mechanisms to ensure that winners only claim prizes at the start of `the next draw period`. In fact, a winner can claim their prize immediately after the owner draws winning tickets through `setWinningTickets()`. 

A winner can view the winning ticket result through querying `mapping(uint256 => mapping(uint256 => uint256[])) public winningTickets;` for `currentRound` and prizeIndexes from 0 to maximum. Then a winner can directly call `claimPrizesForRound()` to claim prizes immediately. 

If the intention is to unify a prize-winning window for a given round (as stated in Doc), then add a check in `claimPrizesForRound()` to ensure a minimal delay timestamp.

```solidity
//

    function claimPrizesForRound(uint256 roundToClaim) external {
        //@audit note: as long as MAX_ROUNT_TIME(30 days) hasn't passed, user can claim prizes as soon as the owner set `winningTickets` for the round. No delay in claiming is enforced.
        require(
            roundStart[roundToClaim] + MAX_ROUND_TIME >= block.timestamp,
            "ICT"
        );
        require(winningTickets[roundToClaim][0].length > 0, "NWT");
...
```
(https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L103-L104)

Recommendations:
If the intention is to unify a prize-winning window for a given round to 'next draw period starts'(as stated in doc), then add a check in `claimPrizesForRound()` to ensure a minimal delay timestamp.

### Low-08 `reqeustRandomNumberMany()` has incorrect array initialization
**Instances(1)**

In ThrusterTreasure.sol, `requestRandomNumberMany()` has incorrect array initialization, which causes the tx to revert. Specifically, a dynamic array `uint64[] memory seqNums` is declared as a return variable, but this variable is never initialized before writing to it through `seqNums[i]`.

A dynamic array declared in memory needs to be initialized with length info first like this  `seqNums = new uint64[](userCommitments.length)`.
```solidity
//thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol
    function requestRandomNumberMany(
        bytes32[] calldata userCommitments
    ) external payable onlyOwner returns (uint64[] memory seqNums) {
        uint256 fee = entropy.getFee(entropyProvider);
        require(address(this).balance >= fee * userCommitments.length, "IF");
        for (uint256 i = 0; i < userCommitments.length; i++) {
            uint64 sequenceNumber = entropy.request{value: fee}(
                entropyProvider,
                userCommitments[i],
                true
            );
            //@audit this will cause tx revert due to seqNums haven't been initialized with userCommitments.length. 
|>          seqNums[i] = sequenceNumber;
            requestedRandomNumber[sequenceNumber] = msg.sender;
            emit RandomNumberRequest(sequenceNumber, userCommitments[i]);
        }
    }
```
(https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L228)

For reference, run the below test contract in remix, method1 will revert but method2 will succeed.
```solidity
pragma solidity ^0.8.23;

contract DynamicArrayTest {

        function updateDynamicArray1(uint[] calldata newData) external pure returns (uint[] memory arrays){
        for (uint i = 0; i < newData.length; i++) {
            arrays[i] = newData[i];
        }
    }

    function updateDynamicArray2(uint[] calldata newData) external pure returns (uint[] memory arrays){
        arrays= new uint[](newData.length);
        for (uint i = 0; i < newData.length; i++) {
            arrays[i] = newData[i];
        }
    }
}
```
Recommendations:
Initiate array `seqNums` first with length data before updating its value.


### Low-09 User's prize might be forever locked if the setting winning tickets transaction is settled on the end timestamp of a round.
**Instances(1)**

In ThrusterTreasure.sol, there is no on-chain check to ensure owner's `setWinningTickets()` transaction will not settle on the max end timestamp of a round `roundStart[_round] + MAX_ROUND_TIME`. 

If `setWinningTickets()` happens to settle on `roundStart[_round] + MAX_ROUND_TIME`, `setWinningTickets()` transaction will still succeed. But in this edge case, winners are likely not to be able to claim their prizes for the round, because `claimPrizesForRound() contains the same check for `roundStart[roundToClaim] + MAX_ROUND_TIME >= block.timestamp`. If user's claiming tx settles after `roundStart[roundToClaim] + MAX_ROUND_TIME`, claiming tx will revert, and users will not be able to claim their prizes.
```solidity
//thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol

    function setWinningTickets(
        uint256 _round,
        uint256 _prizeIndex,
        uint64[] calldata sequenceNumbers,
        bytes32[] calldata userRandoms,
        bytes32[] calldata providerRandoms
    ) external onlyOwner {
        require(roundStart[_round] + MAX_ROUND_TIME >= block.timestamp, "ICT");
...
}

    function claimPrizesForRound(uint256 roundToClaim) external {
        require(
            roundStart[roundToClaim] + MAX_ROUND_TIME >= block.timestamp,
            "ICT"
        );
...
}
```
(https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L276)

Recommendations:
Although the owner can decide on the timing to set winning tickets, it will be good to ensure on-chain that there will always be a grace period between `setWinningTickets()` and `claimPrizesForRound()`. So do not use the same end timestamp check for both. 


### Low-10  A hardcoded minimal liquidity might be insufficient for assets with higher token decimals.
**Instances(1)**

In ThrusterPair.sol, `MINIMUM_LIQUIDITY` is hardcoded as 1000. This minimum liquidity will be locked at the first deposit as a permanent supply. And also used to prevent initial dust deposit. 

However, it should be noted that 1000 might not be suitable for assets with higher decimals. Dust value first deposit might allow initial pool reserve ratio manipulation at a cheap price, and the malicious first depositor can profit from the initial unbalanced reserve ratio through arbitrage.

Given pool deployment is permissionless, it's better to allow a range of configurations of the minimum liquidity value at ThrusterPair initialization, such that the pool deployer can configure reasonable minimum liquidity to prevent cheap initial price manipulation with vulnerable assets.

```solidity
//thruster-protocol/thruster-cfmm/contracts/ThrusterPair.sol
    uint256 public constant MINIMUM_LIQUIDITY = 10 ** 3;
```
(https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-cfmm/contracts/ThrusterPair.sol#L19)

Recommendations:
Consider allowing configuration of minimum liquidity value in `initialize()`.

### Low-11 Thruster pairs are vulnerable to first-depositor price manipulation.
**Instances(1)**

Compared to ThrusterPool.sol, ThrusterPair.sol follows UniswapV2 logic and allows any reserve ratio to be deposited by the first depositor.

Due to ThrusterPair.sol only allows depositing liquidity at the existing reserve ratio, the second depositor might be vulnerable to depositing reserves at an unfavorable ratio, which allows the first depositor to profit through back-running unsuspected depositors with swap tx for profits.

As an example, the first depositor might deposit a cheap token(tokenA) and WETH at a ratio (1:1) which inflates tokenA's price in the pool. The first depositor can deposit dust amount of both just enough to pass `MINIMUM_LIQUIDITY` threshold (e.g. 1100 wei tokenA: 1100 wei WETH). If the second depositor is not as technically proficient, they might simply deposit both token at the normal price (e.g. 1000 ether tokenA : 1 ether WETH). This would allow first depositor to back-run with a swap tx to swap tokenA for WETH taking advantage of the inflated price of tokenA and increased liquidity and realizing a profit.

There could be other scenarios of similar attacks, including the pool deployer maliciously creating a pool and atomically first depositing with a low but unbalanced reserve ratio. 

Note that price manipulations are also possible at later deposits as long as the pair liquidity is low enough for the attacker.

```solidity
//thruster-protocol/thruster-cfmm/contracts/ThrusterPair.sol
    function mint(address to) external lock returns (uint256 liquidity) {
...
        if (_totalSupply == 0) {
            liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
            _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens
        } else {
            liquidity = Math.min(
                amount0.mul(_totalSupply) / _reserve0,
                amount1.mul(_totalSupply) / _reserve1
            );
        }
        require(liquidity > 0, "ThrusterPair: INSUFFICIENT_LIQUIDITY_MINTED");
        _mint(to, liquidity);
...
```
(https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-cfmm/contracts/ThrusterPair.sol#L151-L158)

Recommendations:
Might consider allowing pool deployers to suggest a reasonable reserve ratio at the pair initialization process `initialize()`. Although this will not prevent pool deployer from being malicious, this can mitigate certain price manipulation attempts.

### Low-12 Unnecessary code - `_winningTickets` length will be the same as numWinners
**Instances(1)**

In ThrusterTreasure.sol, `setWinningTickets()` will check `_winningTickets.length == numWinners` in a require statement at the end of the function. This check is unnecessary due to `_winningTickets` is initialized with `numWinners`. So if code runs till the end of the function (no reverts), `_winningTickets.length == numWinners` will always be true. Even if `revealRandomNumber()` silently return default 0 value, the length of `_winningTickets` stays the same.

```solidity
//thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol
    function setWinningTickets(
        uint256 _round,
        uint256 _prizeIndex,
        uint64[] calldata sequenceNumbers,
        bytes32[] calldata userRandoms,
        bytes32[] calldata providerRandoms
    ) external onlyOwner {
...
        uint256[] memory _winningTickets = new uint256[](numWinners);
...
        require(_winningTickets.length == numWinners, "WTL");
}
```
(https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L291)

Recommendations:
Remove the last require statement.

### Low-13 Users who deposit to a ThrusterPool with a rebasing token that allows negative rebasing might incur losses

ThrusterPool.sol is based on UniswapV3 logic which allows concentrated liquidity. A liquidity provider will gain trading fees once their position becomes active. 

According to UniswapV3 [doc](https://docs.uniswap.org/concepts/protocol/integration-issues), a pool that has a potentially negative rebasing tokens will cause liquidity providers to incure losses.

>Rebasing tokens will succeed in pool creation and swapping, but liquidity providers will bear the loss of a negative rebase when their position becomes active, with no way to recover the loss.

Recommendations:
Consider disallowing pools with negative rebasing tokens. 

### Low-14 Use `_safeMint` instead of `_mint` in NonfungiblePositionManager.sol
In NonfungiblePositionManager::mint(), `_mint()` is used instead of `_safeMint()`. 

NonfungiblePositionManager inherits openzeppelin's ERC721.sol which provides both `_mint()` and `_safeMint()`. _mint() is [discouraged](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/d4d8d2ed9798cc3383912a23b5e8d5cb602f7d4b/contracts/token/ERC721/ERC721.sol#L271) in favor of _safeMint() which ensures that the recipient is either an EOA or implements IERC721Receiver. 

Current `_mint()` might cause user to lose their NFT position if the caller is a contract but has no means to manage ERC721 tokens.

```solidity
//thruster-protocol/thruster-clmm/contracts/NonfungiblePositionManager.sol
    function mint(
        MintParams calldata params
    )
...
        _mint(params.recipient, (tokenId = _nextId++));

```
(https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-clmm/contracts/NonfungiblePositionManager.sol#L153)

Recommendation: 
Use `_safeMint()` instead.


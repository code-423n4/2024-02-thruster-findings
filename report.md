---
sponsor: "Thruster"
slug: "2024-02-thruster"
date: "2024-03-28"
title: "Thruster Invitational"
findings: "https://github.com/code-423n4/2024-02-thruster-findings/issues"
contest: 334
---

# Overview

## About C4

Code4rena (C4) is an open organization consisting of security researchers, auditors, developers, and individuals with domain expertise in smart contracts.

A C4 audit is an event in which community participants, referred to as Wardens, review, audit, or analyze smart contract logic in exchange for a bounty provided by sponsoring projects.

During the audit outlined in this document, C4 conducted an analysis of the Thruster smart contract system written in Solidity. The audit took place between February 16—February 23 2024.

## Wardens

In Code4rena's Invitational audits, the competition is limited to a small group of wardens; for this audit, 4 wardens contributed reports:

  1. [rvierdiiev](https://code4rena.com/@rvierdiiev)
  2. [EV\_om](https://code4rena.com/@EV_om)
  3. [oakcobalt](https://code4rena.com/@oakcobalt)
  4. [0xDING99YA](https://code4rena.com/@0xDING99YA)

This audit was judged by [0xleastwood](https://code4rena.com/@leastwood).

Final report assembled by [liveactionllama](https://twitter.com/liveactionllama).

# Summary

The C4 analysis yielded an aggregated total of 5 unique vulnerabilities. Of these vulnerabilities, 0 received a risk rating in the category of HIGH severity and 5 received a risk rating in the category of MEDIUM severity.

Additionally, C4 analysis included 4 reports detailing issues with a risk rating of LOW severity or non-critical. There were also 2 reports recommending gas optimizations.

All of the issues presented here are linked back to their original finding.

# Scope

The code under review can be found within the [C4 Thruster audit repository](https://github.com/code-423n4/2024-02-thruster), and is composed of 12 smart contracts written in the Solidity programming language and includes 1,554 lines of Solidity code.

In addition to the known issues identified by the project team, an [Automated Findings report](https://github.com/code-423n4/2024-02-thruster?tab=readme-ov-file#automated-findings--publicly-known-issues) was generated using the [4naly3er bot](https://github.com/Picodes/4naly3er) and all findings therein were classified as out of scope.

# Severity Criteria

C4 assesses the severity of disclosed vulnerabilities based on three primary risk categories: high, medium, and low/non-critical.

High-level considerations for vulnerabilities span the following key areas when conducting assessments:

- Malicious Input Handling
- Escalation of privileges
- Arithmetic
- Gas use

For more information regarding the severity criteria referenced throughout the submission review process, please refer to the documentation provided on [the C4 website](https://code4rena.com), specifically our section on [Severity Categorization](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization).

# Medium Risk Findings (5)
## [[M-01] Tickets can be entered after prizes for current round have partially been distributed](https://github.com/code-423n4/2024-02-thruster-findings/issues/28)
*Submitted by [EV\_om](https://github.com/code-423n4/2024-02-thruster-findings/issues/28), also found by 0xDING99YA ([1](https://github.com/code-423n4/2024-02-thruster-findings/issues/33), [2](https://github.com/code-423n4/2024-02-thruster-findings/issues/20)), [oakcobalt](https://github.com/code-423n4/2024-02-thruster-findings/issues/17), and [rvierdiiev](https://github.com/code-423n4/2024-02-thruster-findings/issues/6)*

The `ThrusterTreasure` contract is designed to facilitate a lottery game where users can enter tickets to win prizes based on entropy. The contract includes mechanisms for entering tickets into rounds (`enterTickets()`), setting prizes for rounds (`setPrize()`), and claiming prizes (`claimPrizesForRound()`). A critical aspect of the game's integrity is ensuring each ticket has an equal chance to win every prize.

However, there is a significant flaw in `enterTickets()`. The function checks if winning tickets for the prize index 0 have been set by verifying that `winningTickets[currentRound_][0].length == 0`. This check is intended to prevent users from entering tickets after prizes have begun to be distributed, but it does not account for prizes with higher indices that may already have been distributed. As a result, users can still enter tickets after some prizes have been distributed, but these late-entered tickets will not have a chance to win the already distributed prizes:
[ThrusterTreasure.sol#L83-L96](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L83-L96)

```solidity
function enterTickets(uint256 _amount, bytes32[] calldata _proof) external {
    ...
    require(winningTickets[currentRound_][0].length == 0, "ET");
    ...
}
```

### Proof of Concept

1.  The contract owner sets up a new round with multiple prizes.
2.  User A enters tickets early in the round.
3.  The contract owner distributes prizes for index 1.
4.  User B enters tickets into the round.
5.  Due to the flawed logic in `enterTickets()`, User B's tickets are accepted, even though the prizes for indices 1 and above have already been distributed. User B's tickets, therefore, have no chance of winning those prizes and are worth less than user A's, but the system incorrectly allows their participation for the undistributed prize at index 0.

### Recommended Mitigation Steps

Freeze ticket entry for the current round once any prize has been set.

**[jooleseth (Thruster) confirmed, but disagreed with severity and commented](https://github.com/code-423n4/2024-02-thruster-findings/issues/28#issuecomment-1963623821):**
 > This is an issue reported in a few other Medium's as well
> 
> - Rewards should be set atomically in the same transaction call by an admin script. The expectation is that if the 0 index is set, then all should be set for the round, hence the check for the zero index.
> 
> We also use this zero index check in the `claimPrizesForRound` call.
> 
> I would consider this a Quality Assurance to improve the require check, or Medium at most, as reported by [issue 17](https://github.com/code-423n4/2024-02-thruster-findings/issues/17).

**[0xleastwood (judge) decreased severity to Low/Non-Critical and commented](https://github.com/code-423n4/2024-02-thruster-findings/issues/28#issuecomment-1998952936):**
 > It seems to me that users would have to intentionally be negligible and enter tickets into a round that they are not eligible for. Downgrading to QA.

**[EV\_om (warden) commented](https://github.com/code-423n4/2024-02-thruster-findings/issues/28#issuecomment-2001925516):**
 > @0xleastwood - all users are eligible for prizes as long as they have valid tickets. The user cannot know that prizes have already been set - even if they checked before entering their tickets, an owner call to [`setWinningTickets()`](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L269) may end up being included in a block before their transaction.
> 
> Setting rewards atomically via a script addresses the issue, but it is an OOS mitigation. Considering that setting them non-atomically in both ascending and descending order is problematic and that this was not documented, Medium severity seems reasonable.

**[0xleastwood (judge) increased severity to Medium and commented](https://github.com/code-423n4/2024-02-thruster-findings/issues/28#issuecomment-2005544628):**
 > Understandably, this issue would not be mitigated by having all prizes set at the same time because any tickets entered prior would not be eligible for any reward. There needs to be some clear distinction at which a round ends and when prizes are distributed to prevent this from happening. Would typically class front-running issues as QA but this leads to users spending funds with no expected return.



***

## [[M-02] `claimPrizesForRound` transfers the entire amount deposited for a prize regardless of the number of winners](https://github.com/code-423n4/2024-02-thruster-findings/issues/27)
*Submitted by [EV\_om](https://github.com/code-423n4/2024-02-thruster-findings/issues/27), also found by [0xDING99YA](https://github.com/code-423n4/2024-02-thruster-findings/issues/18) and [oakcobalt](https://github.com/code-423n4/2024-02-thruster-findings/issues/15)*

`claimPrizesForRound()` transfers the entire amount of a prize to a winner without considering the total number of winners for that prize.

The prize for a given round and prize index can be set by calling the `setPrize()` function, which pulls the amounts from the caller (the owner) and stores the prize data in the `prizes` array:

[ThrusterTreasure.sol#L163-L184](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L163-L184)

```solidity
function setPrize(uint256 _round, uint64 _prizeIndex, uint256 _amountWETH, uint256 _amountUSDB, uint64 _numWinners)
	external
	onlyOwner
{
	require(_round >= currentRound, "ICR");
	require(_prizeIndex < maxPrizeCount, "IPC");
	depositPrize(msg.sender, _amountWETH, _amountUSDB);
	prizes[_round][_prizeIndex] = Prize(_amountWETH, _amountUSDB, _numWinners, _prizeIndex, uint64(_round));
}

function depositPrize(address _from, uint256 _amountWETH, uint256 _amountUSDB) internal {
	WETH.transferFrom(_from, address(this), _amountWETH);
	USDB.transferFrom(_from, address(this), _amountUSDB);
	emit DepositedPrizes(_amountWETH, _amountUSDB);
}
```

However, the `claimPrizesForRound()` function always transfers the full prize amounts to the first caller, regardless of the number of winners for the prize. Once the prize for a specific index is claimed, other winners of that prize cannot claim their share (or winners of other prizes may end up not being able to claim theirs), effectively being denied their winnings:

[ThrusterTreasure.sol#L102-L134](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L102-L134)

```solidity
function claimPrizesForRound(uint256 roundToClaim) external {
	...
	for (uint256 i = 0; i < maxPrizeCount_; i++) {
		Prize memory prize = prizes[roundToClaim][i];
		uint256[] memory winningTicketsRoundPrize = winningTickets[roundToClaim][i];
		for (uint256 j = 0; j < winningTicketsRoundPrize.length; j++) {
			uint256 winningTicket = winningTicketsRoundPrize[j];
			if (round.ticketStart <= winningTicket && round.ticketEnd > winningTicket) {
				_claimPrize(prize, msg.sender, winningTicket);
			}
		}
	}
	...
}

function _claimPrize(Prize memory _prize, address _receiver, uint256 _winningTicket) internal {
	uint256 amountETH = _prize.amountWETH;
	uint256 amountUSDB = _prize.amountUSDB;
	WETH.transfer(_receiver, amountETH);
	USDB.transfer(_receiver, amountUSDB);
	emit ClaimedPrize(_receiver, _prize.round, _prize.prizeIndex, amountETH, amountUSDB, _winningTicket);
}
```

This approach can lead to scenarios where the amount available to be distributed among prize winners is less than that represented by the prizes stored in the `prizes` array.

This is considered medium severity because:

*   `claimPrizesForRound()` will revert if there aren't enough funds to transfer the prize to the winner, altering the user
*   the owner can mitigate this by simply "refilling" the prize as many times as needed

### Proof of Concept

1.  A prize is set with a certain amount of WETH and USDB for a specific round and prize index, intended for multiple winners.
2.  Multiple users enter the round with tickets that end up winning this prize.
3.  The first user to call `claimPrizesForRound()` for this round and prize index successfully claims the entire prize amount.
4.  Subsequent winners attempting to claim their share of the prize for the same round and prize index find that they cannot, as the prize has already been fully distributed to the first caller.

### Recommended Mitigation Steps

It is unclear whether the amounts passed to `setPrize()` are meant to be distributed among all winners of the given prize or to be paid out to each winner, but the cleaner approach would be the latter. In that case, the amount pulled from the owner can simply be scaled by the number of winners:

```solidity
function depositPrize(address _from, uint64 _numWinners, uint256 _amountWETH, uint256 _amountUSDB) internal {
	WETH.transferFrom(_from, address(this), _amountWETH * _numWinners);
	USDB.transferFrom(_from, address(this), _amountUSDB * _numWinners);
	emit DepositedPrizes(_numWinners, _amountWETH, _amountUSDB);
}
```

**[jooleseth (Thruster) confirmed and commented](https://github.com/code-423n4/2024-02-thruster-findings/issues/27#issuecomment-1963628994):**
 > I agree with the Warden's evaluation.



***

## [[M-03] Dynamic modification of `maxPrizeCount` affects prize claims](https://github.com/code-423n4/2024-02-thruster-findings/issues/25)
*Submitted by [EV\_om](https://github.com/code-423n4/2024-02-thruster-findings/issues/25), also found by [0xDING99YA](https://github.com/code-423n4/2024-02-thruster-findings/issues/31) and [rvierdiiev](https://github.com/code-423n4/2024-02-thruster-findings/issues/10)*

The `ThrusterTreasure` contract is designed to manage rounds of a lottery game, where participants can enter tickets and claim prizes based on random draws. The contract includes a variable `maxPrizeCount` which dictates the maximum number of prizes that can be set for any given round. This variable can be modified by the contract owner at any time through the `setMaxPrizeCount(uint256 _maxPrizeCount)` function:

[ThrusterTreasure.sol#L139-L142](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L139-L142)

```solidity
    function setMaxPrizeCount(uint256 _maxPrizeCount) external onlyOwner {
        maxPrizeCount = _maxPrizeCount;
        emit SetMaxPrizeCount(_maxPrizeCount);
    }
```

The issue arises when `maxPrizeCount` is decreased after prizes for a round have been set but before they have been claimed. Since the `claimPrizesForRound(uint256 roundToClaim)` function iterates over prize indices up to `maxPrizeCount`, reducing this count means that winners of prizes with indices higher than the new `maxPrizeCount` will be unable to claim their winnings:

[ThrusterTreasure.sol#L102-L120](https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L102-L120)

```solidity
    function claimPrizesForRound(uint256 roundToClaim) external {
        ...
        
        uint256 maxPrizeCount_ = maxPrizeCount;
        for (uint256 i = 0; i < maxPrizeCount_; i++) {
            [claim prize]
        }
        entered[msg.sender][roundToClaim] = Round(0, 0, roundToClaim); // Clear user's tickets for the round
        emit CheckedPrizesForRound(msg.sender, roundToClaim);
    }
```

This could lead to a scenario where legitimate winners are denied their prizes due to a change in contract state that is unrelated to the rules of the game or their actions. Moreover, since calling `claimPrizesForRound()` clears the user's entries for the round, reverting `maxPrizeCount` to its previous state does not allow them to claim the remaining tickets. This means they will effectively never be able to claim their prize.

### Proof of Concept

1.  The contract owner sets `maxPrizeCount` to 5 and configures five prizes for a given round.
2.  Users participate in the round, and the round concludes with winners determined for all five prizes.
3.  The contract owner reduces `maxPrizeCount` to 3 for the next round.
4.  Winners of prizes 4 and 5 attempt to claim their prizes but are unable to do so because the `claimPrizesForRound(uint256 roundToClaim)` function now iterates only up to the new `maxPrizeCount` of 3.

### Recommended Mitigation Steps

To address this issue, implementing a checkpoint pattern for the `maxPrizeCount` variable is suggested. This method involves tracking changes to `maxPrizeCount` with checkpoints that record the value and the round number when the change occurs.

A possible implementation could look like this:

```solidity
// Add a struct to store checkpoints for maxPrizeCount changes
struct MaxPrizeCountCheckpoint {
    uint256 round;
    uint256 maxPrizeCount;
}

// Use an array to keep track of all checkpoints
MaxPrizeCountCheckpoint[] public maxPrizeCountCheckpoints;

constructor(
	...
) Ownable(msg.sender) {
	maxPrizeCountCheckpoints.push(
		MaxPrizeCountCheckpoint(0, _maxPrizeCount)
	);
    ...
}

// Modify setMaxPrizeCount to push a new checkpoint to the array
function setMaxPrizeCount(uint256 _maxPrizeCount) external onlyOwner {
	require(_maxPrizeCount != getMaxPrizeCountForRound(currentRound), "same value")
    maxPrizeCountCheckpoints.push(
		MaxPrizeCountCheckpoint(currentRound, _maxPrizeCount)
    );
    emit SetMaxPrizeCount(_maxPrizeCount);
}

// Helper function to get the maxPrizeCount for a given round
// Assumes more recent rounds will be queried more often
function getMaxPrizeCountForRound(uint256 _round) public view returns (uint256) {
    uint256 length = maxPrizeCountCheckpoints.length;
    for (uint256 i = length; i > 0; i--) {
        MaxPrizeCountCheckpoint storage checkpoint = maxPrizeCountCheckpoints[i - 1];
        if (checkpoint.round <= _round) {
            return checkpoint.maxPrizeCount;
        }
    }
    return 0;
}

// Disallow setting prizes for future rounds since the maxPrizeCount could change
function setPrize(uint64 _prizeIndex, uint256 _amountWETH, uint256 _amountUSDB, uint64 _numWinners) external onlyOwner {
    uint256 maxPrizeCount = getMaxPrizeCountForRound(currentRound);
    require(_prizeIndex < maxPrizeCount, "IPC");
    ...
}

function claimPrizesForRound(uint256 roundToClaim) external {
    uint256 maxPrizeCount = getMaxPrizeCountForRound(roundToClaim);
    ...
}
```

This change ensures that each round's prize structure is fixed upon the round's creation, preventing post-hoc alterations that could negatively impact participants. Note that this implementation still requires attention is paid to not calling `setMaxPrizeCount()` for a given round if prizes have already been set for higher indices.

**[jooleseth (Thruster) commented](https://github.com/code-423n4/2024-02-thruster-findings/issues/25#issuecomment-1963634534):**
 > I would consider this QA to ensure to add a require check that `maxPrizeCount` cannot be decreased as that is the intention.

**[EV\_om (warden) commented](https://github.com/code-423n4/2024-02-thruster-findings/issues/25#issuecomment-1964892925):**
 > That's indeed a much simpler fix! 
> 
> I would still say Medium severity is appropriate seeing as that could not have been inferred from the audit scope and given the potential impact, but either way I'll accept the judge's decision.

**[jooleseth (Thruster) confirmed](https://github.com/code-423n4/2024-02-thruster-findings/issues/25#issuecomment-1984513989)**



***

## [[M-04] Incorrect gas claiming logic in `ThrusterPoolDeployer`](https://github.com/code-423n4/2024-02-thruster-findings/issues/24)
*Submitted by [EV\_om](https://github.com/code-423n4/2024-02-thruster-findings/issues/24), also found by [oakcobalt](https://github.com/code-423n4/2024-02-thruster-findings/issues/14) and [rvierdiiev](https://github.com/code-423n4/2024-02-thruster-findings/issues/4)*

From the audit documentation:

> All contracts that use gas should comply with the Blast gas claim logic.

However, the `ThrusterPoolDeployer` contains a flaw in the implementation of `claimGas()` which will prevent it from ever claiming the gas it induces. The function attempts to claim gas for the zero address (`address(0)`) instead of the deployer's own address (`address(this)`):

```solidity
    function claimGas(address _recipient) external override onlyFactory returns (uint256 amount) {
        amount = IBlast(BLAST).claimMaxGas(address(0), _recipient);
    }
```

This misconfiguration prevents the `ThrusterPoolDeployer` from reclaiming any gas, as the `IBlast.claimMaxGas()` call will always fail when provided with the zero address.

### Proof of Concept

<https://github.com/blast-io/blast/blob/master/blast-optimism/packages/contracts-bedrock/src/L2/Blast.sol#L274-L283>

```solidity
/**
 * @notice Claims gas available to be claimed at max claim rate for a specific contract. Called by an authorized user
 * @param contractAddress The address of the contract for which maximum gas is to be claimed
 * @param recipientOfGas The address of the recipient of the gas
 * @return The amount of gas that was claimed
 */
function claimMaxGas(address contractAddress, address recipientOfGas) external returns (uint256) {
	require(isAuthorized(contractAddress), "Not allowed to claim max gas");
	return IGas(GAS_CONTRACT).claimMax(contractAddress, recipientOfGas);
}
```

### Recommended Mitigation Steps

Use `address(this)` rather than `0`.

**[jooleseth (Thruster) confirmed and commented](https://github.com/code-423n4/2024-02-thruster-findings/issues/24#issuecomment-1963636092):**
 > I agree, we caught this issue a few days ago too. Blast had initially in their docs specified it should be address(0) instead of address(this) for gas claiming and we missed changing this in the audit commit freeze. Will agree to a Medium Risk bug for this, as it doesn't affect any user funds.



***

## [[M-05] `ThrusterFactory.setYieldCut` should claim fees for all pools before](https://github.com/code-423n4/2024-02-thruster-findings/issues/12)
*Submitted by [rvierdiiev](https://github.com/code-423n4/2024-02-thruster-findings/issues/12)*

Thruster have introduced ability [to change yield cut](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-cfmm/contracts/ThrusterFactory.sol#L75-L80) for their uniswap v2 like pools.

The yield is charged only, when some LP [manages their position](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-cfmm/contracts/ThrusterPair.sol#L131).

This means that in case if yield cut will be changed for a pool, then protocol fee should be minted for previous period using old yield cut, otherwise the yield cut will be incorrect, especially this is important for pools where liquidity is not added/removed often.

### Impact

Yield can be collected with wrong proportion.

### Tools Used

VsCode

### Recommended Mitigation Steps

I guess protocol just needs to acknowledge the issue as it will be not possible (not worthy) to implement such mechanism.

**[jooleseth (Thruster) acknowledged and commented](https://github.com/code-423n4/2024-02-thruster-findings/issues/12#issuecomment-1963667483):**
 > We acknowledge that this situation is possible, but the effects and consequences of this are very minimal in reality as LPs update often and LPs are made aware of changes to fees with sufficient time.



***

# Low Risk and Non-Critical Issues

For this audit, 4 reports were submitted by wardens detailing low risk and non-critical issues. The [report highlighted below](https://github.com/code-423n4/2024-02-thruster-findings/issues/16) by **oakcobalt** received the top score from the judge.

*The following wardens also submitted reports: [EV\_om](https://github.com/code-423n4/2024-02-thruster-findings/issues/23), [rvierdiiev](https://github.com/code-423n4/2024-02-thruster-findings/issues/2), and [0xDING99YA](https://github.com/code-423n4/2024-02-thruster-findings/issues/21).*

## [L-01] Incorrect comment - Fee should be between 1/6 and 1/2 of the growth in sqrt(k)

### Instances(1)

ThrusterPair.sol allows fee/yieldCut to be adjusted between 1/6 and 1/2 (15\% to 50\%) of growth in sqrt(k). And the yieldCut is calculated in `_mintYieldCut()`. 

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

### Recommendations

Change the comment to reflect the range `1/6 and 1/2`. 

## [L-02] Unnecessary inheritance of `IThrusterERC20` in `TrusterPair.sol`

### Instances(1)

Based on readme, a design decision is made to flatten ERC20 function implementations in TrustedPair.sol. As a result, ERC20 methods are included in both TrusterPair.sol and ITrustedPair.sol. 

There is no need to add to TrustedPair.sol's inheritance chain with addition of IThrusterERC20.sol which is a subset of ITrusterPair.sol.

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

### Recommendations

Remove IThrusterERC20 inheritance.

## [L-03] ThrusterPair may have an outdated manager address due to a vulnerable manager update mechanism

### Instances(1)

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

### Recommendations

In ThrusterYield.sol, instead of storing an isolated `manager` address, in `onlyManager()`, simply call ThrusterFactory for the most current `yieldTo` address.

## [L-04] `claimPrizesForRound()` doesn't satisfy check-effect-interaction patterns

### Instances(1)

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

### Recommendations

Update `entered[msg.sender][roundToClaim]` before calling `_claimPrize()`.


## [L-05] Unnecessary code in `ThrusterTreasure::requestRandomNumberMany` and `ThrusterTreasure::requestRandomNumber`

### Instances(2)

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

### Recommendations

Remove `requestedRandomNumber[sequenceNumber] = msg.sender;`.

## [L-06] Unnecessary implementations for ETH rewards - Pools can’t receive ETH in balance and will never be eligible for ETH rewards

### Instances(2)

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

### Recommendations

Remove both the ETH rewards configuration and the claiming code for ETH rewards for contracts not able to receive ETH.

## [L-07] A user can withdraw prizes immediately after winning tickets drawn, bypassing the intended 'next draw period' delay stated by Doc

### Instances(1)

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

### Recommendations

If the intention is to unify a prize-winning window for a given round to 'next draw period starts'(as stated in doc), then add a check in `claimPrizesForRound()` to ensure a minimal delay timestamp.

## [L-08] `reqeustRandomNumberMany()` has incorrect array initialization

### Instances(1)

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

### Recommendations

Initiate array `seqNums` first with length data before updating its value.

## [L-09] User's prize might be forever locked if the setting winning tickets transaction is settled on the end timestamp of a round

### Instances(1)

In ThrusterTreasure.sol, there is no on-chain check to ensure owner's `setWinningTickets()` transaction will not settle on the max end timestamp of a round `roundStart[_round] + MAX_ROUND_TIME`. 

If `setWinningTickets()` happens to settle on `roundStart[_round] + MAX_ROUND_TIME`, `setWinningTickets()` transaction will still succeed. But in this edge case, winners are likely not to be able to claim their prizes for the round, because `claimPrizesForRound()` contains the same check for `roundStart[roundToClaim] + MAX_ROUND_TIME >= block.timestamp`. If user's claiming tx settles after `roundStart[roundToClaim] + MAX_ROUND_TIME`, claiming tx will revert, and users will not be able to claim their prizes.
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

### Recommendations

Although the owner can decide on the timing to set winning tickets, it will be good to ensure on-chain that there will always be a grace period between `setWinningTickets()` and `claimPrizesForRound()`. So do not use the same end timestamp check for both. 


## [L-10]  A hardcoded minimal liquidity might be insufficient for assets with higher token decimals

### Instances(1)

In ThrusterPair.sol, `MINIMUM_LIQUIDITY` is hardcoded as 1000. This minimum liquidity will be locked at the first deposit as a permanent supply. And also used to prevent initial dust deposit. 

However, it should be noted that 1000 might not be suitable for assets with higher decimals. Dust value first deposit might allow initial pool reserve ratio manipulation at a cheap price, and the malicious first depositor can profit from the initial unbalanced reserve ratio through arbitrage.

Given pool deployment is permissionless, it's better to allow a range of configurations of the minimum liquidity value at ThrusterPair initialization, such that the pool deployer can configure reasonable minimum liquidity to prevent cheap initial price manipulation with vulnerable assets.

```solidity
//thruster-protocol/thruster-cfmm/contracts/ThrusterPair.sol
    uint256 public constant MINIMUM_LIQUIDITY = 10 ** 3;
```
(https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-cfmm/contracts/ThrusterPair.sol#L19)

### Recommendations

Consider allowing configuration of minimum liquidity value in `initialize()`.

## [L-11] Thruster pairs are vulnerable to first-depositor price manipulation

### Instances(1)

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

### Recommendations

Might consider allowing pool deployers to suggest a reasonable reserve ratio at the pair initialization process `initialize()`. Although this will not prevent pool deployer from being malicious, this can mitigate certain price manipulation attempts.

## [L-12] Unnecessary code - `_winningTickets` length will be the same as numWinners

### Instances(1)

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

### Recommendations

Remove the last require statement.

## [L-13] Users who deposit to a ThrusterPool with a rebasing token that allows negative rebasing might incur losses

ThrusterPool.sol is based on UniswapV3 logic which allows concentrated liquidity. A liquidity provider will gain trading fees once their position becomes active. 

According to UniswapV3 [doc](https://docs.uniswap.org/concepts/protocol/integration-issues), a pool that has a potentially negative rebasing tokens will cause liquidity providers to incure losses.

>Rebasing tokens will succeed in pool creation and swapping, but liquidity providers will bear the loss of a negative rebase when their position becomes active, with no way to recover the loss.

### Recommendations

Consider disallowing pools with negative rebasing tokens. 

## [L-14] Use `_safeMint` instead of `_mint` in NonfungiblePositionManager.sol
In NonfungiblePositionManager::mint(), `_mint()` is used instead of `_safeMint()`. 

NonfungiblePositionManager inherits openzeppelin's ERC721.sol which provides both `_mint()` and `_safeMint()`. `_mint()` is [discouraged](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/d4d8d2ed9798cc3383912a23b5e8d5cb602f7d4b/contracts/token/ERC721/ERC721.sol#L271) in favor of `_safeMint()` which ensures that the recipient is either an EOA or implements IERC721Receiver. 

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

### Recommendations

Use `_safeMint()` instead.

**[jooleseth (Thruster) acknowledged](https://github.com/code-423n4/2024-02-thruster-findings/issues/16#issuecomment-1984507461)**

**[0xleastwood (judge) commented](https://github.com/code-423n4/2024-02-thruster-findings/issues/16#issuecomment-2005553538):**
 > All seem valid and useful.



***

# Gas Optimizations

For this audit, 2 reports were submitted by wardens detailing gas optimizations. The [report highlighted below](https://github.com/code-423n4/2024-02-thruster-findings/issues/22) by **oakcobalt** received the top score from the judge.

*The following warden also submitted a report: [rvierdiiev](https://github.com/code-423n4/2024-02-thruster-findings/issues/5).*

## [G-01] Emitting two duplicated events might be unnecessary and wasteful

### Instances(2)

**Total Gas Saved: 2250**

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

### Recommendations

Save users's runtime gas cost by only emitting swap event in the factory contract.

## [G-02] Wasteful runtime gas cost due to maximum for-loop iterations

### Instances(1)

**Total Gas Saved: Various**

In ThrusterTreasure.sol::claimPrizesForRound(), for-loop will always run maximum iterations(`maxPrizeCount`) regardless of the actual maximum prize index for a given round.

For example, when `maxPrizeCount` is set to 4 and there is only one prizeIdx set for a round, the for-loop `maxPrizeCount_` will still run 4 times. This will cost extra gas for users. 

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

### Recommendations

Consider for each round, set a round-specific max prize index to limit the for-loop iterations instead of using a global max number.

## [G-03] Events should be emitted outside of loops

### Instances(1)

**Total Gas Saved: Various**
  
Emitting an event has an overhead of 375 gas, which will be incurred on every iteration of the loop. It is cheaper to `emit` only once after the loop has finished.

In `ThrusterTreasure.sol::requestRandomNumberMany()`, `RandomNumberRequest` will be emitted in every iteration in the for-loop. 

```solidity
//thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol
    function requestRandomNumberMany(
        bytes32[] calldata userCommitments
    ) external payable onlyOwner returns (uint64[] memory seqNums) {
...
        for (uint256 i = 0; i < userCommitments.length; i++) {
...
            emit RandomNumberRequest(sequenceNumber, userCommitments[i]);
        }
```
(https://github.com/code-423n4/2024-02-thruster/blob/3896779349f90a44b46f2646094cb34fffd7f66e/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L230)

### Recommendations

Consider emit the array of seqNum and userCommitments outside of the for-loop.

**[jooleseth (Thruster) acknowledged](https://github.com/code-423n4/2024-02-thruster-findings/issues/22#issuecomment-1984510310)**



***

# Audit Analysis

For this audit, 2 analysis reports were submitted by wardens. An analysis report examines the codebase as a whole, providing observations and advice on such topics as architecture, mechanism, or approach. The [report highlighted below](https://github.com/code-423n4/2024-02-thruster-findings/issues/29) by **oakcobalt** received the top score from the judge.

*The following warden also submitted a report: [rvierdiiev](https://github.com/code-423n4/2024-02-thruster-findings/issues/9).*

### Summary

Thruster is a decentralized exchange (DEX) within the Blast ecosystem, prioritizing decentralized finance (DeFi) and catering to traders looking for high-risk, high-reward opportunities. 

Notably, Thruster takes advantage of Blast’s native yield rewards systems to give back the rewards to users through a lottery drawing feature. 

**Existing Standards:**

- The protocol adheres to conventional Solidity and Ethereum practices, primarily utilizing the ERC20 and ERC721 standards, along with the openzepplin's access control patterns.
- Additionally, ERC20Permit feature is integrated with ThrusterPair.
- EIP-712: Typed structured data hashing and signing in ThrusterPair

It's worth noting that the protocol relies on off-chain + on-chain multi-step random number generation service by Pyth Entrophy. 

### 1- Approach:

- Scope: Due to Thruster containing forks of UniswapV2 and UniswapV3, scope is separated by forked code review and custom/unique code review.
- Roles: Then role specific flows are focused which include various defi user flows (trading, Lping, lottery entry and claiming, etc) and flows of privileged roles.
- Blast yield: Flows of various blast native yield configuration and claiming.

### 2- Centralization Risks:

Here's an analysis of potential centralization risks in the provided contracts:

**Pool contracts:(ThrusterPair, ThrusterPool):**

- Low centralization risk due to permissionless deployment.
- Manager address can claim all pool-generated Blast yield rewards to any address they choose. User-generated yield rewards might not be distributed to the lottery pool as intended.

**Factory/deployer contracts:**

- ThrusterFactory.sol: YieldToCut fee percentage can be adjusted at any time from 16\% to 50\%.

**ThrusterTreasure.sol:**

- The winning tickets drawing process is largely off-chain due to Entropy’s multi-step process which requires contract owner interaction with a web server(Entropy provider). The owner can theoretically choose which combinations of `sequenceNumbers` and `userRandoms` to send during `setWinningTickets()`. Or decide which combination to withhold so that a certain final random number(winning ticket) is not revealed as a winning ticket.
- The owner can withdraw Prize funds freely from ThrusterTreasure at any time through `retrieveTOkens()`. Hypothetically even if a user wins a lottery, there can be no funds to distribute if the owner transfers the prize funds out before user claiming.

### 3- Systemic Risks:

**Price slippage and first depositor price manipulation:**

ThrusterPair.sol is vulnerable to first depositor price manipulation due to any reserve ratio being accepted for the first deposit. 

Due to `MIN_LIQUIDITY` only specifying a minimum dust amount (or even lower than dust amount if the reserve decimal is higher than 18 decimals), the initial price manipulation is still very low cost and potentially profitable.

In addition, currently deployed pool/pair contracts are on Blast testnet, which suggests the initial Blast mainnet launch which will face a period of new pool deployment with low liquidity, which increases the risk of price manipulation.

**Reliability of off-chain process:**

The lottery process requires a random generation service provided by Pyth network which is a multi-step process that requires an off-chain process.

This off-chain and on-chain interface when various intermediate stages of random number values (`sequenceNumber`, `userRandoms`,`userCommitment`, etc) is prone to mistakes.

And also Entropy provider URL request might fail which will DOS the random number generation process.

**Counterparty Risks:**

The reliability of both Entropy and Entropy provider service increases the counterparty risks.

### 4- Mechanism Review:

**Isolated Manager Address: ThrusterYield.sol**

ThrusterYield.sol is inherited by each ThrusterPair and stores a manager address with Blast yield claiming privilege.  Due to this manager address is isolated in each pair contract, the process of updating this manager address will be gas-intensive and cumbersome in the future.

Recommendation:  Refactor the `onlyManager` in ThrusterYield to actively call ThrusterFactory to the active setter contract, to avoid gas intensive individual manager updates

```solidity
    function setManager(address _manager) external onlyManager {
        manager = _manager;
    }
}

    modifier onlyManager() {
        require(msg.sender == manager, "FORBIDDEN");
        _;
    }
```

**Isolated Blast Yield Claiming: ThrusterPool.sol:**

Claim yield functions are isolated in decentralized pool/pair contracts. This means claiming yield generated in multiple pool/pair contracts that contain a Blast reward token can become cumbersome and gas-intensive.

Recommendation: Since USDB and WETHB are relevant reward tokens, consider in factory contracts, create a registry for pool/pairs that contain reward tokens.

**Rebasing tokens in UniswapV2 and UniswapV3: ThrusterPair.sol:**

Since Thruster pool logic forks UniswapV2/V3 the same risks associated with a pool with a rebasing token asset apply. 

For ThrusterPair, a positive rebasing token will generate tokens that can be taken by anyone.

For ThrusterPool, a negative rebasing token might create a loss for Lps. 

See Uniswap doc [v2](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/common-errors#positive-rebasing-tokens) and [v3](https://docs.uniswap.org/concepts/protocol/integration-issues) for more details. 

### Conclusion:

Thruster stands out as a decentralized exchange (DEX) within the Blast ecosystem, emphasizing decentralized finance (DeFi) and catering to traders seeking high-risk, high-reward opportunities. Leveraging Blast's native yield rewards systems, Thruster incorporates innovative features such as a lottery drawing mechanism to redistribute rewards to users. 

The protocol adheres to established standards like ERC20 and ERC721, along with incorporating EIP-712 for structured data hashing and signing. 

However, several centralization risks are identified, particularly regarding the management of pool contracts and the ThrusterTreasure contract's lottery drawing process. Additionally, systemic risks such as price manipulation and reliance on off-chain processes pose challenges to the protocol's reliability. Mechanism reviews highlight the need for isolated manager addresses and blast yield claiming, as well as considerations for rebasing tokens in UniswapV2 and UniswapV3. Moving forward, addressing these identified risks through proactive measures like contract refactoring and enhancing registry systems will be crucial for ensuring the security and stability of the Thruster protocol.

**Time spent:**<br>
40 hours



***


# Disclosures

C4 is an open organization governed by participants in the community.

C4 audits incentivize the discovery of exploits, vulnerabilities, and bugs in smart contracts. Security researchers are rewarded at an increasing rate for finding higher-risk issues. Audit submissions are judged by a knowledgeable security researcher and solidity developer and disclosed to sponsoring developers. C4 does not conduct formal verification regarding the provided code but instead provides final verification.

C4 does not provide any guarantee or warranty regarding the security of this project. All smart contract software should be used at the sole risk and responsibility of users.

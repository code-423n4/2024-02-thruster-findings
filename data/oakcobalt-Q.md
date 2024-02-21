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



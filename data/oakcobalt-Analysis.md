## Summary

Thruster is a decentralized exchange (DEX) within the Blast ecosystem, prioritizing decentralized finance (DeFi) and catering to traders looking for high-risk, high-reward opportunities. 

Notably, Thruster takes advantage of Blast’s native yield rewards systems to give back the rewards to users through a lottery drawing feature. 

**Existing Standards:**

- The protocol adheres to conventional Solidity and Ethereum practices, primarily utilizing the ERC20 and ERC721 standards, along with the openzepplin's access control patterns.
- Additionally, ERC20Permit feature is integrated with ThrusterPair.
- EIP-712: Typed structured data hashing and signing in ThrusterPair

It's worth noting that the protocol relies on off-chain + on-chain multi-step random number generation service by Pyth Entrophy. 

## 1- Approach:

- Scope: Due to thruster contains forks of UniswapV2 and UniswapV3, scope is separated by forked code review and custom/unique code review.
- Roles: Then role specific flows are focused which include various defi user flows (trading, Lping, lottery entry and claiming, etc) and flows of privileged roles.
- Blast yield: Flows of various blast native yield configuration and claiming

## 2- Centralization Risks:

Here's an analysis of potential centralization risks in the provided contracts:

### Pool contracts:(ThrusterPair, ThrusterPool)

- Low centralization risk due to permissionless deployment.
- Manager address can claim all pool-generated Blast yield rewards to any address they choose. User-generated yield rewards might not be distributed to the lottery pool as intended.

### Factory/deployer contracts:

- ThrusterFactory.sol: YieldToCut fee percentage can be adjusted at any time from 16% to 50%

### ThrusterTreasure.sol:

- The winning tickets drawing process is largely off-chain due to Entropy’s multi-step process which requires contract owner interaction with a web server(Entropy provider). The owner can theoretically choose which combinations of `sequenceNumbers` and `userRandoms` to send during `setWinningTickets()`. Or decide which combination to withhold so that a certain final random number(winning ticket) is not revealed as a winning ticket.
- The owner can withdraw Prize funds freely from ThrusterTreasure at any time through `retrieveTOkens()`. Hypothetically even if a user wins a lottery, there can be no funds to distribute if the owner transfers the prize funds out before user claiming.

## 3- Systemic Risks:

### Price slippage and first depositor price manipulation

ThrusterPair.sol is vulnerable to first depositor price manipulation due to any reserve ratio being accepted for the first deposit. 

Due to `MIN_LIQUIDITY` only specifying a minimum dust amount (or even lower than dust amount if the reserve decimal is higher than 18 decimals), the initial price manipulation is still very low cost and potentially profitable.

In addition, currently deployed pool/pair contracts are on Blast testnet, which suggests the initial Blast mainnet launch which will face a period of new pool deployment with low liquidity, which increases the risk of price manipulation.

### Reliability of off-chain process:

The lottery process requires a random generation service provided by Pyth network which is a multi-step process that requires an off-chain process.

This off-chain and on-chain interface when various intermediate stages of random number values (`sequenceNumber`, `userRandoms`,`userCommitment`, etc) is prone to mistakes.

And also Entropy provider URL request might fail which will DOS the random number generation process.

### Counterparty Risks:

The reliability of both Entropy and Entropy provider service increases the counterparty risks.

## 4- Mechanism Review:

### Isolated Manager Address

**ThrusterYield.sol:**

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

### Isolated Blast Yield Claiming

**ThrusterYield.sol:**

**ThrusterPool.sol:**

Claim yield functions are isolated in decentralized pool/pair contracts. This means claiming yield generated in multiple pool/pair contracts that contain a Blast reward token can become cumbersome and gas-intensive.

Recommendation: Since USDB and WETHB are relevant reward tokens, consider in factory contracts, create a registry for pool/pairs that contain reward tokens.

### Rebasing tokens in UniswapV2 and UniswapV3

**ThrusterPair.sol:**

**ThrusterPool.sol:**

Since Thruster pool logic forks UniswapV2/V3 the same risks associated with a pool with a rebasing token asset apply. 

For ThrusterPair, a positive rebasing token will generate tokens that can be taken by anyone.

For ThrusterPool, a negative rebasing token might create a loss for Lps. 

See Uniswap doc [v2](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/common-errors#positive-rebasing-tokens) and [v3](https://docs.uniswap.org/concepts/protocol/integration-issues) for more details. 

### Conclusion:

Thruster stands out as a decentralized exchange (DEX) within the Blast ecosystem, emphasizing decentralized finance (DeFi) and catering to traders seeking high-risk, high-reward opportunities. Leveraging Blast's native yield rewards systems, Thruster incorporates innovative features such as a lottery drawing mechanism to redistribute rewards to users. 

The protocol adheres to established standards like ERC20 and ERC721, along with incorporating EIP-712 for structured data hashing and signing. 

However, several centralization risks are identified, particularly regarding the management of pool contracts and the ThrusterTreasure contract's lottery drawing process. Additionally, systemic risks such as price manipulation and reliance on off-chain processes pose challenges to the protocol's reliability. Mechanism reviews highlight the need for isolated manager addresses and blast yield claiming, as well as considerations for rebasing tokens in UniswapV2 and UniswapV3. Moving forward, addressing these identified risks through proactive measures like contract refactoring and enhancing registry systems will be crucial for ensuring the security and stability of the Thruster protocol.

### Time spent:
40 hours
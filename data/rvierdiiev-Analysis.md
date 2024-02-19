## Thruster-cfmm
### Overview
Thruster-cfmm is a uniswap v2 fork that will be deployed on Blast chain. As blast provides ability to claim eth, weth, usdb yield and gas fees, protocol implements such ability in their contracts.

### Concerns and Recommendations
- Protocol has provided ability to change LP fee part that goes to the protocol. Uniswap v2 code have hardcoded such protocol fee as 6th part of LP fees, which is 0.05%. Thruster have provided ability to increase it up to 50%. This should be used carefull as can have negative impact on LP providers, which then can left pools.
- Protocol has ability to claim eth, weth, usdb and gas yields, however do not share them among LP providers directly to incentivize them and make yields higher. I would recommend to consider this.

## Thruster-clmm
### Overview
Thruster-clmm is a uniswap v3 fork that will be deployed on Blast chain. As blast provides ability to claim eth, weth, usdb yield and gas fees, protocol implements such ability in their contracts. Almost nothing was changed by thruster protocol, just gauge metering was added.

### Concerns and Recommendations
- Protocol has ability to claim eth, weth, usdb and gas yields, however do not share them among LP providers to incentivize them and make yields higher. I would recommend to consider this.

## Thruster-treasure
### Overview
Thruster-treasure is module that allows protocol to distribute prizes among eligible participants. There were no info about how prizes will be distributed and who will be able to get them, but i think that LP providers can be good candidates and those who do a lot of swaps.

### Concerns and Recommendations
- The design of contract doesn't work as state machine and allows users to interact with contract in the moments, when it should be restricted, for example provide new root, when old round is not finalized or add new tickets, when winner selection started.
- In order to get random number contract uses Pyth entropy, which adds additional integration risks in case if Pyth will face problems.
- Pyth entropy needs a payment to be done with random request. Thus protocol team is responsible to ensure that there is enough funds in the contract to cover those calls and not suspend winner selection.
- Admin of contract has super power to change prizes and to withdraw funds from contract. I think, this should be redesigned.

### Time spent:
16 hours
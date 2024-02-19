## QA-01. Change gas manager, when yieldToSetter is changed
### Description
When `ThrusterFactory` is deployed, then it's gas manager [is set to `yieldToSetter`](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-cfmm/contracts/ThrusterFactory.sol#L35C82-L35C96).

Later it's possible that `yieldToSetter` [is changed](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-cfmm/contracts/ThrusterFactory.sol#L68-L73) for any reason(for example private key leak), but gas manager is not changed and can be then changed by attacker or used to claim gas to own address.

Similarly for all pools factory `yieldToSetter` [is set as gas and yield manager](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-cfmm/contracts/ThrusterPair.sol#L79). In case if `yieldToSetter` is changed in factory, then manager should be changed, however it will be not possible to do because of amount of deployed pools.

### Recommendation
Change gas manager for factory.

## QA-02. DOMAIN_SEPARATOR is cached for the pools
### Description
When pool is constructed, then [DOMAIN_SEPARATOR is calculated](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-cfmm/contracts/ThrusterPair.sol#L85-L92). It uses chainid at the moment of deploy. DOMAIN_SEPARATOR is used for permit functionality.

In case if hardfork will happen that will change chain id, then this functionality will not work as expected anymore.

### Recommendation
In case if chain is not what was cached on deploy, then recalculate DOMAIN_SEPARATOR.

## QA-03. NonfungibleTokenPositionDescriptor doesn't have ability to claim gas
### Description
`NonfungibleTokenPositionDescriptor` contract have only view functions, but it is still possible that it will be used on chain, which means that gas can be claimed for such usage. But it [is not configured](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-clmm/contracts/NonfungibleTokenPositionDescriptor.sol#L26-L29).

### Recommendation
Add ability to claim gas.

## QA-04. ThrusterTreasure.setPrize allows to change prize after winners selected
### Description
ThrusterTreasure.setPrize function allows to [set prize for current round](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L167) and for any index in it. It's possible that winner were already selected, but `setRoot` was not called yet, so [round has not increased](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-treasure/contracts/ThrusterTreasure.sol#L255), then owner has ability to decrease prize.
### Recommendation
In case if prize was set, then don't allow to change it.

## QA-05. Admin has ability to withdraw funds
### Description
Using `ThrusterTreasure.retrieveTokens` function admin can withdraw funds that were provided as prizes and create insolvency of contract.
### Recommendation
Once prize is set, then don't allow to change it and withdraw funds.
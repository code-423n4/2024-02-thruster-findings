## QA-01. Change gas manager, when yieldToSetter is changed
### Description
When `ThrusterFactory` is deployed, then it's gas manager [is set to `yieldToSetter`](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-cfmm/contracts/ThrusterFactory.sol#L35C82-L35C96).

Later it's possible that `yieldToSetter` [is changed](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-cfmm/contracts/ThrusterFactory.sol#L68-L73) for any reason(for example private key leak), but gas manager is not changed and can be then changed by attacker or used to claim gas to own address.

Similarly for all pools factory `yieldToSetter` [is set as gas and yield manager](https://github.com/code-423n4/2024-02-thruster/blob/main/thruster-protocol/thruster-cfmm/contracts/ThrusterPair.sol#L79). In case if `yieldToSetter` is changed in factory, then manager should be changed, however it will be not possible to do because of amount of deployed pools.

### Recommendation
Change gas manager for factory.
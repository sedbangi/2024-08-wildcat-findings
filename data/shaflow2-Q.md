| Issue Number | Title                                                                                                                                                                                                          
|--------------|-------------------------------------------------|
| [01]         | Unnecessary Checks                              |
| [02]         | Lack Hooks Disabled Check                    |
| [03]         | It Is Not Possible to Remove Already Deployed Hooks |
| [04]         | The Current Salt Validation Might Expose the Market Deployment Transaction to Potential DoS Attacks  |
| [05]         | The Impact of Blacklisting Tokens on the System  |
| [06]         | Check Status Before Overriding Sanctions        |

## [01] Unnecessary checks
### Link
github:[https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L531](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L531)
### Description
The `deployMarketAndHooks` function checks `templateDetails.exists`, which is unnecessary because the relevant checks are already performed in `_deployHooksInstance`.  

```solidity
  function deployMarketAndHooks(
    address hooksTemplate,
    bytes calldata hooksTemplateArgs,
    DeployMarketInputs memory parameters,
    bytes calldata hooksData,
    bytes32 salt,
    address originationFeeAsset,
    uint256 originationFeeAmount
  ) external override nonReentrant returns (address market, address hooksInstance) {
    if (!IWildcatArchController(_archController).isRegisteredBorrower(msg.sender)) {
      revert NotApprovedBorrower();
    }
    HooksTemplate memory templateDetails = _templateDetails[hooksTemplate];
    if (!templateDetails.exists) {
      revert HooksTemplateNotFound();
    }
    hooksInstance = _deployHooksInstance(hooksTemplate, hooksTemplateArgs);
    ...
  }
```

### Recommended Mitigation Steps
Remove the redundant check
```diff
  function deployMarketAndHooks(
    address hooksTemplate,
    bytes calldata hooksTemplateArgs,
    DeployMarketInputs memory parameters,
    bytes calldata hooksData,
    bytes32 salt,
    address originationFeeAsset,
    uint256 originationFeeAmount
  ) external override nonReentrant returns (address market, address hooksInstance) {
    if (!IWildcatArchController(_archController).isRegisteredBorrower(msg.sender)) {
      revert NotApprovedBorrower();
    }
-   HooksTemplate memory templateDetails = _templateDetails[hooksTemplate];
-   if (!templateDetails.exists) {
-     revert HooksTemplateNotFound();
-   }
    hooksInstance = _deployHooksInstance(hooksTemplate, hooksTemplateArgs);
    ...
  }
```
## [02] Lack hooks disabled check
### Link
github:[https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L491](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L491)
### Description
The `HooksFactory` only checks whether the hook instance is registered when deploying the market contract, but does not check if it is disabled.  
This could allow users to deploy markets using hooks instances that are already deployed but have their configurations disabled.  

```solidity
  function deployMarket(
    DeployMarketInputs calldata parameters,
    bytes calldata hooksData,
    bytes32 salt,
    address originationFeeAsset,
    uint256 originationFeeAmount
  ) external override nonReentrant returns (address market) {
    if (!IWildcatArchController(_archController).isRegisteredBorrower(msg.sender)) {
      revert NotApprovedBorrower();
    }
    address hooksInstance = parameters.hooks.hooksAddress();
    address hooksTemplate = getHooksTemplateForInstance[hooksInstance];
    if (hooksTemplate == address(0)) {
      revert HooksInstanceNotFound();
    }
    HooksTemplate memory templateDetails = _templateDetails[hooksTemplate];
    market = _deployMarket(
      parameters,
      hooksData,
      hooksTemplate,
      templateDetails,
      salt,
      originationFeeAsset,
      originationFeeAmount
    );
  }
```
### Recommended Mitigation Steps
```diff
  function deployMarket(
    DeployMarketInputs calldata parameters,
    bytes calldata hooksData,
    bytes32 salt,
    address originationFeeAsset,
    uint256 originationFeeAmount
  ) external override nonReentrant returns (address market) {
    if (!IWildcatArchController(_archController).isRegisteredBorrower(msg.sender)) {
      revert NotApprovedBorrower();
    }
    address hooksInstance = parameters.hooks.hooksAddress();
    address hooksTemplate = getHooksTemplateForInstance[hooksInstance];
    if (hooksTemplate == address(0)) {
      revert HooksInstanceNotFound();
    }

+   if (!_templateDetails[hooksTemplate].enabled) {
+       revert HooksTemplateNotAvailable();
+   }

    HooksTemplate memory templateDetails = _templateDetails[hooksTemplate];
    market = _deployMarket(
      parameters,
      hooksData,
      hooksTemplate,
      templateDetails,
      salt,
      originationFeeAsset,
      originationFeeAmount
    );
  }
```
## [03]It is not possible to remove already deployed hooks.
### Link
github:[https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L327](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L327)
### Description
Already deployed `hooksInstance` will be added to the `getHooksTemplateForInstance` mapping. Over time, some `hooksInstance` may become deprecated, but there is no method to remove these `hooksInstance`. Users can still deploy markets using them, and even if the corresponding `_templateDetails` is disabled, it cannot prevent this.
```solidity
  function _deployHooksInstance(
    address hooksTemplate,
    bytes calldata constructorArgs
  ) internal returns (address hooksInstance) {
    ...
    emit HooksInstanceDeployed(hooksInstance, hooksTemplate);
    getHooksTemplateForInstance[hooksInstance] = hooksTemplate;
  }
```

### Recommended Mitigation Steps
Add a function to remove deprecated hooks instances.
```diff
+ function removeHooksInstance(address hooksTemplate) external onlyArchControllerOwner{
+   delete getHooksTemplateForInstance[hooksInstance];
+ }
```
## [04] The current salt validation might expose the market deployment transaction to potential DoS attacks.
### Link
github:[https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L407](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L407)
### Description
The contract uses the `create2` method to deploy the market, allowing users to provide a salt.
However, since the salt check allows the last 20 bytes of the salt to be zero, deployment transactions with such salts could be subject to front-running attacks by malicious users who use the same salt.
```solidity
    if (!(address(bytes20(salt)) == msg.sender || bytes20(salt) == bytes20(0))) {
      revert SaltDoesNotContainSender();
    }
```
### Recommended Mitigation Steps
```diff
-   if (!(address(bytes20(salt)) == msg.sender || bytes20(salt) == bytes20(0))) {
+   if (!(address(bytes20(salt)) == msg.sender)) {
      revert SaltDoesNotContainSender();
    }
```

## [05] The impact of blacklisting tokens on the system
### Description
The protocol supports blacklisted tokens, but if the market contract itself is blacklisted, any funds within the contract cannot be withdrawn. During this period, lenders can still initiate withdrawal requests, but borrowers will be unable to repay their loans. This situation remains until the protocol is removed from the blacklist, which could result in penalties.  
### Recommended Mitigation Steps
It is recommended to check if the involved parties are blacklisted. If they are, implement solutions such as halting the calculation of expiry times, pausing lender withdrawal requests, or migrating the contract to a new address.

## [06] Check status before overriding sanctions
### Link 
github: [https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatSanctionsSentinel.sol#L96](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatSanctionsSentinel.sol#L96)
github: [https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatSanctionsSentinel.sol#L104](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatSanctionsSentinel.sol#L104)
### Description
Since the status change of `sanctionOverrides` will trigger an event, it's best to check the status of `sanctionOverrides` before updating it to avoid emitting redundant events.  
```solidity
  /**
   * @dev Overrides the sanction status of `account` for `borrower`.
   */
  function overrideSanction(address account) public override {
    sanctionOverrides[msg.sender][account] = true;
    emit SanctionOverride(msg.sender, account);
  }

  /**
   * @dev Removes the sanction override of `account` for `borrower`.
   */
  function removeSanctionOverride(address account) public override {
    sanctionOverrides[msg.sender][account] = false;
    emit SanctionOverrideRemoved(msg.sender, account);
  }
```
### Recommended Mitigation Steps
```diff
  /**
   * @dev Overrides the sanction status of `account` for `borrower`.
   */
  function overrideSanction(address account) public override {
+   require(!sanctionOverrides[msg.sender][account], "Already Overrides");
    sanctionOverrides[msg.sender][account] = true;
    emit SanctionOverride(msg.sender, account);
  }

  /**
   * @dev Removes the sanction override of `account` for `borrower`.
   */
  function removeSanctionOverride(address account) public override {
+   require(sanctionOverrides[msg.sender][account], "Not Overrides");
    sanctionOverrides[msg.sender][account] = false;
    emit SanctionOverrideRemoved(msg.sender, account);
  }
```
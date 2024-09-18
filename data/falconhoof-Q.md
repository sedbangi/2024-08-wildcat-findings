# Wildcat Protocol QA Report
____
## [L-01] `AccessControlHooks::onQueueWithdrawal` missing validation
____
### Context
See: https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/AccessControlHooks.sol#L812-L825

### Description
The `onQueueWithdrawal()` hook in AccessControlHooks is missing access control that ensures it can only be called by a hooked market.
In some cases this would allow a malicious user to unset a victim's credentials via `_tryValidateAccess()`

### Recommendation
Add the check as below:

```diff
  function onQueueWithdrawal(
    address lender,
    uint32 /* expiry */,
    uint /* scaledAmount */,
    MarketState calldata /* state */,
    bytes calldata hooksData
  ) external override {

+   if (!market.isHooked) revert NotHookedMarket();

    LenderStatus memory status = _lenderStatus[lender];
    if (
      !isKnownLenderOnMarket[lender][msg.sender] && !_tryValidateAccess(status, lender, hooksData)
    ) {
      revert NotApprovedLender();
    }
  }
```


## [L-02] onBorrow() uses the wrong `Base_Size`
____

### Context
See: https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/AccessControlHooks.sol#L812-L825

### Description
When making the `onBorrow` call from hooksConfig to the hooks Instance; `RepayHook_Base_Size` is used instead of `Borrow_Base_Size`

### Recommendation
Update as below:

```diff
  function onBorrow(HooksConfig self, uint256 normalizedAmount, MarketState memory state) internal {
    address target = self.hooksAddress();
    uint32 onBorrowSelector = uint32(IHooks.onBorrow.selector);
    if (self.useOnBorrow()) {
      assembly {
        let extraCalldataBytes := sub(calldatasize(), BorrowCalldataSize)
        let ptr := mload(0x40)
        let headPointer := add(ptr, 0x20)

        mstore(ptr, onBorrowSelector)
        // Copy `normalizedAmount` to hook calldata
        mstore(headPointer, normalizedAmount)
        // Copy market state to hook calldata
        mcopy(add(headPointer, BorrowHook_State_Offset), state, MarketStateSize)
        // Write bytes offset for `extraData`
        mstore(
          add(headPointer, BorrowHook_ExtraData_Head_Offset),
          BorrowHook_ExtraData_Length_Offset
        )
        // Write length for `extraData`
        mstore(add(headPointer, BorrowHook_ExtraData_Length_Offset), extraCalldataBytes)
        // Copy `extraData` from end of calldata to hook calldata
        calldatacopy(
          add(headPointer, BorrowHook_ExtraData_TailOffset),
          BorrowCalldataSize,
          extraCalldataBytes
        )

-       let size := add(RepayHook_Base_Size, extraCalldataBytes)
+       let size := add(Borrow_Base_Size, extraCalldataBytes)
        if iszero(call(gas(), target, 0, add(ptr, 0x1c), size, 0, 0)) {
          returndatacopy(0, 0, returndatasize())
          revert(0, returndatasize())
        }
      }
    }
  }
```


## [L-03] deployMarket() doesn't check that hooksInstance is enabled && exists
____

### Context
See: https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L393-L489
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L275-L287
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L491-L545

### Description
The process for deploying a market involves first deploying a hooksInstance, which is based off a template, and then deploying a market using that hooksInstance.
However, the template used to create the hooksInstance may have subsequently been found to contain a bug and disabled by the protocol and hence should not be used anymore.

Given that multiple markets can be run off a single hooksInstance it is likely that a borrower would not realise that the hook has been disabled and is no longer fit for purpose anymore.

The check here in [`deployMarket()`](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L491-L545) will revert if the template doesn't exist but there is non check whether the template has been disabled.

```
    address hooksTemplate = getHooksTemplateForInstance[hooksInstance];
    if (hooksTemplate == address(0)) {
      revert HooksInstanceNotFound();
    }
```

### Recommendation
Add the check as below:

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
+   if (!template.enabled) {
+     revert HooksTemplateNotAvailable();
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


## [L-04] functions can revert due to overflow if underlying asset has high decimals
____

### Context
See: 

### Description
If a market uses a token with high decimals, such as NEAR token (24 decimals), it is possible for there to be an overflow in [`_applyWithdrawalBatchPayment()`](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketBase.sol#L665-L695) which would cause reverts across a wide range of functions.

If the `maxTotalSupply` is larger than uint104; which is permitted as it can go up to a maximum of uint128 then the scaledAvailableLiquidity calculation can oveflow which will cause a revert due to the [`safeCastLib`](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/libraries/SafeCastLib.sol#L65-L67) being used.

```
    function _applyWithdrawalBatchPayment(
      WithdrawalBatch memory batch,
      MarketState memory state,
      uint32 expiry,
      uint256 availableLiquidity
    ) internal {
>>>       uint104 scaledAvailableLiquidity = state.scaleAmount(availableLiquidity).   toUint104();
    
      // more code
    }

```

`_applyWithdrawalBatchPayment()` gets called within `_getUpdatedState()` which is called at the beginning of every function call so it would lead to a DOS across the protocol though the likelihood of a borrower setting up a market with a 24 decimal token is low.

### POC
1. Market opens with `maxTotalSupply > uint104`
2. Market quickly receives more than uint104 of underlying asset before borrower borrows
3. Most functions of the market will now be DOS'd  due to the overflow revert in `_applyWithdrawalBatchPayment()`

### Recommendation
Use a larger type than `uint104` closer to maxTotalSupply like `uint128`

## [L-05] pushProtocolFeeBipsUpdates() reverts if there are closed markets in the array
____

### Context
See: https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L594-L596
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L553-L587
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketConfig.sol#L164-L176

### Description
The protocol updates fees by first calling `updateHooksTemplateFees()` to store the new fee in a particular template which from then on will apply to any new markets created with that template. 

The fee change should then be propagated out to existing markets, using that template, by calling one of the two `pushProtocolFeeBipsUpdates()` functions [here](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L594-L596) and [here](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L553-L587).

They can be called by anyone and take a starting index and ending index. Likely it would be the protocol proogating a fee increase or lenders/borrower for a fee decrease.

```
  function pushProtocolFeeBipsUpdates(address hooksTemplate) external {
    pushProtocolFeeBipsUpdates(hooksTemplate, 0, type(uint256).max);
  }
```

The issue is that the function which makes the change on each individual market [`setProtocolFeeBips()`](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketConfig.sol#L164-L176) reverts if there is a closed market in the indexes of markets provided.
So the function would need to be called multiple times in order to update the fees across all markets. It owuld become more and more expensive as time goes on and more markets close.

```
    function setProtocolFeeBips(
      uint16 _protocolFeeBips
    ) external nonReentrant sphereXGuardExternal {
      if (msg.sender != factory) revert_NotFactory();
      if (_protocolFeeBips > 1_000) revert_ProtocolFeeTooHigh();
      MarketState memory state = _getUpdatedState();
      // @audit : revert here
>>>   if (state.isClosed) revert_ProtocolFeeChangeOnClosedMarket();
      if (_protocolFeeBips == state.protocolFeeBips) revert_ProtocolFeeNotChanged();
      hooks.onSetProtocolFeeBips(_protocolFeeBips, state);
      state.protocolFeeBips = _protocolFeeBips;
      emit ProtocolFeeBipsUpdated(_protocolFeeBips);
      _writeState(state);
    }
```

### Recommendation
Skip markets which are closed rather than reverting
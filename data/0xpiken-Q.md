# Low
## [L&#x2011;01] [`HooksFactory#deployMarketAndHooks()`](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/HooksFactory.sol#L518-L545) might revert due to incorrect shifting in [`LibHooksConfig#setHooksAddress()`](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/types/HooksConfig.sol#L84-L94)
`hooks` should be shifted left 160 bits then right 160 bits to clean the address:
```diff
  function setHooksAddress(
    HooksConfig hooks,
    address _hooksAddress
  ) internal pure returns (HooksConfig updatedHooks) {
    assembly {
      // Shift twice to clear the address
-     updatedHooks := shr(96, shl(96, hooks))
+     updatedHooks := shr(160, shl(160, hooks))
      // Set the new address
      updatedHooks := or(updatedHooks, shl(96, _hooksAddress))
    }
  }
```
## [L&#x2011;02] [`HooksFactory#pushProtocolFeeBipsUpdates()`](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/HooksFactory.sol#L553-L587) might revert if the market is closed or the protocol fee of the market is same as the new value.
Check whether the market is closed or the protocol fee is same as the new value before calling `market.setProtocolFeeBips()`:
```diff
  function pushProtocolFeeBipsUpdates(
    address hooksTemplate,
    uint marketStartIndex,
    uint marketEndIndex
  ) public override nonReentrant {
    HooksTemplate memory details = _templateDetails[hooksTemplate];
    if (!details.exists) revert HooksTemplateNotFound();

    address[] storage markets = _marketsByHooksTemplate[hooksTemplate];
    marketEndIndex = MathUtils.min(marketEndIndex, markets.length);
    uint256 count = marketEndIndex - marketStartIndex;
    uint256 setProtocolFeeBipsCalldataPointer;
    uint16 protocolFeeBips = details.protocolFeeBips;
    assembly {
      // Write the calldata for `market.setProtocolFeeBips(protocolFeeBips)`
      // this will be reused for every market
      setProtocolFeeBipsCalldataPointer := mload(0x40)
      mstore(0x40, add(setProtocolFeeBipsCalldataPointer, 0x40))
      // Write selector for `setProtocolFeeBips(uint16)`
      mstore(setProtocolFeeBipsCalldataPointer, 0xae6ea191)
      mstore(add(setProtocolFeeBipsCalldataPointer, 0x20), protocolFeeBips)
      // Add 28 bytes to get the exact pointer to the first byte of the selector
      setProtocolFeeBipsCalldataPointer := add(setProtocolFeeBipsCalldataPointer, 0x1c)
    }
    for (uint256 i = 0; i < count; i++) {
      address market = markets[marketStartIndex + i];
+     if (WildcatMarket(market).isClosed() || WildcatMarket(market).currentState().protocolFeeBips == protocolFeeBips) continue;
      assembly {
        if iszero(call(gas(), market, 0, setProtocolFeeBipsCalldataPointer, 0x24, 0, 0)) {
          // Equivalent to `revert SetProtocolFeeBipsFailed()`
          mstore(0, 0x4484a4a9)
          revert(0x1c, 0x04)
        }
      }
    }
  }
```
## [L&#x2011;03] [`AccessControlHooks#grantRole()`](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L376-L382) doesn't check whether `roleGrantedTimestamp` is greater than `block.timestamp`.
Check if `roleGrantedTimestamp` is greater than `block.timestamp` before granting credential to the lender:
```diff
  function _grantRole(
    RoleProvider callingProvider,
    address account,
    uint32 roleGrantedTimestamp
  ) internal {
    LenderStatus memory status = _lenderStatus[account];

+   if (roleGrantedTimestamp > block.timestamp) revert InvalidCredential();
    uint256 newExpiry = callingProvider.calculateExpiry(roleGrantedTimestamp);

    // Check if the new credential is still valid
    if (newExpiry < block.timestamp) revert GrantedCredentialExpired();

    // Check if the account has ever had a credential
    if (status.hasCredential()) {
      RoleProvider lastProvider = _roleProviders[status.lastProvider];

      // Check if the provider that last granted access is still supported
      if (!lastProvider.isNull()) {
        uint256 oldExpiry = lastProvider.calculateExpiry(status.lastApprovalTimestamp);

        // Can only update role if the caller is the previous role provider or the new
        // expiry is greater than the previous expiry.
        if (!((status.lastProvider == msg.sender).or(newExpiry > oldExpiry))) {
          revert ProviderCanNotReplaceCredential();
        }
      }
    }

    _setCredentialAndEmitAccessGranted(status, callingProvider, account, roleGrantedTimestamp);
  }
```
# Info
## [I&#x2011;01] `BorrowHook_Base_Size` should be used instead of `RepayHook_Base_Size` in [`HooksConfig#onBorrow()`](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/types/HooksConfig.sol#L505-L540):
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
+       let size := add(BorrowHook_Base_Size, extraCalldataBytes)
        if iszero(call(gas(), target, 0, add(ptr, 0x1c), size, 0, 0)) {
          returndatacopy(0, 0, returndatasize())
          revert(0, returndatasize())
        }
      }
    }
  }
```
## [I&#x2011;02] `normalizedAmountWithdrawn` should be used instead of `scaledAmount` in [`HooksConfig#onExecuteWithdrawal‎()`](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/types/HooksConfig.sol#L382-L426) to avoid misunderstanding:
```diff
  function onExecuteWithdrawal(
    HooksConfig self,
    address lender,
-   uint256 scaledAmount,
+   uint256 normalizedAmountWithdrawn,
    MarketState memory state,
    uint256 baseCalldataSize
  ) internal {
```
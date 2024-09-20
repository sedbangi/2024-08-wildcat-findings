# Findings Summary

| ID     | Title                                                                          | Severity     |
| ------ | ------------------------------------------------------------------------------ | --------     |
| [L-01] | When the market is closed, the getAvailableWithdrawalAmount view function and the withdrawal function behave inconsistently, which may prevent withdrawals | Low          |
| [N-01] | _loopTryGetCredential should query from back to front to save gas              | Non-Critical |

# Detailed Findings

# [L-01] When the market is closed, the getAvailableWithdrawalAmount view function and the withdrawal function behave inconsistently, which may prevent withdrawals

## Description

```solidity
  function _queueWithdrawal(
    MarketState memory state,
    Account memory account,
    address accountAddress,
    uint104 scaledAmount,
    uint normalizedAmount,
    uint baseCalldataSize
  ) internal returns (uint32 expiry) {

    // Cache batch expiry on the stack for gas savings
    expiry = state.pendingWithdrawalExpiry;

    // If there is no pending withdrawal batch, create a new one.
    if (state.pendingWithdrawalExpiry == 0) {
      // @audit If the market is closed, use zero for withdrawal batch duration.
      uint duration = state.isClosed.ternary(0, withdrawalBatchDuration);
      expiry = uint32(block.timestamp + duration);
      emit_WithdrawalBatchCreated(expiry);
      state.pendingWithdrawalExpiry = expiry;
    }
  }

  function _executeWithdrawal(
    MarketState memory state,
    address accountAddress,
    uint32 expiry,
    uint baseCalldataSize
  ) internal returns (uint256) {
    WithdrawalBatch memory batch = _withdrawalData.batches[expiry];
    // @audit If the market is closed, allow withdrawal prior to expiry.
    if (expiry >= block.timestamp && !state.isClosed) {
      revert_WithdrawalBatchNotExpired();
    }
  }

  function getAvailableWithdrawalAmount(
    address accountAddress,
    uint32 expiry
  ) external view nonReentrantView returns (uint256) {
    @audit If the market is closed, getAvailableWithdrawalAmount may still revert
    if (expiry >= block.timestamp) {
      revert_WithdrawalBatchNotExpired();
    }
  }
```

When the market is closed, the view function and the withdrawal function behave inconsistently, which may prevent withdrawals if the user monitors the available withdrawal funds through `getAvailableWithdrawalAmount`.

## Recommendations

When the market is closed, the `getAvailableWithdrawalAmount` need to return the correct amount immediately instead of waiting for the expiration time.

# [N-01] _loopTryGetCredential should query from back to front to save gas

## Description

```solidity
  function _grantRole(
    RoleProvider callingProvider,
    address account,
    uint32 roleGrantedTimestamp
  ) internal {
    LenderStatus memory status = _lenderStatus[account];

    uint256 newExpiry = callingProvider.calculateExpiry(roleGrantedTimestamp);

    // Check if the new credential is still valid
    if (newExpiry < block.timestamp) revert GrantedCredentialExpired();

    // Check if the account has ever had a credential
    if (status.hasCredential()) {
      RoleProvider lastProvider = _roleProviders[status.lastProvider];

      // Check if the provider that last granted access is still supported
      if (!lastProvider.isNull()) {
        uint256 oldExpiry = lastProvider.calculateExpiry(status.lastApprovalTimestamp);

        // @audit Can only update role if the caller is the previous role provider or the new
        // expiry is greater than the previous expiry.
        if (!((status.lastProvider == msg.sender).or(newExpiry > oldExpiry))) {
          revert ProviderCanNotReplaceCredential();
        }
      }
    }

    _setCredentialAndEmitAccessGranted(status, callingProvider, account, roleGrantedTimestamp);
  }

  function _loopTryGetCredential(
    LenderStatus memory status,
    address accountAddress,
    uint256 pullProviderIndexToSkip
  ) internal view returns (bool foundCredential) {
    uint256 providerCount = _pullProviders.length;
    // @audit should query from back to front to save gas
    for (uint256 i = 0; i < providerCount; i++) {
      if (i == pullProviderIndexToSkip) continue;
      RoleProvider provider = _pullProviders[i];
      if (_tryGetCredential(status, provider, accountAddress)) return (true);
    }
  }
```

Provider's expiry is increasing, so the later provider is more likely to be valid, and you should look for a backwards to save gas.

## Recommendations

_loopTryGetCredential should query from back to front to save gas

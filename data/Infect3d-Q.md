# L01 - `WildcatArchController::removeMarket` do not call XSphere.removeAllowedSender
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/WildcatArchController.sol#L347-L360

## Vulnerability details
Adding a market to the Arch Controller also add its address to the allowed sender of the XSphere component.
But removing the market from the Arch Controller do not remove it from the allowed senders

```solidity
File: src/WildcatArchController.sol
347:   function registerMarket(address market) external onlyController {
348:     if (!_markets.add(market)) {
349:       revert MarketAlreadyExists();
350:     }
351:     _addAllowedSenderOnChain(market);
352:     emit MarketAdded(msg.sender, market);
353:   }
354: 
355:   function removeMarket(address market) external onlyOwner {
356:     if (!_markets.remove(market)) {
357:       revert MarketDoesNotExist();
358:     }
359:     emit MarketRemoved(market);
360:   }
```

## Impact
If a market behave maliciously and is removed from the Arch Controller, it will still be considered a trustable address for the XSphere defensive mechanism.

## Tools Used
Manual review

## Recommended Mitigation Steps
Call `removeAllowedSender` : https://github.com/spherex-xyz/spherex-contracts/blob/f025ca3fcdc3705c0d87a0b198a2cf762bca7310/src/SphereXEngine.sol#L161

# L02 - `onRepay` hook can be avoided by directly sending funds to market as assets are accounted as balanceOf
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarket.sol#L212-L212

## Summary
Because `market.totalAssets()` uses `asset.balanceOf()` to compute the available assets to execute withdrawals, it is possible to repay the market without going through the `onRepay` function, which might be important.

## Vulnerability details
Right now, none of the developed hooks implemented a logic for `onRepay` so there is no impact.
But once this hook will be used, please note that the borrower will be able to possibly not be subject to the necessary hook logic by simply transfering the repay amount directly to the contract.

This is the case 

## Impact
Borrower can avoid executing the `onRepay` logic.

## Tools Used
Manual review

## Recommended Mitigation Steps
Using an internal accounting for `totalAssets` would be a solution, but this would  make rebasing tokens incompatible.

# L03 - Market should allow rescuing market's internal tokens as there is no risk of stealing from borrower
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarket.sol#L38-L38

## Vulnerability details
The `WildcatMarket::rescueToken` function prevent borrower to rescue market's asset tokens, which make total sense as this would allow to borrow the funds without going through the borrower Chainalysis sanction check.

But the function disallow the transfer of market's shares, which make seems to make less sense, as the market should never have a balance of its own tokens in the first place, as only Accounts that deposited, or got transfered market's shares have a balance.

## Impact
If non tech-savy lenders mistakenly transfer market's shares to the market (which is likely to happen as this can be seen in many DeFi protocols), there is no way to recover them, while this is the purpose of the function.

## Tools Used
Manual review

## Recommended Mitigation Steps
To keep the `rescueToken` unchanged, a check could be added in `WildcatMarketToken::transfer` if the recipient `to` is the market itself.

# L04 - As `_calculateTemporaryReserveRatioBips` rounds down the `relativeDiff` value, borrower can reduce more than 25% without triggering a Reserve Ratio update
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/MarketConstraintHooks.sol#L167-L167

## Vulnerability details
The `_calculateTemporaryReserveRatioBips` checks if the APR has been reduced by more than 25% from its previous value.
[If this is the case](https://docs.wildcat.finance/using-wildcat/day-to-day-usage/borrowers#reducing-apr), the Reserve Ratio of the market must be increased based on a formula
As the precision of the calculations are Bips (10_000), if the borrower reduce the APR by 25.009%, this will be considered 25.00% still, and will not trigger a Reserve Ratio update.

## Impact
Borrower can reduce the APR more than it should without triggering the Reserve Ratio update.

## Tools Used
Manual review

## Recommended Mitigation Steps
The `relativeDiff` calculation should use the `bipMul` function which proceed with a half-rounding, rather than always rounding down.

# L05 - Nothing prevent to create an escrow contract after borrower gets sanctionned


## Vulnerability details
https://docs.wildcat.finance/using-wildcat/day-to-day-usage/the-sentinel#borrower-gets-sanctioned
The Wildcat Documentation states that once a borrower get sanctionned no escrow contract can be created for the borrower's related markets.
But this is not true, right now, nothing prevent an escrow contract to be created:
- `WildcatMarketWithdrawals::executeWithdrawal` do not check borrower's sanction before creating the escrow contract
- the `WildcatSanctionsSentinel::createEscrow` function is not protected and allow anyone to create an escrow contract

## Impact
What seems a requirement from the specification is not respected by the implementation.

## Proof of Concept
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarketWithdrawals.sol#L194-L194
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/WildcatSanctionsSentinel.sol#L121-L121

## Tools Used
Manual review

## Recommended Mitigation Steps
check borrower's sanction before creating the escrow contract.

# L06 - `getLenderStatus` calls `_tryGetCredential` without checking if `provider` is a pullProvider

## Summary
`_tryGetCredential` should only be called if a provider is a pull provider [as explained in the function natspec](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L467-L468), but `getLenderStatus` do not perform this check thoroughly.

## Vulnerability details
Before calling `_tryGetCredential`, `getLenderStatus` does a check on the lender status to verify if the previous:
```solidity
File: src/access/AccessControlHooks.sol
344:         if (status.canRefresh) {
345:           if (_tryGetCredential(status, provider, accountAddress)) {
346:             return status;
347:           }
```

To inform they are pull providers (can automatically refresh credential), role providers must implement [a function `isPullProvider()`](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/IRoleProvider.sol#L5)
The implementation is not define, and do not say if the function will always return 'true', as it might depend on an internal logic.

That means that it is possible that a role provider who stop becoming a pull provider for any reason will still be accessed through the function to get the lender status and tell if his credential are valid. 

## Impact
`_tryGetCredential` called without checking if the provider is still a pull provider

## Tools Used
Manual review

## Recommended Mitigation Steps
Call the provider address with `isPullProvider()` to ensure it can still grant credentials


# L07 - Sanctioned account can poison market by transfering assets directly to market
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarketBase.sol#L458

## Vulnerability details
A malicious actor with sanctionned funds can send tokens to a market right before `updateState/_getUpdatedState` and `_writeState` are called. The funds will be marked for a batch, and registered in the `availableLiquidity`

Then, it will be used to calculate the new values for `batch.scaledAmountBurned` and `state.scaledPendingWithdrawals`, basically poisoning the market and lenders who received those assets with assets from an OFAC sanctionned address.
This could cause multiple lenders to be also flagged as sanctionned causing further legal issues.

Because assets can be deposited to be distributed without interacting with the protocol external functions, XSphere protection will have no power to block this.
Also, this might not be noticed by the borrower.

## Impact
Market can distribute sanctionned funds to lenders which might cause trouble to borrower and all users

## Tools Used
Manual review

## Recommended Mitigation Steps
Use an internal accounting system to ensure no poisoned funds can be dispersed through the market.



# L08 - In `_getUpdateState`, the second `updateScaleFactorAndFees` for non expired batches undervaluate the protocol fees when `_processExpiredWithdrawalBatch` is called right before

## Vulnerability details
See similar finding: https://github.com/code-423n4/2023-10-wildcat-findings/issues/455

Every time `_getUpdateState` is called, two fees update are computed following this scheme:
1. if expired batches exists, compute fees for expired batch and update `lastInterestAccruedTimestamp = expiry`
2. process expired batch, which update the `state.scaledTotalSupply` value (reducing it)
3. compute fees from `lastInterestAccruedTimestamp` to `block.timestamp`

The thing is that `updateScaleFactorAndFees` uses `state.scaledTotalSupply` [to compute the protocol fees](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/libraries/FeeMath.sol#L40-L50), but as the `state.scaledTotalSupply` is reduced in step (2), the amount of fees paid for the period `block.timestamp - expiry` is computed on the `state.scaledTotalSupply` **minus what has been processed in `_processExpiredWithdrawalBatch`**, even though that the `state.scaledTotalSupply` was present in the market until `block.timestamp`.

## Impact
Wrong computation of the scale factor and fees.



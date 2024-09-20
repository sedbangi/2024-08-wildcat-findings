# QA Report for **Wildcat**
## Table of Contents

| Issue ID | Description |
| -------- | ----------- |
| [QA-01](#qa-01-market-can-immediately-fall-into-delinquency) | Market can immediately fall into delinquency |
| [QA-02](#qa-02-use-a-pull-pattern-instead-of-push-when-collecting-fees) | Use a pull pattern instead of push when collecting fees |
| [QA-03](#qa-03-borrowers-would-pay-lesser-apr-in-some-supported-assets) | Borrowers would pay lesser APR in some supported assets |
| [QA-04](#qa-04-amount-are-not-scaled-during-approvals-and-allows-for-wrong-amount-to-be-transferred) | Amount are not scaled during approvals and allows for wrong amount to be transferred |
| [QA-05](#qa-05-borrowers-would-lose-a-lot-of-funds-if-market-is-intentionally/unintentionally-frequently-updated) | Borrowers would lose a lot of funds if market is intentionally/unintentionally frequently updated |
| [QA-06](#qa-06-deposits-are-broken-in-an-edge-case) | Deposits are broken in an edge case |
| [QA-07](#qa-07-releasing-escrowed-funds-might-silently-fail) | Releasing escrowed funds might silently fail |
| [QA-08](#qa-08-apply-some-sort-of-ac-to-wildcatmarketwithdrawals#executewithdrawal()) | Apply some sort of AC to `WildcatMarketWithdrawals#executeWithdrawal()` |
| [QA-09](#qa-09-setters-dont-have-equality-checkers-) | Setters don't have equality checkers  |
| [QA-10](#qa-10-transferfrom-should-only-use-allowance-when--spender-!=-from) | transferFrom should only use allowance when  `spender != from` |
| [QA-11](#qa-11-total-supply-would-be-out-of-sync-for-wmtokens) | Total supply would be out of sync for WMTokens |
| [QA-12](#qa-12-some-assets-cant-be-used-as-underlying-) | Some assets can't be used as underlying  |
| [QA-13](#qa-13-valid-deposits-would-be-bricked-in-an-edge-case) | Valid deposits would be bricked in an edge case |
| [QA-14](#qa-14-sanctioned-lenders-can-contaminate-market-funds) | Sanctioned Lenders Can Contaminate Market Funds |
| [QA-15](#qa-15-market-settlements-can-be-bricked) | Market Settlements can be bricked |
| [QA-16](#qa-16-unauthorized-actors-can-sidestep-their-restrictions-and-still-integrate-with-wildcat) | Unauthorized actors can sidestep their restrictions and still integrate with Wildcat |
| [QA-17](#qa-17-a-borrower-can-remove-their-escrow-address-via-removesanctionoverride()) | A borrower can remove their escrow address via `removeSanctionOverride()` |
| [QA-18](#qa-18-annual-interest--&-reserve-ratio-can-still-be-changed-for-an-already-closed-market) | Annual interest  & reserve ratio can still be changed for an already closed market |
| [QA-19](#qa-19-admin-could-intentionally/unintentionally-front-run-borrower-with-the-fee-increasing-) | Admin could intentionally/unintentionally front-run borrower with the fee increasing  |
| [QA-20](#qa-20-market-list-can-easily-get-bloated) | Market list can easily get bloated |
| [QA-21](#qa-21-potential-limitations-of-openzeppelins-enumerableset-usage) | Potential Limitations of OpenZeppelin's EnumerableSet Usage |
| [QA-22](#qa-22-import-declarations-should-import-specific-identifiers-rather-than-the-whole-file) | Import declarations should import specific identifiers, rather than the whole file |
| [QA-23](#qa-23-credentials-would-no-longer-be-grant-able-after-a-while) | Credentials would no longer be grant-able after a while |
| [QA-24](#qa-24-hascredential-does-not-factor-in-expiration) | `hasCredential` does not factor in expiration |
| [QA-25](#qa-25-remove-linter-errors-from-code) | Remove linter errors from code |
| [QA-26](#qa-26-push0-opcode-is-not-supported-on-all-to-deploy-chains) | PUSH0 Opcode is not supported on all to-deploy chains |
## QA-01 Market can immediately fall into deliquency

### Proof of Concept
First note that the protocol allows borrowers to set a reserve ratio that they must maintain to avoid being charged a delinquency fee.

Now, see https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/MarketConstraintHooks.sol#L44-L45

```solidity
  uint16 internal constant MaximumReserveRatioBips = 10_000;

```

From here we can see that the maximum acceptable value for this is `10_000`.

Now when creating the market, this is called: https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/MarketConstraintHooks.sol#L136-L150

```solidity

  function _onCreateMarket(
    address /* deployer */,
    address /* marketAddress */,
    DeployMarketInputs calldata parameters,
    bytes calldata /* extraData */
  ) internal virtual override returns (HooksConfig) {
    enforceParameterConstraints(
      parameters.annualInterestBips,
      parameters.delinquencyFeeBips,
      parameters.withdrawalBatchDuration,
      parameters.reserveRatioBips,
      parameters.delinquencyGracePeriod
    );
  }
```
Issue however is that the value of the reserve ratio is allowed to be equal to this _maximum_, see https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/MarketConstraintHooks.sol#L76-L114

```solidity

  function enforceParameterConstraints(
    uint16 annualInterestBips,
    uint16 delinquencyFeeBips,
    uint32 withdrawalBatchDuration,
    uint16 reserveRatioBips,
    uint32 delinquencyGracePeriod
  ) internal view virtual {
//snip
    assertValueInRange(
      reserveRatioBips,
      MinimumReserveRatioBips,
      MaximumReserveRatioBips,
      ReserveRatioBipsOutOfBounds.selector
    );
    assertValueInRange(
      delinquencyGracePeriod,
      MinimumDelinquencyGracePeriod,
      MaximumDelinquencyGracePeriod,
      DelinquencyGracePeriodOutOfBounds.selector
    );
  }
```
Which calls https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/MarketConstraintHooks.sol#L57-L69

```solidity
function assertValueInRange(
    uint256 value,
    uint256 min,
    uint256 max,
    bytes4 errorSelector
  ) internal pure {
    assembly {
      if or(lt(value, min), gt(value, max)) {
        mstore(0, errorSelector)
        revert(0, 4)
      }
    }
  }
```

If this happens however, this then means that the entire functionality is now redundant, as borrowers would not be able to withdraw any funds from the market. 

### Impact
As hinted under _Proof of Concept_, borrowers would not be able to withdraw any funds from the market. Additionally the market would fall into delinquency immediately after the start.
### Recommended Mitigation Steps

Consider having a lesser `BIP` max value for the reserve ratio, or better still introduce a different helper functionality that ensure the value of `reserveRatioBips` is strictly less than `MaximumReserveRatioBips`
## QA-02 Use a pull pattern instead of push when collecting fees


### Proof of Concept
First note that from the readMe, this has been stated: https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/README.md#L276

```markdown
| [Blocklists](https://github.com/d-xo/weird-erc20?tab=readme-ov-file#tokens-with-blocklists)                                                                | In scope    |
```


Now, take a look at https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarket.sol#L125-L137

```solidity
  function collectFees() external nonReentrant sphereXGuardExternal {
    MarketState memory state = _getUpdatedState();
    if (state.accruedProtocolFees == 0) revert_NullFeeAmount();

    uint128 withdrawableFees = state.withdrawableProtocolFees(totalAssets());
    if (withdrawableFees == 0) revert_InsufficientReservesForFeeWithdrawal();

    state.accruedProtocolFees -= withdrawableFees;
    asset.safeTransfer(feeRecipient, withdrawableFees);
    _writeState(state);
    emit_FeesCollected(withdrawableFees);
  }

```

This function helps withdraw available protocol fees to the fee recipient.

The current implementation uses an immutable `feeRecipient` address to collect fees:

```solidity

    asset.safeTransfer(feeRecipient, withdrawableFees);

```

If `feeRecipient` gets blacklisted by the asset used in the market, fee collection will fail for all associated markets.

### Impact
Multiple cases:
1. Loss of protocol revenue if `feeRecipient` is blacklisted.
2. Inability to collect fees from multiple markets simultaneously.
3. Increased risk with centralized or newly released assets, which are more prone to blacklisting.
4. Potential exploitation by borrowers to avoid fee payments by using assets likely to blacklist the `feeRecipient`.


### Recommended Mitigation Steps

Either make `feeRecipient` updatable per market or use a pull pattern instead of push when collecting fees, a pseudo fix could be:
   ```solidity
   function collectFees() external nonReentrant {
       // ... state updates and checks ...
       accumulatedFees[feeRecipient] += withdrawableFees;
       emit FeesCollected(withdrawableFees);
   }

   function withdrawFees() external {
       uint256 amount = accumulatedFees[msg.sender];
       accumulatedFees[msg.sender] = 0;
       asset.safeTransfer(msg.sender, amount);
   }
   ```



## QA-03 Borrowers would pay lesser APR in some supported assets

### Proof of Concept
First, per the readMe, we should assume rebasing tokens are in scope, see https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/README.md#L262

```markdown
| [Balance changes outside of transfers](https://github.com/d-xo/weird-erc20?tab=readme-ov-file#balance-modifications-outside-of-transfers-rebasingairdrops) | In scope    |
```

Where as this integration is welcome as the WIldMarketTokens themselves are somewhat rebasing in nature, this then means that users could pay lesser APR, which is because if they are used as underlying assets for markets, when the borrower/market contracts hold these tokens while they are lent, the newly accrued tokens may either be credited to the borrower, or inside the market itself, which in our case would count as the borrower adding liquidity. And result in the borrower _needing_ to pay a lower Annual Percentage Rate (APR) than initially set.

### Impact
Users would pay lesser APR for some to-be supported borrowable assets
### Recommended Mitigation Steps
Since disallowing rebasing assets is not an option, either track the balance change for these assets or heavily document this behaviour to users.
## QA-04 Amount are not scaled during approvals and allows for wrong amount to be transferred

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketToken.sol#L72-L91

```solidity
  function _transfer(address from, address to, uint256 amount, uint baseCalldataSize) internal virtual {
    MarketState memory state = _getUpdatedState();
    uint104 scaledAmount = state.scaleAmount(amount).toUint104();

    if (scaledAmount == 0) revert_NullTransferAmount();

    hooks.onTransfer(from, to, scaledAmount, state, baseCalldataSize);

    Account memory fromAccount = _getAccount(from);
    fromAccount.scaledBalance -= scaledAmount;
    _accounts[from] = fromAccount;

    Account memory toAccount = _getAccount(to);
    toAccount.scaledBalance += scaledAmount;
    _accounts[to] = toAccount;

    _writeState(state);
    emit_Transfer(from, to, amount);
  }

```
Evidently, during transfers we can see that there is a scaling of the amount being done.

Now this approach is not applied when approving tokens, see https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketToken.sol#L67-L70

```solidity
  function _approve(address approver, address spender, uint256 amount) internal virtual {
    allowance[approver][spender] = amount;
    emit_Approval(approver, spender, amount);
  }
```

This then means a step by step issue could arise as such:

> First keep in mind that the WildMarket token is rebasing
- User A makes the calculation on the amount of tokens they'd like to approve User B to be able to transfer [after taking the scaling factor into consideration](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketToken.sol#L74).
- User A arrives that the amount should be 100 `WMTokens`
User A approves User B 100 `WMTokens`.
- User B waits for some time for some rebase to occur.
- If positive then User B can then withdraw more than User A had approved them for.

### Impact
QA.

> TLDR: Amount approved != Amount transferred.
### Recommended Mitigation Steps
Consider scaling the amounts during approvals and transfers.
## QA-05 Borrowers would lose a lot of funds if market is intentionally/unintentionally frequently updated

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketBase.sol#L406-L465

```solidity
  function _getUpdatedState() internal returns (MarketState memory state) {
    state = _state;
    // Handle expired withdrawal batch
    if (state.hasPendingExpiredBatch()) {
      uint256 expiry = state.pendingWithdrawalExpiry;
      // Only accrue interest if time has passed since last update.
      // This will only be false if withdrawalBatchDuration is 0.
      uint32 lastInterestAccruedTimestamp = state.lastInterestAccruedTimestamp;
      if (expiry != lastInterestAccruedTimestamp) {
        (uint256 baseInterestRay, uint256 delinquencyFeeRay, uint256 protocolFee) = state
          .updateScaleFactorAndFees(
            delinquencyFeeBips,
            delinquencyGracePeriod,
            expiry
          );
          //snip
      }
          //snip
  }
```

This function is used to return the cached MarketState after accruing interest and delinquency / protocol fees and processing expired withdrawal batch, if any. Now going to the implementation of `updateScaleFactorAndFees()` that's called to increase the market's `scaleFactor` based on the annual interest rate, we see that the interest rate is not a fixed rate based on time, but rather, it compounds depending on how frequently `updateScaleFactorAndFees()` is called, see https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/libraries/FeeMath.sol#L140-L170

```solidity
  function updateScaleFactorAndFees(
    MarketState memory state,
    uint256 delinquencyFeeBips,
    uint256 delinquencyGracePeriod,
    uint256 timestamp
  )
    internal
    pure
    returns (uint256 baseInterestRay, uint256 delinquencyFeeRay, uint256 protocolFee)
  {
//..snip
    uint256 prevScaleFactor = state.scaleFactor;
    uint256 scaleFactorDelta = prevScaleFactor.rayMul(baseInterestRay + delinquencyFeeRay);

    state.scaleFactor = (prevScaleFactor + scaleFactorDelta).toUint112();
    state.lastInterestAccruedTimestamp = uint32(timestamp);
  }
```
This then means that if `updateState()` is called repeatedly, the final amount owed by the borrower in a market will be higher as compared to if `updateState()` was called once, see [this](https://www.investopedia.com/terms/c/compoundinterest.asp) for more info on hwo we get to this. So markets which have their state updated more frequently will have a slightly higher interest rate, which means the borrower will owe lenders slightly more assets. This easily occurs, if say the market is popular, and state-changing functions are always called or a lender even intentionally calls `updateState()` repeatedly.
### Impact
QA, this seems as a design choice, however this leads to a slight loss of funds for the borrower, and a slight gain for lenders.

### Recommended Mitigation Steps
In scope, consider  calling `updateScaleFactorAndFees()` after a certain time period has passed. This ensures that a lender cannot intentionally call `updateState()` repeatedly to inflate the interest rate within the chosen duration. 
## QA-06 Deposits are broken in an edge case

### Proof of Concept

First, per the readMe, we can see the below stated: https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/README.md#L243-L249


### General Questions


| Question                                | Answer                       |
| --------------------------------------- | ---------------------------- |
| ERC20 used by the protocol              | ERC-20s used as underlying assets for markets require no fee on transfer, `totalSupply` to be not at all close to 2^128, arbitrary mint/burn must not be possible, and `name`, `symbol` and `decimals` must all return valid results (for name and symbol, either bytes32 or a string). Creating markets for rebasing tokens breaks the underlying interest rate model.      |


This means that  the amount of assets that can be borrowed in a market should be up to `type(uint128).max`.

However whenever a lender calls `depositUpTo()` to deposit assets, the asset amount is scaled up and added to `scaledTotalSupply` which is limited to `toUint104`, see https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarket.sol#L55-L92

```solidity
  function _depositUpTo(
    uint256 amount
  ) internal virtual nonReentrant returns (uint256 /* actualAmount */) {
    // Get current state
    MarketState memory state = _getUpdatedState();

    if (state.isClosed) revert_DepositToClosedMarket();

    // Reduce amount if it would exceed totalSupply
    amount = MathUtils.min(amount, state.maximumDeposit());

    // Scale the mint amount
    uint104 scaledAmount = state.scaleAmount(amount).toUint104();//@audit
    if (scaledAmount == 0) revert_NullMintAmount();

    // ...snip
    // Increase supply
    state.scaledTotalSupply += scaledAmount;

    // Update stored state
    _writeState(state);

    return amount;
  }

```

As stated earlier on, this means that the maximum amount of assets that can be borrowed through a market is implicitly limited by `type(uint104).max * scaleFactor / 1e27`.

When a market is first deployed, its `scaleFactor` is `1e27`, which limits the maximum amount borrowable to `type(uint104).max` contrary to what's been stated in the [docs](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/README.md#L243-L249).

### Impact
Borrowers can't borrow more than `type(uint104).max` assets.
### Recommended Mitigation Steps

Increase the precision of `scaleFactor` to `uint128` instead. Alternatively, if this is intended then update the [docs](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/README.md#L243-L249).
## QA-07 Releasing escrowed funds might silently fail


### Proof of Concept

Take a look at https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatSanctionsEscrow.sol#L33-L44

```solidity

  function releaseEscrow() public override {
    if (!canReleaseEscrow()) revert CanNotReleaseEscrow();

    uint256 amount = balance();
    address _account = account;
    address _asset = asset;

    asset.safeTransfer(_account, amount);

    emit EscrowReleased(_account, _asset, amount);
  }
```

This function is used to released the escrowed balance of an address, issue however is that it does not return any value on failure/success. This would cause external integrators to not be able to track the success status.



### Impact
QA, however if this silently fails, this is even more problematic as a faux event would be emitted
### Recommended Mitigation Steps
Track the success of releasing the escrowed funds.

## QA-08 Apply some sort of AC to `WildcatMarketWithdrawals#executeWithdrawal()`

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketWithdrawals.sol#L194-L211

```solidity
  function executeWithdrawal(
    address accountAddress,
    uint32 expiry
  ) public nonReentrant sphereXGuardExternal returns (uint256) {
    MarketState memory state = _getUpdatedState();
    // Use an obfuscated constant for the base calldata size to prevent solc
    // function specialization.
    uint256 normalizedAmountWithdrawn = _executeWithdrawal(
      state,
      accountAddress,
      expiry,
      _runtimeConstant(0x44)
    );
    // Update stored state
    _writeState(state);
    return normalizedAmountWithdrawn;
  }

```

This function is used to execute a pending withdrawal request for a batch that has expired, evidently it lacks any form of access control whatsoever, which then means that anyone can call this function and then specify any lender as the `accountAddress` to withdraw assets to the lender's address.


Issue however is that the lender might not be in the position to receive the asset at the time, assume it's a smart contract wallet that's undergoing an upgrade and can't receive funds immediately.

### Impact
QA, as this somewhat seems as users fault, however lenders might not want their assets to be transferred to their address without their permission due to them being unable to receive funds temporarily, this can be bypassed in the current implementation of withdrawals.
### Recommended Mitigation Steps
Apply some sort of access control to `WildcatMarketWithdrawals#executeWithdrawal()` by using `msg.sender` as the lender's address instead:
```diff
  function executeWithdrawal(
-    address accountAddress,
    uint32 expiry
  ) public nonReentrant sphereXGuardExternal returns (uint256) {
    MarketState memory state = _getUpdatedState();
    // Use an obfuscated constant for the base calldata size to prevent solc
    // function specialization.
    uint256 normalizedAmountWithdrawn = _executeWithdrawal(
      state,
-     accountAddress,
+     msg.sender,
      expiry,
      _runtimeConstant(0x44)
    );
    // Update stored state
    _writeState(state);
    return normalizedAmountWithdrawn;
  }

```

## QA-09 Setters don't have equality checkers 

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/LenderStatus.sol#L54-L70

```solidity

  function setCredential(
    LenderStatus memory status,
    RoleProvider provider,
    uint256 timestamp
  ) internal pure {
    // User is approved, update status with new expiry and last provider
    status.lastApprovalTimestamp = uint32(timestamp);
    status.lastProvider = provider.providerAddress();
    status.canRefresh = provider.isPullProvider();
  }

  function unsetCredential(LenderStatus memory status) internal pure {
    status.canRefresh = false;
    status.lastApprovalTimestamp = 0;
    status.lastProvider = address(0);
  }
```


These functions are used to set/unset the credential status of a lender, issue however is that they never check to ensure the the value being set is not what's already stored.

### Impact
QA
### Recommended Mitigation Steps
Apply equality checkers.

## QA-10 transferFrom should only use allowance when  `spender != from`

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketToken.sol#L49-L66

```solidity
  function transferFrom(
    address from,
    address to,
    uint256 amount
  ) external virtual nonReentrant sphereXGuardExternal returns (bool) {
    uint256 allowed = allowance[from][msg.sender];

    // Saves gas for unlimited approvals.
    if (allowed != type(uint256).max) {
      uint256 newAllowance = allowed - amount;
      _approve(from, msg.sender, newAllowance);
    }

    _transfer(from, to, amount, 0x64);

    return true;
  }

```

This function is used for the popular implementation of transferfrom(), issue however is that the method always uses allowance even if `spender = from`. 

### Impact
QA, cause currently if `sender==from` when calling transferFrom function, it will also deduct allowances, which should not be deducted.

> NB: In many protocols only transferFrom is used, this design will break the compatibility with many protocols.
### Recommended Mitigation Steps
Only use allowance when `spender != from`.



## QA-11 Total supply would be out of sync for WMTokens

### Proof of Concept

Protocol has both underlying  and market tokens. Lenders deposit underlying tokens and get market tokens, which earn interest. When they want to withdraw their assets, they will burn their market tokens and get the underlying tokens back with some interest. All of the internal accounting is made with these market tokens and they are tracked as scaled balances. Market tokens are minted and burned and these actions affect scaledTotalSupply.

Market tokens can also be transferred between accounts like a regular ERC20 token. See https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketToken.sol#L41-L90

```solidity
  function transfer(
    address to,
    uint256 amount
  ) external virtual nonReentrant sphereXGuardExternal returns (bool) {
    _transfer(msg.sender, to, amount, 0x44);
    return true;
  }

  function _transfer(address from, address to, uint256 amount, uint baseCalldataSize) internal virtual {
    MarketState memory state = _getUpdatedState();
    uint104 scaledAmount = state.scaleAmount(amount).toUint104();

    if (scaledAmount == 0) revert_NullTransferAmount();

    hooks.onTransfer(from, to, scaledAmount, state, baseCalldataSize);

    Account memory fromAccount = _getAccount(from);
    fromAccount.scaledBalance -= scaledAmount;
    _accounts[from] = fromAccount;

    Account memory toAccount = _getAccount(to);
    toAccount.scaledBalance += scaledAmount;
    _accounts[to] = toAccount;

    _writeState(state);
    emit_Transfer(from, to, amount);
  }
```

Issue however is that transfers are allowed to the `0` address, which then means that the total supply should be reduced, since the tokens are _burnt_, issue however is that this is never done, causing for the total supply to be out of sync.

Note that per `WildcatMarketBase` we can make the conclusion that  internal and external balance tracking will be out of sync because of allowing transfer to zero addresses. That is by checking some functions that use total supply while calculating important parameters will now return wrong values. For e.g `applyProtocolFee()`, `liquidityRequired()`, and `totalDebts()`.

### Impact

Internal and external trackers to be out of sync, since this easily causes accounting issues. Parameters that uses `scaledTotalSupply` _(protocol fee, required liquidity etc)_ will be greater than the actual amount.
### Recommended Mitigation Steps
Either reduce tokens that gets sent to `0x0` by burning them or not allowing transfers to the `0x0` address.
## QA-12 Some assets can't be used as underlying 
### Proof of Concept

Take a look at https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketBase.sol#L295-L300

```solidity

  function totalAssets() public view returns (uint256) {
    return asset.balanceOf(address(this));
  }
```
This function is used to get the total balance in underlying asset and it does this by querying the `balanceOf()` method, issue however is that not all ERC20 support the `balanceOf()`, for eg Aura stash tokens, which stalls adoption and make protocol unusable with these tokens.   


### Impact
QA 
### Recommended Mitigation Steps

Query `balanceOf()` on a low level.
## QA-13 Valid deposits would be bricked in an edge case

### Proof of Concept

During deposits, there is an inconsistent behavior when handling deposits that exceed the maximum total supply. While the `depositUpTo()` method correctly limits the deposit amount to the available capacity, see https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarket.sol#L55-L65

```solidity
  function _depositUpTo(
    uint256 amount
  ) internal virtual nonReentrant returns (uint256 /* actualAmount */) {
    // Get current state
    MarketState memory state = _getUpdatedState();

    if (state.isClosed) revert_DepositToClosedMarket();

    // Reduce amount if it would exceed totalSupply
    amount = MathUtils.min(amount, state.maximumDeposit());
..snip
  }
```
 However, the `deposit()` function incorrectly reverts if the actual deposited amount doesn't match the requested amount:
   ```solidity
   function deposit(uint256 amount) external virtual {
     uint256 actualAmount = depositUpTo(amount);
     if (amount != actualAmount) {
       revert MaxSupplyExceeded();
     }
   }
   ```

This implementation leads to unexpected reverts even when partial deposits are successfully processed, i.e deposits that are exactly = to the maximum deposits

### Impact 

`maximumDeposit` might never actually be deposited since protocol expect users to guerss right down to the wei value how much is the maximum, cause anything heigher would always revert [here](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarket.sol#L119).
### Recommended Mitigation Steps

Remove the redundant check in the `deposit()` function, as `depositUpTo()` already handles the maximum supply limit:

```solidity
function deposit(uint256 amount) external virtual {
  uint256 actualAmount = depositUpTo(amount);
  // Remove the following check
  // if (amount != actualAmount) { 
  //   revert MaxSupplyExceeded();
  // }
}
```


## QA-14 Sanctioned Lenders Can Contaminate Market Funds

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarket.sol#L104-L108

```solidity
  function depositUpTo(
    uint256 amount
  ) external virtual sphereXGuardExternal returns (uint256 /* actualAmount */) {
    return _depositUpTo(amount);
  }
    function _depositUpTo(
    uint256 amount
  ) internal virtual nonReentrant returns (uint256 /* actualAmount */) {
    // Get current state
    MarketState memory state = _getUpdatedState();

    if (state.isClosed) revert_DepositToClosedMarket();

    // Reduce amount if it would exceed totalSupply
    amount = MathUtils.min(amount, state.maximumDeposit());

    // Scale the mint amount
    uint104 scaledAmount = state.scaleAmount(amount).toUint104();
    if (scaledAmount == 0) revert_NullMintAmount();

    // Cache account data and revert if not authorized to deposit.
    Account memory account = _getAccount(msg.sender);

    hooks.onDeposit(msg.sender, scaledAmount, state);

    // Transfer deposit from caller
    asset.safeTransferFrom(msg.sender, address(this), amount);

    account.scaledBalance += scaledAmount;
    _accounts[msg.sender] = account;

    emit_Transfer(_runtimeConstant(address(0)), msg.sender, amount);
    emit_Deposit(msg.sender, amount, scaledAmount);

    // Increase supply
    state.scaledTotalSupply += scaledAmount;

    // Update stored state
    _writeState(state);

    return amount;
  }
```

Evidently, the current implementation allows sanctioned lenders to still deposit funds under certain conditions, since there are no checks to see the accounts are sanctioned, that's to say:
1. When a sanctioned lender has never interacted with the market before.
2. When a lender becomes sanctioned after depositing and doesn't attempt to withdraw.



### Impact

Since there are no checks to see the accounts are sanctioned Wildcat allows sanctioned addresses to contaminate the entire market's funds, potentially exposing all lenders and borrowers to sanctions risk. Which undermines the system's `OFAC` compliance measures and could lead to severe legal and financial consequences for innocent participants.

### Recommended Mitigation Steps

Implement a sanction check in the `depositUpTo()` 

## QA-15 Market Settlements can be bricked

### Proof of Concept
First note that from the readMe, this has been stated: https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/README.md#L276

```markdown
| [Blocklists](https://github.com/d-xo/weird-erc20?tab=readme-ov-file#tokens-with-blocklists)                                                                | In scope    |
```


Now, take a look at https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarket.sol#L226-L288

```solidity
  function closeMarket() external onlyBorrower nonReentrant sphereXGuardExternal {
    MarketState memory state = _getUpdatedState();

    if (state.isClosed) revert_MarketAlreadyClosed();

    uint256 currentlyHeld = totalAssets();
    uint256 totalDebts = state.totalDebts();
    if (currentlyHeld < totalDebts) {
      // Transfer remaining debts from borrower
      uint256 remainingDebt = totalDebts - currentlyHeld;
      _repay(state, remainingDebt, 0x04);
      currentlyHeld += remainingDebt;
    } else if (currentlyHeld > totalDebts) {
      uint256 excessDebt = currentlyHeld - totalDebts;
      // Transfer excess assets to borrower
      asset.safeTransfer(borrower, excessDebt);
      currentlyHeld -= excessDebt;
    }
    hooks.onCloseMarket(state);
    state.annualInterestBips = 0;
    state.isClosed = true;
    state.reserveRatioBips = 10000;
    // Ensures that delinquency fee doesn't increase scale factor further
    // as doing so would mean last lender in market couldn't fully redeem
    state.timeDelinquent = 0;

    // Still track available liquidity in case of a rounding error
    uint256 availableLiquidity = currentlyHeld -
      (state.normalizedUnclaimedWithdrawals + state.accruedProtocolFees);

    // If there is a pending withdrawal batch which is not fully paid off, set aside
    // up to the available liquidity for that batch.
    if (state.pendingWithdrawalExpiry != 0) {
      uint32 expiry = state.pendingWithdrawalExpiry;
      WithdrawalBatch memory batch = _withdrawalData.batches[expiry];
      if (batch.scaledAmountBurned < batch.scaledTotalAmount) {
        (, uint128 normalizedAmountPaid) = _applyWithdrawalBatchPayment(
          batch,
          state,
          expiry,
          availableLiquidity
        );
        availableLiquidity -= normalizedAmountPaid;
        _withdrawalData.batches[expiry] = batch;
      }
    }

    uint256 numBatches = _withdrawalData.unpaidBatches.length();
    for (uint256 i; i < numBatches; i++) {
      // Process the next unpaid batch using available liquidity
      uint256 normalizedAmountPaid = _processUnpaidWithdrawalBatch(state, availableLiquidity);
      // Reduce liquidity available to next batch
      availableLiquidity -= normalizedAmountPaid;
    }

    if (state.scaledPendingWithdrawals != 0) {
      revert_CloseMarketWithUnpaidWithdrawals();
    }

    _writeState(state);
    emit_MarketClosed(block.timestamp);
  }

```

Evidently, the `closeMarket` function attempts to transfer assets to the borrower. Issue however is that if the borrower's address is blacklisted by the asset, these transfers will fail, preventing market closure.

### Impact

Disrupt the market settlement process by preventing proper market closure, i.e potentially locking funds in the contract indefinitely.


### Recommended Mitigation Steps

Implement a function to update the borrower address or better still consider implementing a withdrawal pattern for excess funds instead of direct transfers

## QA-16 Unauthorized actors can sidestep their restrictions and still integrate with Wildcat

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatArchController.sol#L245-L251

```solidity
  function registerControllerFactory(address factory) external onlyOwner {
    if (!_controllerFactories.add(factory)) {
      revert ControllerFactoryAlreadyExists();
    }
    _addAllowedSenderOnChain(factory);
    emit ControllerFactoryAdded(factory);
  }
```

This function is used to register a controller factory, this factory is being used in the window for authorization purposes, issue however is that  `ArchController` allows for authorizing `MarketControllerFactory` instances that point to _another `ArchController`_.


### Impact
Otherwise unauthorized actors would be allowed to perform operations i.e. change the protocol fee configuration or deploy controllers.
### Recommended Mitigation Steps
Consider adding a check in `ArchController.registerControllerFactory` to make sure that the to-be added `MarketControllerFactory` is correctly paired with it.




## QA-17 A borrower can remove their escrow address via `removeSanctionOverride()`

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatSanctionsSentinel.sol#L104-L107

```solidity
  function removeSanctionOverride(address account) public override {
    sanctionOverrides[msg.sender][account] = false;
    emit SanctionOverrideRemoved(msg.sender, account);
  }
```

This function is used to remove the sanction override of `account` for `borrower`, issue however is that this function allows the borrower to remove their sanction override address linked to them [when creating an escrow](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatSanctionsSentinel.sol#L120-L143) since no restriction is applied to only allow removal of the lender addresses instead.

### Impact
QA, impact here is quite minimal.
### Recommended Mitigation Steps
Do not allow borrowers to remove the sanction override for them and store them in the Sentinel instead.
## QA-18 Annual interest  & reserve ratio can still be changed for an already closed market

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketConfig.sol#L123-L163

```solidity
  function setAnnualInterestAndReserveRatioBips(
    uint16 _annualInterestBips,
    uint16 _reserveRatioBips
  ) external onlyBorrower nonReentrant sphereXGuardExternal {
    MarketState memory state = _getUpdatedState();
    if (state.isClosed) revert_AprChangeOnClosedMarket();

    uint256 initialReserveRatioBips = state.reserveRatioBips;

    (_annualInterestBips, _reserveRatioBips) = hooks.onSetAnnualInterestAndReserveRatioBips(
      _annualInterestBips,
      _reserveRatioBips,
      state
    );

    if (_annualInterestBips > BIP) {
      revert_AnnualInterestBipsTooHigh();
    }

    if (_reserveRatioBips > BIP) {
      revert_ReserveRatioBipsTooHigh();
    }

    if (_reserveRatioBips < initialReserveRatioBips) {
      if (state.liquidityRequired() > totalAssets()) {
        revert_InsufficientReservesForOldLiquidityRatio();
      }
    }
    state.reserveRatioBips = _reserveRatioBips;
    state.annualInterestBips = _annualInterestBips;
    if (_reserveRatioBips > initialReserveRatioBips) {
      if (state.liquidityRequired() > totalAssets()) {
        revert_InsufficientReservesForNewLiquidityRatio();
      }
    }

    _writeState(state);
    emit_AnnualInterestBipsUpdated(_annualInterestBips);
    emit_ReserveRatioBipsUpdated(_reserveRatioBips);
  }

```

This function is used to set the annual interest rate earned by lenders in bips, issue however is that knowing that after the closure of a market the rate is set to `0`, a borrower can still call this method and set the annual/interest rate to whatever.

### Impact
QA
### Recommended Mitigation Steps
Consider checking for market state within the `setAnnualInterestAndReserveRatioBips` function, i.e implement the `state.isClosed = true` check.

## QA-19 Admin could intentionally/unintentionally front-run borrower with the fee increasing 

### Proof of Concept

Protocol allows the borrower to deploy a new market and pay a fee to the protocol if it exists. This fee could be changed at any moment by admin.

Admin can accidentally/intentionally front-run borrowers the call to make a new market and set fee to bigger value, which borrower isn't expecting.


### Impact
QA, _Admin window_.
### Recommended Mitigation Steps
Consider adding a timelock to change fee parameters. This way, frontrunning will be impossible, and borrowers will know which fee they agree to.

## QA-20 Market list can easily get bloated

### Proof of Concept

See https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarket.sol#L226-L288

```solidity
  function closeMarket() external onlyBorrower nonReentrant sphereXGuardExternal {
    MarketState memory state = _getUpdatedState();

    if (state.isClosed) revert_MarketAlreadyClosed();

    //snip
    hooks.onCloseMarket(state);
    state.annualInterestBips = 0;
    state.isClosed = true;
    state.reserveRatioBips = 10000;
    // Ensures that delinquency fee doesn't increase scale factor further
    // as doing so would mean last lender in market couldn't fully redeem
    // ..snip
  }

```

This function is used to set the market APR to 0% and marks the market as closed, this also effectively transfers all funds back into the market and setting the `isClosed` parameter of the state to `true`.  While this prevents new lenders from depositing into the market, it only allows lenders to withdraw their funds and interest. The issue is that, once a borrower uses this function, the market _cannot_ be reopened. If the borrower wants to have another market for the same asset, they must deploy a new market.

### Impact
Markets list would get more and more bloated. 

### Recommended Mitigation Steps
Consider allowing borrwers to reset a market instead.
## QA-21 Potential Limitations of OpenZeppelin's EnumerableSet Usage

The protocol uses OpenZeppelin's EnumerableSet for managing markets, controller factories, borrowers, and controllers, see https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatArchController.sol#L16-L22

```solidity

  EnumerableSet.AddressSet internal _markets;
  EnumerableSet.AddressSet internal _controllerFactories;
  EnumerableSet.AddressSet internal _borrowers;
  EnumerableSet.AddressSet internal _controllers;
  EnumerableSet.AddressSet internal _assetBlacklist;

```





### Impact

1. Unordered Storage: EnumerableSet doesn't maintain insertion order. If the protocol needs to determine the chronological order of additions (e.g., which borrowers or markets were added first), this implementation may not provide accurate results.


2. Primary Use Case: EnumerableSet is primarily designed for efficient membership checks, not for maintaining ordered lists.

3. Gas Consumption: As the sets grow larger, operations become increasingly gas-intensive.

4. Size Limitations: Being based on Solidity arrays, these sets have an upper size limit.

> Also, using an Enumerable set can cause a Dos in the contract if the set grows large enough and it’s used in a function that modifies the state of the contract, this is commented in the openzeppelin documentation and it’s something to keep in mind for future iterations of the contracts.
### Recommended Mitigation Steps

N/A
## QA-22 Import declarations should import specific identifiers, rather than the whole file

### Proof of Concept

Multiple instances in scope, for example take a look at https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L4-L13

```solidity
import './libraries/LibERC20.sol';
import './interfaces/IWildcatArchController.sol';
import './libraries/LibStoredInitCode.sol';
import './libraries/MathUtils.sol';
import './ReentrancyGuard.sol';
import './interfaces/WildcatStructsAndEnums.sol';
import './access/IHooks.sol';
import './IHooksFactory.sol';
import './types/TransientBytesArray.sol';
import './spherex/SphereXProtectedRegisteredBase.sol';
```

Evidently, the imports being done is not name specific, but this is not the best implementation cause this could lead to polluting the symbol namespace.

### Impact

QA, albeit this could lead to the potential pollution of the symbol namespace and a slower compilation speed.

### Recommended Mitigation Steps

Consider using import declarations of the form `import {<identifier_name>} from "some/file.sol"` which avoids polluting the symbol namespace making flattened files smaller, and speeds up compilation (but does not save any gas).


## QA-23 Credentials would no longer be grant-able after a while

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/RoleProvider.sol#L37-L42

```solidity
  function calculateExpiry(
    RoleProvider provider,
    uint256 timestamp
  ) internal pure returns (uint256) {
    return timestamp.satAdd(provider.timeToLive(), type(uint32).max);
  }
```

This function helps calculate the expiry for a credential granted at `timestamp` by `provider`, this is done by adding its time-to-live to the timestamp and maxing out at the max uint32, indicating indefinite access.


Issue however is that protocol tries to implement this to have indefinite access with `uint32.max`, but this doesn't really mean infinite access, considering there is a definitive end to the expiry, see https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/libraries/MathUtils.sol#L70-L77

```solidity
  /**
   * @dev Saturation addition. Add `a` to `b` and return the result
   *      if it is less than `maxValue` or `maxValue` if it overflows.
   */
  function satAdd(uint256 a, uint256 b, uint256 maxValue) internal pure returns (uint256 c) {
    unchecked {
      c = a + b;
      return ternary(c < maxValue, c, maxValue);
    }
  }

```

This would mean that in a weird _low_ likelihood case that we want signatures over the timestamp of `4294967295` then it is not possible, due to the hardcoded implementation of `return timestamp.satAdd(provider.timeToLive(), type(uint32).max)`.
### Impact
QA, protocol still functions properly for a while since duration to `uint32.max` is still long.

Cause in that case credentials can only be expired since no timestamp is considered after that, see https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/LenderStatus.sol#L29-L35

```solidity
  function credentialExpired(
    LenderStatus memory status,
    RoleProvider provider
  ) internal view returns (bool) {
    return provider.calculateExpiry(status.lastApprovalTimestamp) < block.timestamp;
  }

```
### Recommended Mitigation Steps

## QA-24 `hasCredential` does not factor in expiration

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/LenderStatus.sol#L36-L38

```solidity
  function hasCredential(LenderStatus memory status) internal pure returns (bool) {
    return status.lastApprovalTimestamp > 0;
  }
```

This function is used to know if the lender has a credential, issue however is that expired credentials would also return true for this check, considering the check here is against the `lastApprovalTimestamp` being more than `0`, but when granting the credential the `lastApprovalTimestamp` is set to non-zero. 

### Impact
A lender with an expired credential would pass the has credential check.
### Recommended Mitigation Steps
Assume `expired credentials` to be `no credentials`

## QA-25 Remove linter errors from code

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/TransientBytesArray.sol#L1-L5

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.25;
import { Panic_ErrorSelector, Panic_ErrorCodePointer, Panic_InvalidStorageByteArray, Error_SelectorPointer, Panic_ErrorLength } from '../libraries/Errors.sol';

type TransientBytesArray is uint256;
```

This is the start of the `LibTransientBytesArray` library, hovering over the line we can see that there is a linter error:
> Linter: Line length must be no more than 100 but current length is 159. [max-line-length]

### Impact
QA
### Recommended Mitigation Steps
Make the line length less than 159
## QA-26 PUSH0 Opcode is not supported on all to-deploy chains

### Proof of Concept

Per the readMe protocol is to also deploy on multiple optimistic chains see:
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/README.md#L254

```markdown
| Chains the protocol will be deployed on | Ethereum, Base, Arbitrum, Polygon |
```

Now contracts are being compiled with versions higher than `0.8.20` see https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/TransientBytesArray.sol#L1-L5

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.25;
..snip
```

Issue however is that the PUSH0 opcode is not supported across all these chains.

Note that this version (and every version after 0.8.19) will use the PUSH0 opcode, which is still not supported on some EVM-based chains, for example Arbitrum. Consider using version 0.8.19 so that the same deterministic bytecode can be deployed to all chains. 


### Impact
QA
### Recommended Mitigation Steps


Use a different pragma version.
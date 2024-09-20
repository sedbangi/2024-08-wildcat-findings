## 1. `availableLiquidity > 0` check is not performed before the `_applyWithdrawalBatchPayment` function is called

The `WildcatMarket.closeMarket` function the `_processUnpaidWithdrawalBatch` function is called to process the unpaid withdrawal batches in the `_withdrawalData.unpaidBatches` FIFOQueue. In the `WildcatMarketWithdrawals._processUnpaidWithdrawalBatch` call the `_applyWithdrawalBatchPayment` function is called by providing the `availableLiquidity` as an input parameter.

But the issue is `availableLiquidity > 0` check is not performed here. But this `availableLiquidity > 0` check is performed prior to every other occassion where the `_applyWithdrawalBatchPayment` is called. 

Since the `_processUnpaidWithdrawalBatch` is called in a loop the `_applyWithdrawalBatchPayment` is also called in a loop and as a result of `availableLiquidity > 0` not being checked the trasnaction will continue loop through with the `availableLiquidity == 0` until the `unpaid batches index increment to the end`. In addition to that the `_processUnpaidWithdrawalBatch` function will always check the last batch which could not be fully paid off in the `_withdrawalData.unpaidBatches.first()` retrieval since the `_withdrawalData.unpaidBatches.shift()` is not called as there was not enough liquidity fully pay of the withdrawals of the batch.

This will result in undue wastage of gas and redundant execution. Hence it is recommended to update the logic used to settle the unpaid batches in the `WildcatMarketWithdrawals._processUnpaidWithdrawalBatch` function as shown below:

```solidity
    uint256 numBatches = _withdrawalData.unpaidBatches.length();
    uint256 i;
    while (i++ < numBatches && availableLiquidity > 0) { 
      // Process the next unpaid batch using available liquidity
      uint256 normalizedAmountPaid = _processUnpaidWithdrawalBatch(state, availableLiquidity); \
      // Reduce liquidity available to next batch
      availableLiquidity -= normalizedAmountPaid;
    } 
```

The above change will ensure that the `availableLiquidity > 0` check is performed before the `_applyWithdrawalBatchPayment` call is made such that the redundant execution is avoided thus saving on gas consumed.

https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarket.sol#L273-L279
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarketWithdrawals.sol#L324-L344

## 2. The `WildcatMarketWithdrawals.getUnpaidBatchExpiries` function will not return most upto date expiry timestamps of the unpaid batches

The `WildcatMarketWithdrawals.getUnpaidBatchExpiries` function is used to return the expiry timestamp of every unpaid withdrawal batch. But this will not include the current withdrawal batch if it has expired and falls under the `unpaid batch category`. This happens since the `MarketState` is not updated before the `_withdrawalData.unpaidBatches.values()` is called. 

But in the `WildcatMarketWithdrawals.getWithdrawalBatch` and `WildcatMarketWithdrawals.getAvailableWithdrawalAmount` functions the current batch is updated for the latest `scaleFactor` and `latest batch parameters` by calling the `_calculateCurrentState()` function.

Hence the `WildcatMarketWithdrawals.getUnpaidBatchExpiries` will not return the most upto date `unpaid batches expiries` if the current expired withdrawal batch is an `unpaidBatch`. Hence if a user uses the output of the `WildcatMarketWithdrawals.getUnpaidBatchExpiries` function to determine the `expiries` of all the `unpaid batches` and uses the output in a `wildcat protocol` transaction, it will result in unexpected behaviour since it does not use the most upto date `expiries` of the `unpaid batches`.

Hence it is recommended to update the `WildcatMarketWithdrawals.getUnpaidBatchExpiries` function to call the `_calculateCurrentState()` function before calling the `_withdrawalData.unpaidBatches.values()` function and return the `expiry` if the `current batch is expired and is unpaid`. Hence the return values of the `_calculateCurrentState()` function will have to updated to accomadate the above change. If the `current pending withdrawal batch` is expired and unpaid then this can be added to the return value of hte `_withdrawalData.unpaidBatches.values()` call. This will ensure that most upto date `expiries` will be returned by the `WildcatMarketWithdrawals.getUnpaidBatchExpiries` function.

https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarketWithdrawals.sol#L22-L24

## 3. Potential Exploit in rescueTokens function Due to Borrower-Controlled Underlying Asset

Vulnerability Description:
The `rescueTokens` function in `WildcatMarket.sol` is designed to allow the borrower to recover tokens accidentally sent to the market contract. However, the borrower has control over which `asset token` is set as the `underlying asset` for the market. This creates an opportunity for the borrower to maliciously set a token which has two different addresses as the underlying token. 

The critical line of code checks whether the token to be rescued is either the underlying asset or the contract's own address as shown below:

```solidity
if ((token == asset).or(token == address(this))) {
    revert_BadRescueAsset();
}
```

However, since the underlyign token has two addresses, the malicious borrower can use the second address (not the address assigned to the state variable `asset`) as the `token` and call the `WildcatMarket.rescueTokens` function. This will bypass the safety check and allow the borrower to transfer the underlying asset from the `WildcatMarket` contract.

Impact:
By using an underlying token with two addresses, the borrower can exploit the `rescueTokens function` to transfer the entire underlying token balance deposited by the lenders to the malicious borrower himself. This opens up the protocol to a significant exploit, where a malicious borrower could essentially drain tokens from the market, bypassing normal operational checks.

This issue is further prevalent since only a `black list` is used to control the `underlying assets` which can be used to deploy markets.

Proof of Concept (PoC):

The vulnerability lies in the following code in WildcatMarket.sol:

```solidity
function rescueTokens(address token) external onlyBorrower {
    if ((token == asset).or(token == address(this))) {
        revert_BadRescueAsset();
    }
    token.safeTransferAll(msg.sender);
}
```

Recommended Fix:
To prevent this exploit, it is recommended to enable a whitelist of tokens in the `wildcat protocol` and only allow the borrowers to select tokens from this `whitelist` to deploy markets.

https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarket.sol#L37-L42

## 4. Lack of onlyBorrower Modifier in WildcatMarket.repay Function                      

Vulnerability Description: 
In the `WildcatMarket.repay` function, a NatSpec comment advises that this function should only be used by the borrower and not by lenders or unrelated third parties:

However, the function does not include the onlyBorrower modifier to enforce this restriction, leaving the function open to being called by any account. This could result in unintended behavior where lenders or third parties mistakenly or maliciously attempt to repay on behalf of the borrower, potentially leading to mismanagement of funds or unintended repayment actions.

Impact: 
Without the onlyBorrower modifier, there is a risk that unrelated parties could interact with the repay function, leading to:

Unauthorized repayment attempts by lenders or third parties.
Potential for confusion or errors in handling repayments.


Proof of Concept (PoC): 
Currently, the repay function does not restrict its caller:

```solidity
function repay(uint256 amount) external nonReentrant sphereXGuardExternal {
    if (amount == 0) revert_NullRepayAmount();

    asset.safeTransferFrom(msg.sender, address(this), amount);
    emit_DebtRepaid(msg.sender, amount);

    MarketState memory state = _getUpdatedState();
    if (state.isClosed) revert_RepayToClosedMarket();
    
    hooks.onRepay(amount, state, _runtimeConstant(0x24));
    _writeState(state);
}
```

Recommended Fix: 
To ensure that only the borrower can call the repay function, the onlyBorrower modifier should be added:

```solidity
function repay(uint256 amount) external onlyBorrower nonReentrant sphereXGuardExternal {
    if (amount == 0) revert_NullRepayAmount();

    asset.safeTransferFrom(msg.sender, address(this), amount);
    emit_DebtRepaid(msg.sender, amount);

    MarketState memory state = _getUpdatedState();
    if (state.isClosed) revert_RepayToClosedMarket();
    
    hooks.onRepay(amount, state, _runtimeConstant(0x24));
    _writeState(state);
}
```

This change ensures that only the borrower is able to initiate the repayment process, aligning the function's behavior with its intended usage.

https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarket.sol#L193-L202

## 5. Incorrect `Base Size` parameter Used in `HooksConfig.onBorrow` Calldata `Size` Calculation

In the `HooksConfig.onBorrow` function the `size` of the calldata for the `hooks.onBorrow` function call is calculated as follows:

```solidity
        let size := add(RepayHook_Base_Size, extraCalldataBytes)
```

As you could see the wrong base size value (`RepayHook_Base_Size`) is used. The correct value `BorrowHook_Base_Size` should have been used in its place.

Hence it is recommended to update the `calldata size calcualtion` as shown below:

```solidity
        let size := add(BorrowHook_Base_Size, extraCalldataBytes)
```

https://github.com/code-423n4/2024-08-wildcat/blob/main/src/types/HooksConfig.sol#L533

## 6. Free Memory Pointer Not Updated at the end of the  `HooksConfig.onRepay`, `HooksConfig.onBorrow` functions

Vulnerability Description:

In the HooksConfig contract, the assembly blocks within functions such as onRepay and onBorrow do not update the free memory pointer at the end of the block. The free memory pointer is used by the EVM to track the location of free memory space, and failing to update it could potentially lead to issues if subsequent operations expect the pointer to be accurate.

Currently, this does not seem to cause an issue within the protocol's existing structure. However, this practice might introduce problems if the protocol is extended or modified in the future. Code that relies on accurate memory management could encounter unexpected behavior, such as memory corruption, overwriting, or vulnerabilities, as new functions may assume an incorrect free memory location.

Impact:

While the immediate impact is low, failing to update the free memory pointer may create risks in future versions or extensions of the protocol. As the protocol evolves, functions depending on proper memory management may exhibit erratic behavior or cause vulnerabilities if the free memory pointer is not correctly maintained.

Proof of Concept (PoC):

In the onRepay and onBorrow functions, the free memory pointer is loaded at the beginning of the assembly block but is not updated at the end:


assembly {
    let freeMemoryPointer := mload(0x40)
    ...
    // Free memory pointer is not updated
}
As a result, subsequent code that relies on the correct free memory location could behave unexpectedly.

Recommended Fix:

To mitigate this risk, update the free memory pointer at the end of the assembly blocks in the onRepay, onBorrow functions. The pointer should be stored back into the appropriate memory location (0x40) after any memory allocations to ensure that future operations are aware of the correct location of free memory:


assembly {
    let freeMemoryPointer := mload(0x40)
    ...
    // Update the free memory pointer at the end
    mstore(0x40, add(freeMemoryPointer, /*memory-used*/))
}
This ensures that memory is managed correctly, reducing the risk of future vulnerabilities.

https://github.com/code-423n4/2024-08-wildcat/blob/main/src/types/HooksConfig.sol#L509-L538
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/types/HooksConfig.sol#L562-L588

## 7. Wrong Natspec comments in the `HooksConfig` contract 

The following natspec comment is wrong.

```solidity
  // Size of lender + scaledAmount + state + extraData.offset + extraData.length
  uint256 internal constant DepositHook_Base_Size = 0x0244;
```

The `DepositHook_Base_Size` includes the `4 bytes` of the `IHooks.onDeposit.selector` function selector as well. But it is not referenced in the above natspec comment. Hence the above natspec comment should be updated accordingly.

```solidity
  // Size of onDeposit.selector + lender + scaledAmount + state + extraData.offset + extraData.length
  uint256 internal constant DepositHook_Base_Size = 0x0244;
```

Same issue exist in the `natspec comments` for the `RepayHook_Base_Size` value declaration.

https://github.com/code-423n4/2024-08-wildcat/blob/main/src/types/HooksConfig.sol#L255-L256
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/types/HooksConfig.sol#L498-L499
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/types/HooksConfig.sol#L546-L547

## 8. Wrong natspec comments in the `MarketState.borrowableAssets` function

The `natspec comments` of the `MarketState.borrowableAssets` function is errorneous. It does not state the `inclusion of the accrued protocol fees` as shown below:

```
  /**
   * @dev Returns the amount of underlying assets that can be borrowed.
   *
   *      The borrower must maintain sufficient assets in the market to
   *      cover 100% of pending withdrawals, 100% of previously processed
   *      withdrawals (before they are executed), and the reserve ratio
   *      times the outstanding debt (deposits not pending withdrawal).
   *
   *      Any underlying assets in the market above this amount can be borrowed.
   */
```

Hence the above natspec comment should be updated to mention the `inclusion of the accrued protocol fees` when accounting for the `state.liquidityRequired` amount.

https://github.com/code-423n4/2024-08-wildcat/blob/main/src/libraries/MarketState.sol#L113-L122

## 9. A malicious lender `DoS` a borrower from borrowing from the market

A malicious lender `DoS` a borrower from borrowing from the market. Let's consider the following scenario:

1. Malicious lender deposits a large amount of tokens into the market.
2. He then waits for teh borrower to initiate a `borrow` transaction.
3. lender front-runs the `borrow` transaction with a withdrawal and makes the `state.scaledPendingWithdrawals` larger thus the `liquidityRequired` is increased.
4. The `borrowable` amount is now less.
5. The `if (amount > borrowable) revert_BorrowAmountTooHigh();` condition evaluates to true and the transaction reverts.

Hence recommended to enable a `limit` on how much an individual lender can deposit into the market in addition to the overall `maxTotalSupply` of the market.

https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarket.sol#L157-L158

## 10. Redundant casting to `address` data type

In the `WildcatMarketBase._createEscrowForUnderlyingAsset` function there are redundant casting of address state variables to `address` again.

```solidity
    address tokenAddress = address(asset);
    ...
    address sentinelAddress = address(sentinel);    
```

Recommended to assign the address state variables directly to the address local variables without redundant casting as shown below:


```solidity
    address tokenAddress = asset;
    ...
    address sentinelAddress = sentinel;    
```

https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarketBase.sol#L772
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarketBase.sol#L774

## 11. The `WildcatMarketWithdrawals.getWithdrawalBatch` function can be optimized
 
The `WildcatMarketWithdrawals.getWithdrawalBatch` function has the following logic to retrive the `withdrawal batch` from the `_withdrawalData`:

```solidity
    WithdrawalBatch storage _batch = _withdrawalData.batches[expiry];
    batch.scaledTotalAmount = _batch.scaledTotalAmount;
    batch.scaledAmountBurned = _batch.scaledAmountBurned;
    batch.normalizedAmountPaid = _batch.normalizedAmountPaid;
```

Above code snippet executes redundant state variable reads which can be omitted and the code can be optimized as shown below:

```solidity
    batch = _withdrawalData.batches[expiry];
```

https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarketWithdrawals.sol#L34-L37

## 12. No fund recovery mechanism for external tokens, in the `WildcatSanctionsEscrow` contract

The `WildcatSanctionsEscrow` contract is used to temporarily hold the funds of the sanctioned `lenders` until they are released from the sanctions. This contract allows anyone to recover the funds to the `lender,s` account the underlying tokens after the `lender` is released from the sanctions.

But there is no recovery function in the `WildcatSanctionsEscrow` contract for other tokens which could be mistakenly deposited to the `WildcatSanctionsEscrow` contract. But similar functinality is currently implemented in the `WildcatMarket.rescueTokens` contract to withdraw the mistakenly deposited funds to the `WildcatMarket` contract.

Hence it is recommended to add a recovery function in the `WildcatSanctionsEscrow` contract to withdraw the mistakenly deposited funds to the `WildcatSanctionsEscrow` contract. And the `borrower` should be allowed to call this function to retrive these funds to  his account similart to how it is done in the `WildcatMarket.rescueTokens`.

https://github.com/code-423n4/2024-08-wildcat/blob/main/src/WildcatSanctionsEscrow.sol#L34-L44
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarket.sol#L37-L42

## 13. Address collission in the `escrow address` could lead to loss of funds of the sanctioned lender

The `WildcatSanctionsSentinel.getEscrowAddress` function is used to calculate the address of the escrow contract of the sanctioned lender. When address is derived via `CREATE2` the truncation of the `bytes32` to bytes20 happesn thus the least significant 20 bytes are taken as the `escrow contract address`.

The `WildcatSanctionsSentinel.createEscrow` function is used to deploy the escrow contract. In this fucntion if the computed `escrow contract address` already has contract logic deployed the trasnaction will return the `escrow contract address` without any further checks.

But there is a possibitliy of `address collissioin in CREATE2` address derivation. Now let's consider the following scenario:

1. The `WildcatSanctionsSentinel.createEscrow` is called to create a new escrow contract for a sanctioned lender.
2. The derived address via CREATE2 is `Address A`.
3. But there is already a contract deployed at `Address A` (Address collision).
4. The following check is performed :

```solidity
    if (escrowContract.code.length != 0) return escrowContract;
```

Since there is already code deployed in this contract address the `escrowContract` will be returned without any further checks.

5. The `WildcatMarketWithdrawals._executeWithdrawal` function will transfer the `funds of the sanctioned lender` to the `escrow contract address` which is owned by a different user. 

6. As a result the `sanctioned lender` will permenantly lose his funds.

Even though the address collssion are extremely rare there is still a possibility of it happening and if it happens the `sanctioned lender` will permenantly lose his funds thus incurring loss of funds.

Hence if the calcluated `escrow contract` is already deployed it is recommended to check the parameters in the `already deployed escrow contract` actually match the input parameters (borrower, account, asset) of the deploying transaction. If they are the same then returning the `escrow contract address` is fine. Else need to deploy a new escrow contract. For this the `salt value derivation logic` will have to be modified to accomadate a `nonce` value which can be changed.

https://github.com/code-423n4/2024-08-wildcat/blob/main/src/WildcatSanctionsSentinel.sol#L169
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/WildcatSanctionsSentinel.sol#L129

## 14. Function selectors are hardcoded in the low-level assembly call fucntions

The `WildcatMarketBase` contract calls the `WildcatSanctionsSentinel` contract for different parameter validation checks such as in the `WildcatMarketBase._isFlaggedByChainalysis` and `WildcatMarketBase._isSanctioned` function. 

But when these low level `calls` are made via assembly, to the `sentinel` contract the function selector is hardcoded as shown below:

```solidity
      mstore(0, 0x06e74444) 
```

Hence if the function selectors are changed in the future due to addtion of a new parameter this could lead to unexpected behaviour in the `WildcatMarketBase` contract and the protocol as a whole.

Hence it is recommended get the function selector via the `interface of the contract` and calling the `function.selector` of the respective function in the corresponding interface.

https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarketBase.sol#L259
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/market/WildcatMarketBase.sol#L757
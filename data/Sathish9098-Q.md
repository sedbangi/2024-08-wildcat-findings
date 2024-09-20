# QA Report

##

| Issue Count | Issue Title                                                                |
|-------------|-------------------------------------------------------------------------------------------|
| [L-1]       | Borrower Address ``Blocklisting`` by ``Tokens`` Preventing ``Market Close``                       |
| [L-2]       | Risk of Lenders Deposit Reverts Due to Sudden Decrease in `maxTotalSupply`                |
| [L-3]       | Front-Running Risk in ``Multi-Lender`` Deposit Process Leading to ``Transaction Reverts``         |
| [L-4]       | Deterministic Contract Address Generation and Replay Attack Exposure in `create` Deployment |
| [L-5]       | Ability to Update Deprecated ``hooksTemplate`` Poses Security and Lifecycle Management Risks                                    |
| [L-6]       | ``Governance`` Conflict in Controller Management Due to Separate ``Addition`` and ``Removal Authorities`` |
| [L-7]       | ``Lack of Validation`` for Critical ``Market Parameters`` Leads to Potential Misconfiguration     |
| [L-8]       | Potential ``Zero-Word`` Issue for Strings Exactly ``32 Bytes`` Long                               |
| [L-9]       | Panic on Length Mismatch in `extractByteArrayLength`                                      |
| [L-10]      | Unchecked `keccak256` and `for` Loop in `readToPointer`                                  |
| [L-11]      | Potential for Indirect Reentrancy in `updateSphereXEngineOnRegisteredContracts` function  |
| [L-12]      | Lack of ``contract existence`` check when using ``assembly calls``                                |
| [L-13]      | Redundant `address(0)` Validation                                                         |
| [L-14]      | Brittle and Inefficient Assembly-based Custom Error Handling                              |
| [L-15]      | Unrestricted Gas Forwarding in External Calls Leading to Potential ``Out-of-Gas`` Errors      |
| [L-16]      | Potential for `` DoS`` by returning overly large arrays                                        |
| [L-17]      | Lack of input validations when assigning values to ``immutable variables``                    |
| [L-18]      | Race Condition on Market Initialization in `_deployMarket`                                |
| [L-19]      | Logical Mistake in `rescueTokens()` function Conditional Check                            |
| [L-20]      | `WithdrawalBatchExpired` Event Emitted even if not Processing `WithdrawalBatchPayment()`   |
| [L-21]      | Lack of ``reentrancy modifiers`` in critical functions `depositUpTo()`, `deposit()`            |
| [L-22]      | `collectFees()` function not followed the CEI Pattern                                     |
| [L-23]      | No Fallback Function for ``WildcatSanctionsEscrow`` contract                                  |
| [L-24]      | Risk of Failed ``Escrow`` Release Due to ``Blocklisted`` Recipient Address                        |
| [L-25]      | `getAvailableWithdrawalAmount()` function potential to Dust Accumulation from Rounding Errors |
| [L-26]      | Inefficient Processing of Withdrawal Batches Due to Missing ``Available Liquidity Check``     |
| [L-27]      | Inability to Remove Deprecated Hook Templates from ``_hooksTemplates`` Array    |
| [L-28]      | Sanctioned ``Borrowers`` Allowed to Deploy New Markets, Violating ``Protocol Compliance Guidelines``     |



##

## [L-1] Borrower Address Blocklisting by Tokens Preventing Market Shutdown

In this closeMarket function, the borrower is responsible for closing the market, which involves reconciling debts and transferring excess assets to or from the borrower. However, if the borrower's address is blocklisted by a token (such as USDC or USDT), the market cannot be closed successfully.

- If the borrower owes more than the contract currently holds (currentlyHeld < totalDebts), the borrower is required to repay the remaining debt.
The _repay() function will be called to transfer assets from the borrower to the contract.

- If the contract holds more assets than the total debts (currentlyHeld > totalDebts), the excess assets are transferred back to the borrower using asset.safeTransfer(borrower, excessDebt).

If the borrower's address is blocklisted, the token contract will prevent the transfer to the borrower. As a result, the excess assets cannot be transferred to the borrower, which will prevent the market from closing successfully.


```solidity
FILE: 2024-08-wildcat/src/market/WildcatMarket.sol

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
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarket.sol#L241

### Recommended Mitigation
Allow the borrower to specify an alternative address that is not blocklisted, so that debt repayment and asset transfers can proceed even if their primary address is blocklisted.

##

## [L-2] Risk of Lenders Deposit Reverts Due to Sudden Decrease in ``maxTotalSupply``

If the borrower reduces maxTotalSupply while there are pending deposit transactions (either in the mempool or currently executing), this could lead to deposits failing to be processed successfully.
Specifically, if the maxTotalSupply is reduced below the current totalSupply or to a point where a pending deposit would now exceed the limit, the deposit will revert.

### Example Scenario:
Before Update:

maxTotalSupply: 1,000 tokens
scaledTotalSupply: 900 tokens
Lender A wants to deposit 50 tokens (this is valid, as the new total supply would be 950, which is less than maxTotalSupply).
Borrower Action: The borrower calls setMaxTotalSupply() and reduces maxTotalSupply to 920 tokens.

Pending Deposit Reverts:

When Lender A’s deposit is processed, the new total supply would be 950 tokens (900 existing + 50 new), but the maxTotalSupply is now only 920 tokens, which would cause the deposit to revert.

```solidity
FILE:2024-08-wildcat/src/market/WildcatMarketConfig.sol

 function setMaxTotalSupply(
    uint256 _maxTotalSupply
  ) external onlyBorrower nonReentrant sphereXGuardExternal {
    MarketState memory state = _getUpdatedState();
    if (state.isClosed) revert_CapacityChangeOnClosedMarket();

    hooks.onSetMaxTotalSupply(_maxTotalSupply, state);
    state.maxTotalSupply = _maxTotalSupply.toUint128();
    _writeState(state);
    emit_MaxTotalSupplyUpdated(_maxTotalSupply);
  }
```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketConfig.sol#L101-L111

### Recommended Mitigation
Introduce a grace period or buffer during which the maxTotalSupply cannot be reduced by a large amount immediately. 

##

## [L-3] Front-Running Risk in Multi-Lender Deposit Process Leading to Transaction Reverts

Multiple lenders are allowed to deposit into the same market. The _depositUpTo function allows lenders to deposit underlying assets, and the amount of deposit is constrained by the maxTotalSupply, which is the configured maximum allowable supply of market tokens. The problem arises when multiple lenders attempt to deposit simultaneously or in quick succession, which could lead to one lender front-running another, potentially causing a revert.

### Front-Running Scenario (Lender B Front-Runs Lender A)
#### Problem:
- Multiple lenders (Lender A and Lender B) are trying to deposit into the same market, but the available space (i.e., maxTotalSupply - scaledTotalSupply) is limited.
- Suppose Lender A submits a transaction to deposit an amount X when the remaining available deposit is Y (where Y >= X).
- While Lender A's transaction is pending (in the mempool), Lender B detects this and front-runs Lender A by submitting their deposit transaction with an equal or larger gas price.
- Lender B's transaction gets processed first, and if Lender B deposits a large amount (say the entire available space Y), then there’s no room left for Lender A's transaction when it gets processed.
#### Result:
- When Lender A's transaction is executed after Lender B's deposit, the available space (state.maximumDeposit()) might have been reduced to 0, or a much smaller amount than Lender A intended to deposit.
- This causes Lender A's transaction to revert, because the scaled amount that would be minted for Lender A would now be 0, leading to a revert_NullMintAmount().

The deposit process as written is not inherently FIFO (First In, First Out). The order of execution of transactions is determined by the blockchain’s transaction confirmation process, which means that transactions with higher gas prices can be confirmed faster, allowing for front-running. In this case, Lender B can "jump the queue" by paying a higher gas fee, which leads to Lender A’s transaction failing if the available space is reduced.

```solidity
FILE: 2024-08-wildcat/src/market/WildcatMarket.sol

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
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarket.sol#L55C3-L91C4

### Recommended Mitigation
Implement a reservation mechanism where each lender can reserve the available deposit space before transferring funds. This ensures that once a lender commits to a deposit, the space is locked, preventing others from front-running.

##

## [L-4] Deterministic Contract Address Generation and Replay Attack Exposure in ``create`` Deployment Mechanism

### Impact

``Create`` uses the deployer’s address and current nonce to calculate the new contract’s address. This is where the issues of address predictability and replay attacks come into play.

The address of the newly deployed contract is calculated as

The deployer's address (the contract or account executing this code).
The nonce of the deployer, which is the number of contracts they have already deployed.

In a replay attack, the same contract creation transaction could be executed on another network or a fork of the blockchain where the deployer’s nonce is the same. Since create uses only the deployer’s address and nonce to calculate the address, this could result in the contract being deployed at the same address across different networks or forks.

#### Problems
 - Since the nonce is incremented with each contract deployment, anyone who knows the deployer's address and current nonce can predict the address of the contract that will be deployed next. 
 - If the deployed contract interacts with network-specific assets (like tokens or governance functions), it could cause unintended behavior, such as duplicate contract instances interacting with the same external systems.

```soldiity
FILE: 2024-08-wildcat/src/HooksFactory.sol

hooksInstance := create(0, initCodePointer, initCodeSizeWithArgs)

```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L319

### Recommended Mitigation
By implementing ``create2``, your contract deployment process becomes more secure, protecting against both front-running and replay attacks.

The salt is a value you choose, and it allows you to control the address deterministically but unpredictably.

```solidity
hooksInstance := create2(0, initCodePointer, initCodeSizeWithArgs, someUniqueSalt)

```

##

## [L-5] Ability to Update Deprecated ``hooksTemplate`` Poses Security and Lifecycle Management Risks

In the current implementation, there are no checks ensuring that the hooksTemplate being updated is either a deployed or valid template. As per the documentation, it’s possible to update template attributes like fees even when the ``_templateDetails[hooksTemplate]``.enabled flag is set to false (i.e., the template is deprecated or disabled). 

The contract does not check whether the hooksTemplate being referenced is ``deployed`` or ``active`` before allowing updates to its attributes, such as fees or configurations.

This means deprecated or disabled templates (where _templateDetails[hooksTemplate].enabled == false) can still have their attributes modified, despite not being in use.



```solidity
FILE: 2024-08-wildcat/src/HooksFactory.sol

/// @dev Update the fees for a hooks template
  /// Note: The new fee structure will apply to all NEW markets created with existing
  ///       or future instances of the hooks template, and the protocol fee can be pushed
  ///       to existing markets using `pushProtocolFeeBipsUpdates`.
  function updateHooksTemplateFees(
    address hooksTemplate,
    address feeRecipient,
    address originationFeeAsset,
    uint80 originationFeeAmount,
    uint16 protocolFeeBips
  ) external override onlyArchControllerOwner {
    if (!_templateDetails[hooksTemplate].exists) {
      revert HooksTemplateNotFound();
    }
    _validateFees(feeRecipient, originationFeeAsset, originationFeeAmount, protocolFeeBips);
    HooksTemplate storage template = _templateDetails[hooksTemplate];
    template.feeRecipient = feeRecipient;
    template.originationFeeAsset = originationFeeAsset;
    template.originationFeeAmount = originationFeeAmount;
    template.protocolFeeBips = protocolFeeBips;
    emit HooksTemplateFeesUpdated(
      hooksTemplate,
      feeRecipient,
      originationFeeAsset,
      originationFeeAmount,
      protocolFeeBips
    );
  }

  function disableHooksTemplate(address hooksTemplate) external override onlyArchControllerOwner {
    if (!_templateDetails[hooksTemplate].exists) {
      revert HooksTemplateNotFound();
    }
    _templateDetails[hooksTemplate].enabled = false;
    // Emit an event to indicate that the template has been removed
    emit HooksTemplateDisabled(hooksTemplate);
  }

```

### Recommended Mitigation 
Modify the logic only active hooks hooks templates possible to modify instead of allowing deprecated hooks template also to modify 

##

## [L-6] Governance Conflict in Controller Management Due to Separate Addition and Removal Authorities

The ControllerFactory could add a controller that introduces vulnerabilities or unwanted features, and the owner may not be able to remove it quickly enough to prevent damage. During the time it takes for the owner to remove the controller, the system could be exposed to exploitation.

```solidity
FILE:2024-08-wildcat/src
/WildcatArchController.sol

function registerController(address controller) external onlyControllerFactory {
    if (!_controllers.add(controller)) {
      revert ControllerAlreadyExists();
    }
    _addAllowedSenderOnChain(controller);
    emit ControllerAdded(msg.sender, controller);
  }

  function removeController(address controller) external onlyOwner {
    if (!_controllers.remove(controller)) {
      revert ControllerDoesNotExist();
    }
    emit ControllerRemoved(controller);
  }

```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatArchController.sol#L296-L309

### Recommended Mitigation
Add the onlyOwner access control for both registerController() and removeController() functions

##

## [L-7] Lack of Validation for Critical Market Parameters Leads to Potential Misconfiguration

The issue here involves missing validation checks for critical parameters when deploying new markets, specifically for parameters like parameters.maxTotalSupply, parameters.withdrawalBatchDuration, parameters.reserveRatioBips, and parameters.delinquencyGracePeriod. These parameters likely play an important role in the functionality and behavior of the new market, and failing to validate them properly before deployment introduces several risks, such as misconfiguration, abuse, or security vulnerabilities.

```solidity
FILE:2024-08-wildcat/src/HooksFactory.sol

 TmpMarketParameterStorage memory tmp = TmpMarketParameterStorage({
      borrower: msg.sender,
      asset: parameters.asset,
      packedNameWord0: bytes32(0),
      packedNameWord1: bytes32(0),
      packedSymbolWord0: bytes32(0),
      packedSymbolWord1: bytes32(0),
      decimals: decimals,
      feeRecipient: templateDetails.feeRecipient,
      protocolFeeBips: templateDetails.protocolFeeBips,
      maxTotalSupply: parameters.maxTotalSupply,
      annualInterestBips: parameters.annualInterestBips,
      delinquencyFeeBips: parameters.delinquencyFeeBips,
      withdrawalBatchDuration: parameters.withdrawalBatchDuration,
      reserveRatioBips: parameters.reserveRatioBips,
      delinquencyGracePeriod: parameters.delinquencyGracePeriod,
      hooks: parameters.hooks
    });
    {

```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L439-L457

### Recommended Mitigation
Add required checks >0 ,min, max value checks before assigning values 



```solidity
if (!hooksTemplate.enabled){ revert;}

```
##

## [L-8] Potential Zero-Word Issue for Strings Exactly 32 Bytes Long

The mload function loads 32 bytes of data from memory. In the function you provided, the goal is to pack a string into two bytes32 words. The first word (word0) contains the length of the string and its first 31 bytes, while the second word (word1) is intended to hold the remaining part of the string (bytes 32 through 63).

The problem arises when the string is exactly 32 bytes long. Specifically, the check gt(mload(str), 0x1f) (i.e., mload(str) > 31) evaluates to true because the string length (32 bytes) is indeed greater than 31 bytes. 

### Explantion

- The memory layout of Solidity strings includes the first 32 bytes (a word) for the length of the string. The string data starts immediately after this length word.
- When the string is exactly 32 bytes long, the first word (word0) captures the first 31 bytes, along with the length byte.
- The second word (word1) should hold the 32nd byte (i.e., the last byte of the string) and should not be zeroed out.

- The code mload(add(str, 0x3f)) loads memory starting 63 bytes after the length, but for a string of exactly 32 bytes, there is no valid memory region that contains additional bytes.
- This means that mload(add(str, 0x3f)) might load undefined or invalid data, or it might return zeros depending on the underlying memory state.

- The current logic might end up zeroing out word1 even for strings exactly 32 bytes long. This is incorrect because word1 should store the 32nd byte, not be empty.

```solidity
FILE: 2024-08-wildcat/src/HooksFactory.sol

word1 := mul(mload(add(str, 0x3f)), gt(mload(str), 0x1f))

```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L389C7-L389C64

### Recommended mitgation
Adjust the comparison condition to account for the boundary case where the string length is exactly 32 bytes.

```solidity

word1 := mul(mload(add(str, 0x3f)), gt(mload(str), 0x20))

```

##

## [L-9] Panic on Length Mismatch in ``extractByteArrayLength``

The extractByteArrayLength function throws a panic if the encoding of the byte array is out of place. The condition for this panic is complex and could be prone to error if encoding is not handled properly, leading to the contract reverting unexpectedly.

```solidity
FILE: 2024-08-wildcat/src/types/TransientBytesArray.sol

 if eq(outOfPlaceEncoding, lt(length, 32)) {
          // Store the Panic error signature.
          mstore(0, Panic_ErrorSelector)
          // Store the arithmetic (0x11) panic code.
          mstore(Panic_ErrorCodePointer, Panic_InvalidStorageByteArray)
          // revert(abi.encodeWithSignature("Panic(uint256)", 0x22))
          revert(Error_SelectorPointer, Panic_ErrorLength)
        }

```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/TransientBytesArray.sol#L28C8-L35C10

##

## [L-10] Unchecked ``keccak256`` and ``for`` Loop in ``readToPointer``

In the readToPointer function, there’s a loop that reads transient data using keccak256 to calculate the storage slot of each subsequent data block. The loop does not have overflow protection, nor does it check for potential out-of-bound errors when reading large amounts of data.

```solidity
FILE: 2024-08-wildcat/src/types/TransientBytesArray.sol

 let i := 0
        for {

        } lt(i, length) {
          i := add(i, 0x20)
        } {
          tstore(dataTSlot, mload(add(memoryPointer, i)))
          dataTSlot := add(dataTSlot, 1)
        }

```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/TransientBytesArray.sol#L95C8-L105

### Recommended Mitigation
Add overflow checks and verify that the length and dataTSlot calculations are correctly bounded.


##

## [L-11] Potential for Indirect Reentrancy in updateSphereXEngineOnRegisteredContracts() function

Even with access control, it's still important to consider reentrancy protections when dealing with low-level call() operations.

Even if the ``call()`` target is controlled by a contract that implements access control, the reentrancy risk comes from how the external contract could behave. If the external contract (the ``target`` in ``_callWith``) is compromised or implements malicious logic, it can invoke other contracts or itself within the same transaction in ways that circumvent the access control. For instance, the external contract could call back into the original contract or exploit any other unprotected function. 

Example Scenario:
The external contract A receives the call via call() and immediately calls back into the contract making the original call (WildcatArchController in this case). If the WildcatArchController has any other publicly accessible functions that don't have reentrancy protections, this callback could reenter those functions and cause unintended behavior.

```solidity
FILE: 2024-08-wildcat/src/WildcatArchController.sol

function _updateSphereXEngineOnRegisteredContractsInSet(
    EnumerableSet.AddressSet storage set,
    address engineAddress,
    address[] memory contracts,
    bytes memory changeSphereXEngineCalldata,
    bytes memory addAllowedSenderOnChainCalldata,
    bytes4 notInSetErrorSelectorBytes
  ) internal {
    for (uint256 i = 0; i < contracts.length; i++) {
      address account = contracts[i];
      if (!set.contains(account)) {
        uint32 notInSetErrorSelector = uint32(notInSetErrorSelectorBytes);
        assembly {
          mstore(0, notInSetErrorSelector)
          revert(0x1c, 0x04)
        }
      }
      _callWith(account, changeSphereXEngineCalldata);
      if (engineAddress != address(0)) {
        assembly {
          mstore(add(addAllowedSenderOnChainCalldata, 0x24), account)
        }
        _callWith(engineAddress, addAllowedSenderOnChainCalldata);
        emit_NewAllowedSenderOnchain(account);
      }
    }

 function _callWith(address target, bytes memory data) internal {
    assembly {
      if iszero(call(gas(), target, 0, add(data, 0x20), mload(data), 0, 0)) {
        returndatacopy(0, 0, returndatasize())
        revert(0, returndatasize())
      }
    }
  }

```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatArchController.sol#L144-L150

### Recommended Mitigation
Apply the ``Checks-Effects-Interactions`` pattern to ensure that state changes are made before any external calls. Additionally, consider using the ``ReentrancyGuard`` from ``OpenZeppelin`` to prevent reentrant calls.

##

## [L-12] Lack of contract existence check when using assembly calls 

```
The low-level functions call, delegatecall and staticcall return true as their first
return value if the account called is non-existent, as part of the design of the
EVM. Account existence must be checked prior to calling if needed.

```
 A snippet of the Solidity documentation detailing unexpected behavior related to call


It is important to check whether the target address is a valid smart contract before performing a low-level call. Without this check, the call may fail if the target is an EOA or a non-existent contract, potentially causing unintended behavior.

```solidity
FILE: 2024-08-wildcat/src/WildcatArchController.sol

146:  if iszero(call(gas(), target, 0, add(data, 0x20), mload(data), 0, 0)) {

```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatArchController.sol#L146

## Recommended Mitigation
Implement a contract existence check before each assembly call 


##

## [L-13] Redundant address(0) Validation 

The engineAddress is fetched from a predefined storage slot (SPHEREX_ENGINE_STORAGE_SLOT), which is set to hold a constant and valid address. Since this slot is guaranteed to contain a valid non-zero address, engineAddress can never be address(0).

```solidity
FILE: 2024-08-wildcat/src/WildcatArchController.sol

84: if (engineAddress != address(0)) {

```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatArchController.sol#L84

### Recommended Mitigation
Remove redundant check 

##

## [L-14] Brittle and Inefficient Assembly-based Custom Error Handling

The code uses assembly to revert with a custom error (revert(0x1c, 0x04)). This is inefficient and brittle for modern Solidity code. The usage of hardcoded memory offsets for error data (e.g., 0x1c and 0x04) could be incorrect if the error handling format changes, leading to unexpected behavior.

```solidity
FILE:2024-08-wildcat/src/WildcatArchController.sol

assembly {
          mstore(0, notInSetErrorSelector)
          revert(0x1c, 0x04)
        }

```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatArchController.sol#L128-L132

## Recommended Mitigation
Consider using Solidity's native require or revert functionality instead of assembly to avoid the need to handle memory manually.

##

## [L-15] Unrestricted Gas Forwarding in External Calls Leading to Potential Out-of-Gas Errors

The code does not control gas usage properly in the call() function in _callWith(). Since it forwards all available gas (gas()), it can potentially lead to out-of-gas errors in the calling contract if the target contract consumes excessive gas.

```solidity
FILE: 2024-08-wildcat/src/WildcatArchController.sol

 function _callWith(address target, bytes memory data) internal {
    assembly {
      if iszero(call(gas(), target, 0, add(data, 0x20), mload(data), 0, 0)) {
        returndatacopy(0, 0, returndatasize())
        revert(0, returndatasize())
      }
    }

```

## Recommended Mitigation
Limit the gas passed to external calls to avoid running out of gas in the calling contract. You can modify the assembly block like so:

```solidity

 call(gasleft(), target, 0, add(data, 0x20), mload(data), 0, 0)

```

##

## [L-16] Potential for DoS by returning overly large arrays

The arr array is created dynamically based on the count of assets to return, which could result in an overly large array if the difference between start and end is large. This can lead to excessive memory usage and higher gas costs, potentially causing the function to fail or making it cost-prohibitive to use.

```solidity
FILE: 2024-08-wildcat/src/WildcatArchController.sol

function getRegisteredBorrowers(
    uint256 start,
    uint256 end
  ) external view returns (address[] memory arr) {
    uint256 len = _borrowers.length();
    end = MathUtils.min(end, len);
    uint256 count = end - start;
    arr = new address[](count);
    for (uint256 i = 0; i < count; i++) {
      arr[i] = _borrowers.at(start + i);
    }
  }

function getBlacklistedAssets(
    uint256 start,
    uint256 end
  ) external view returns (address[] memory arr) {
    uint256 len = _assetBlacklist.length();
    end = MathUtils.min(end, len);
    uint256 count = end - start;
    arr = new address[](count);
    for (uint256 i = 0; i < count; i++) {
      arr[i] = _assetBlacklist.at(start + i);
    }
  }


```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatArchController.sol#L222-L233

### Recommend Mitigation 
In addition to limiting the batch size (as in the previous point), ensure that the function is only called with reasonable ranges.

```solidity

require(count <= maxBatchSize, "Requested range is too large");

```

##

## [L-17] Lack of input validations when assigning values to immutable variables

The addresses passed to ``archController_``, ``_sanctionsSentinel``, and ``_marketInitCodeStorage`` are assigned directly without validation. If any of these addresses are incorrectly set (e.g., set to a zero address or malicious address), the contract might malfunction or become vulnerable to security exploits.

 ``_marketInitCodeHash`` is assigned directly without any validation or checks on the integrity or correctness of the input. If an incorrect hash is passed, the contract might fail to operate as expected, particularly in situations where market code initialization is critical.

If the constructor allows external or untrusted parties to pass in values, it opens up a risk of malicious input, which could compromise the security or functionality of the contract.

```solidity
FILE: 2024-08-wildcat/src/HooksFactory.sol

 constructor(
    address archController_,
    address _sanctionsSentinel,
    address _marketInitCodeStorage,
    uint256 _marketInitCodeHash
  ) {
    marketInitCodeStorage = _marketInitCodeStorage;
    marketInitCodeHash = _marketInitCodeHash;
    _archController = archController_;
    sanctionsSentinel = _sanctionsSentinel;
    __SphereXProtectedRegisteredBase_init(IWildcatArchController(archController_).sphereXEngine());
  }

```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L55-L66

## Recommended Mitigation
Add ``address(0)`` check and uint checks like > 0 and min,max value checks


##

## [L-18] Race Condition on Market Initialization in _deployMarket

In the _deployMarket function, the contract performs several operations before the actual market contract is deployed, including setting temporary market parameters in transient storage. However, the _setTmpMarketParameters(tmp) call occurs before the market is actually deployed.

#### Issue
 A race condition could occur if multiple market deployments are attempted simultaneously. Since the temporary market parameters are set in a shared storage variable (_tmpMarketParameters), two concurrent deployments could overwrite each other’s data, leading to incorrect market initialization or corrupted data.

```solidity
FILE: 2024-08-wildcat/src/HooksFactory.sol

 function _deployMarket(
    DeployMarketInputs memory parameters,
    bytes memory hooksData,
    address hooksTemplate,
    HooksTemplate memory templateDetails,
    bytes32 salt,
    address originationFeeAsset,
    uint256 originationFeeAmount
  ) internal returns (address market) {
    if (IWildcatArchController(_archController).isBlacklistedAsset(parameters.asset)) {
      revert AssetBlacklisted();
    }
    address hooksInstance = parameters.hooks.hooksAddress();

    if (!(address(bytes20(salt)) == msg.sender || bytes20(salt) == bytes20(0))) {
      revert SaltDoesNotContainSender();
    }

    if (
      originationFeeAsset != templateDetails.originationFeeAsset ||
      originationFeeAmount != templateDetails.originationFeeAmount
    ) {
      revert FeeMismatch();
    }

    if (originationFeeAsset != address(0)) {
      originationFeeAsset.safeTransferFrom(
        msg.sender,
        templateDetails.feeRecipient,
        originationFeeAmount
      );
    }

    market = LibStoredInitCode.calculateCreate2Address(ownCreate2Prefix, salt, marketInitCodeHash);

    parameters.hooks = IHooks(hooksInstance).onCreateMarket(
      msg.sender,
      market,
      parameters,
      hooksData
    );
    uint8 decimals = parameters.asset.decimals();

    string memory name = string.concat(parameters.namePrefix, parameters.asset.name());
    string memory symbol = string.concat(parameters.symbolPrefix, parameters.asset.symbol());

    TmpMarketParameterStorage memory tmp = TmpMarketParameterStorage({
      borrower: msg.sender,
      asset: parameters.asset,
      packedNameWord0: bytes32(0),
      packedNameWord1: bytes32(0),
      packedSymbolWord0: bytes32(0),
      packedSymbolWord1: bytes32(0),
      decimals: decimals,
      feeRecipient: templateDetails.feeRecipient,
      protocolFeeBips: templateDetails.protocolFeeBips,
      maxTotalSupply: parameters.maxTotalSupply,
      annualInterestBips: parameters.annualInterestBips,
      delinquencyFeeBips: parameters.delinquencyFeeBips,
      withdrawalBatchDuration: parameters.withdrawalBatchDuration,
      reserveRatioBips: parameters.reserveRatioBips,
      delinquencyGracePeriod: parameters.delinquencyGracePeriod,
      hooks: parameters.hooks
    });
    {
      (tmp.packedNameWord0, tmp.packedNameWord1) = _packString(name);
      (tmp.packedSymbolWord0, tmp.packedSymbolWord1) = _packString(symbol);
    }

    _setTmpMarketParameters(tmp);

    if (market.code.length != 0) {
      revert MarketAlreadyExists();
    }
    LibStoredInitCode.create2WithStoredInitCode(marketInitCodeStorage, salt);

    IWildcatArchController(_archController).registerMarket(market);

    _tmpMarketParameters.setEmpty();

    _marketsByHooksTemplate[hooksTemplate].push(market);

    emit MarketDeployed(
      hooksTemplate,
      market,
      name,
      symbol,
      tmp.asset,
      tmp.maxTotalSupply,
      tmp.annualInterestBips,
      tmp.delinquencyFeeBips,
      tmp.withdrawalBatchDuration,
      tmp.reserveRatioBips,
      tmp.delinquencyGracePeriod,
      tmp.hooks
    );
  }

```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L393C2-L489C4

### Recommended Mitgation
Use a unique storage slot or a mapping keyed by the deployer or the market address to store temporary parameters.


##

## [L-19] Logical Mistake in rescueTokens() function Conditional Check

using .or for the conditional check, which is incorrect for Solidity.

```solidity
FILE:2024-08-wildcat/src/market/WildcatMarket.sol

38: if ((token == asset).or(token == address(this))) {

``` 
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarket.sol#L38

### Recommended Mitigation
 Use ``||`` for ``logical OR``.

##

## [L-20] ``WithdrawalBatchExpired`` Event Emitted even if not Processing ``WithdrawalBatchPayment()``

The issue arises when there is insufficient liquidity (availableLiquidity == 0) during withdrawal batch processing. This can lead to unprocessed or unpaid withdrawals, as the system may emit the WithdrawalBatchExpired event without properly handling the pending withdrawals, potentially leaving users unable to access their funds.

```solidity
FILE: 2024-08-wildcat/src/market
/WildcatMarketBase.sol

   emit_WithdrawalBatchExpired(
      expiry,
      batch.scaledTotalAmount,
      batch.scaledAmountBurned,
      batch.normalizedAmountPaid
    );

```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketBase.sol#L643-L648

### Recommended Mitigation
Only emit emit_WithdrawalBatchExpired event when availableLiquidity > 0 . Move emit to inside the if block 

##

## [L-21] Lack of reentrancy modifiers in critical functions depositUpTo(),deposit() 

The functions depositUpTo() and deposit() handle critical operations such as transferring assets and minting tokens. Without proper reentrancy protection, an attacker could exploit these functions by calling them recursively before the state is fully updated.

```solidity
FILE:2024-08-wildcat/src/market/WildcatMarket.sol

 function depositUpTo(
    uint256 amount
  ) external virtual sphereXGuardExternal returns (uint256 /* actualAmount */) {
    return _depositUpTo(amount);
  }

function deposit(uint256 amount) external virtual sphereXGuardExternal {
    uint256 actualAmount = _depositUpTo(amount);
    if (amount != actualAmount) revert_MaxSupplyExceeded();
  }

```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarket.sol#L117-L120

### Recommended Mitigation
Add nonReentrant modifiers that are dealing transfers 

##

## [L-22] collectFees() function not followed the CEI Pattern

The function makes an external token transfer to feeRecipient before updating the internal state with _writeState(state). This violates the CEI pattern because it opens up a potential vulnerability where an attacker could exploit reentrancy through the external safeTransfer call, which may call back into the contract before the internal state is updated.

```solidity
FILE: 2024-08-wildcat/src/market/WildcatMarket.sol

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
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarket.sol#L125-L136

### Recommended Mitigation
Implement the Check-Effects-Interaction patterns

##

## [L-23] No Fallback Function for for WildcatSanctionsEscrow contract

The contract is designed for ERC-20 tokens (asset is treated as an ERC-20 token), but there’s no mechanism to handle the escrow of native Ether. If someone sends Ether to this contract, it will be locked permanently.

Accidental or malicious Ether transfers would cause the contract to hold Ether that cannot be recovered.

```soldiity
FILE:2024-08-wildcat/src/WildcatSanctionsEscrow.sol

contract WildcatSanctionsEscrow is IWildcatSanctionsEscrow {

```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatSanctionsEscrow.sol#L9

### Recommended Mitigation
Add a fallback function to prevent receiving Ether

```solidity

receive() external payable {
    revert("Ether not accepted");
}

```

##

## [L-24] Risk of Failed Escrow Release Due to Blocklisted Recipient Address

Even the account unsanctioned its not possible to transfer escrow funds to account.

releaseEscrow() function, there is a critical issue if the asset is a token that enforces blocklists, such as USDC or USDT. These tokens have an admin-controlled mechanism that blocks transfers to or from blocklisted addresses. If the account address is blocklisted, the transfer will fail.This can prevent escrow funds from being released as expected, leading to unexpected behavior or a stuck escrow situation.

```solidity
FILE:2024-08-wildcat/src/WildcatSanctionsEscrow.sol

function releaseEscrow() public override {
    if (!canReleaseEscrow()) revert CanNotReleaseEscrow();

    uint256 amount = balance();
    address _account = account;
    address _asset = asset;

    asset.safeTransfer(_account, amount);

    emit EscrowReleased(_account, _asset, amount);
  }

```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatSanctionsEscrow.sol#L34-L44

### Recommended Mitigation

Before attempting to transfer funds, you need to check if the account is blocklisted for the specific token (e.g., USDC, USDT). However, not all token contracts provide an accessible method for checking this, so it may require off-chain checks or relying on third-party services that monitor blocklists.


##

## [L-25]  ``getAvailableWithdrawalAmount()`` function potential to Dust Accumulation from Rounding Errors

In the ``getAvailableWithdrawalAmount`` function, there is a possibility of dust accumulation due to rounding errors during withdrawal calculations. The comment itself acknowledges this, but there is no mechanism to deal with the dust effectively:

Over time, small amounts of dust could accumulate and may become unrecoverable, leading to accounting discrepancies.

```

// Rounding errors will lead to some dust accumulating in the batch, but the cost of
// executing a withdrawal will be lower for users.
  uint256 previousTotalWithdrawn = status.normalizedAmountWithdrawn;
    uint256 newTotalWithdrawn = uint256(batch.normalizedAmountPaid).mulDiv(
      status.scaledAmount,
      batch.scaledTotalAmount
    );

```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketWithdrawals.sol#L66-L69

### Recommended Mitigation

```solidity

uint256 private dustBalance;

```
When calculating the withdrawal amount, we can store any dust (remainder) in the ``dustBalance`` and handle it as needed.You can introduce a function that allows the contract owner or an automated process to sweep the dust and handle it appropriately.

##

## [L-26] Inefficient Processing of Withdrawal Batches Due to Missing Available Liquidity Check

the system processes pending withdrawal batches and sets aside available liquidity to fulfill them. However, the code lacks a check to ensure that availableLiquidity is greater than 0 before attempting to allocate liquidity to a pending withdrawal batch. This can lead to unnecessary calculations and operations when there is no available liquidity, ultimately having no effect if availableLiquidity == 0.

```solidity
FILE: 2024-08-wildcat/src/market/WildcatMarket.sol

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


 for (uint256 i; i < numBatches; i++) {
      // Process the next unpaid batch using available liquidity
      uint256 normalizedAmountPaid = _processUnpaidWithdrawalBatch(state, availableLiquidity);
      // Reduce liquidity available to next batch
      availableLiquidity -= normalizedAmountPaid;
    }

```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarket.sol#L274-L279

### Recommended Mitigation
Add a Check for ``availableLiquidity > 0``

##

## [L-27] Inability to Remove Deprecated Hook Templates from ``_hooksTemplates`` Array

Adds a new hooksTemplate to the _hooksTemplates array, but the array is defined as address[] internal, meaning it only allows adding new templates (pushing), not removing (popping). This could cause several issues if a hook becomes deprecated, malicious, or no longer needed.

Specifically, you won't be able to pop or remove deprecated hooks from the array, leading to the following risks:

 - Accumulation of Deprecated Hooks
 - Difficulty in Managing Hook Lifecycle

```solidity
FILE: 2024-08-wildcat/src/HooksFactory.sol

_hooksTemplates.push(hooksTemplate);

```
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L142

##

## [L-28] Sanctioned ``Borrowers`` Allowed to Deploy New Markets, Violating ``Protocol Compliance Guidelines``

### Documentation

```
After first contact, what Wildcat is looking for is proof that the borrower is a legal entity in a jurisdiction that is a) not sanctioned, and b) we reasonably believe won't expose the protocol to legal or regulatory risk.

```

In the current implementation, there appears to be no enforcement of sanctions checks when borrowers attempt to deploy new markets. According to the documentation, borrowers must be legal entities in non-sanctioned jurisdictions to avoid exposing the protocol to legal or regulatory risks. However, if a sanctioned borrower is able to deploy a new market, this violates the protocol’s compliance and documentation guidelines.

```solidity
FILE: 2024-08-wildcat/src/HooksFactory.sol

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
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L491-L516

### Recommended Mitigation
Add this check to deployMarket() and deployMarketAndHooks() functions 

```solidity
 if (_isFlaggedByChainalysis(msg.sender)) {
        revert_BorrowWhileSanctioned();
    }
```




 






| Issue Number | Title |
|--------------|-----------------------------------------|
| [L-01](#l-01---incorrect-packing-of-strings-in-_packstring) | Incorrect Packing of Strings in `_packString` |
| [L-02](#l-02---incorrect-salt-validation-in-_deploymarket) | Incorrect Salt Validation in `_deployMarket` |
| [L-03](#l-03---potential-initialization-code-length-miscalculation-in-_deployhooksinstance) | Potential Initialization Code Length Miscalculation in `_deployHooksInstance` |
| [L-04](#l-04---missing-access-control-on-pushprotocolfeebipsupdates-function) | Missing Access Control on `pushProtocolFeeBipsUpdates` Function |
| [L-05](#l-05---potential-storage-collision-with-transient-storage) | Potential Storage Collision with Transient Storage |
| [L-06](#l-06---potential-misplacement-of-account-in-addallowedsenderonchaincalldata) | Potential Misplacement of Account in `addAllowedSenderOnChainCalldata` |
| [L-08](#l-08---improper-handling-of-expired-credentials-without-cleanup) | Improper Handling of Expired Credentials Without Cleanup |
| [L-09](#l-09---misaligned-data-handling-in-assembly-causing-incorrect-calldata-formatting) | Misaligned Data Handling in Assembly Causing Incorrect Calldata Formatting |
| [L-10](#l-10---potential-data-overflow-in-fixedtermendtime) | Potential Data Overflow in `fixedTermEndTime` |
| [L-11](#l-11---memory-corruption-in-_issanctioned-function) | Memory Corruption in `_isSanctioned` Function |
| [L-12](#l-12---incorrect-calldata-construction-in-_issanctioned-function) | Incorrect Calldata Construction in `_isSanctioned` Function |
| [L-13](#l-13---incorrect-storage-packing-in-_writestate-function) | Incorrect Storage Packing in `_writeState` Function |
| [L-14](#l-14---lack-of-sanctions-check-in-_createescrowforunderlyingasset-function) | Lack of Sanctions Check in `_createEscrowForUnderlyingAsset` Function |
| [L-15](#l-15---missing-checks-in-_processunpaidwithdrawalbatch-function) | Missing Checks in `_processUnpaidWithdrawalBatch` Function |
| [L-16](#l-16---missing-input-validation-in-queuewithdrawal-and-queuefullwithdrawal-functions) | Missing Input Validation in `queueWithdrawal` and `queueFullWithdrawal` Functions |
| [L-17](#l-17---incorrect-bitmasking-and-shifting-in-encodehooksdeploymentconfig-function) | Incorrect Bitmasking and Shifting in `encodeHooksDeploymentConfig` Function |
| [L-18](#l-18---overwriting-free-memory-pointer-0x40-in-hook-functions) | Overwriting Free Memory Pointer (`0x40`) in Hook Functions |
| [L-19](#l-19---incorrect-function-selector-offset-in-hook-calls) | Incorrect Function Selector Offset in Hook Calls |
| [L-20](#l-20---misaligned-calldata-structure-in-hook-functions) | Misaligned Calldata Structure in Hook Functions |
| [L-21](#l-21---incorrect-calculation-of-size-variable-in-hook-calls) | Incorrect Calculation of `size` Variable in Hook Calls |
| [L-22](#l-22---potential-misalignment-in-mergeflags-function) | Potential Misalignment in `mergeFlags` Function |
| [L-23](#l-23---potential-loss-of-extradata-in-hook-calls) | Potential Loss of `extraData` in Hook Calls |
| [L-24](#l-24---missing-shift-operation-when-writing-short-byte-arrays) | Missing Shift Operation When Writing Short Byte Arrays |


---



## [L-01] - Incorrect Packing of Strings in `_packString`

### Description:
The [`_packString` function](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L376-L391) attempts to pack a string into two `bytes32` words. However, it incorrectly calculates memory offsets, leading to potential misalignment and incorrect packing of string data. Specifically, the function adds `0x1f` and `0x3f` to the string pointer instead of the correct offsets, which can result in incomplete or malformed string representations.

**Exact Code Snippet:**
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L376-L391
```solidity
function _packString(string memory str) internal pure returns (bytes32 word0, bytes32 word1) {
    assembly {
      let length := mload(str)
      // Equivalent to:
      // if (str.length > 63) revert NameOrSymbolTooLong();
      if gt(length, 0x3f) {
        mstore(0, 0x19a65cb6)
        revert(0x1c, 0x04)
      }
      // Load the length and first 31 bytes of the string into the first word
      // by reading from 31 bytes after the length pointer.
      word0 := mload(add(str, 0x1f))
      // If the string is less than 32 bytes, the second word will be zeroed out.
      word1 := mul(mload(add(str, 0x3f)), gt(mload(str), 0x1f))
    }
}
```

### Impact :
Incorrect string packing can lead to malformed names or symbols, causing token metadata issues or unexpected behavior in markets.

### Recommendation:
Correct the memory offsets used in the assembly to accurately pack the string into two `bytes32` words. Ensure that the first word starts at `str + 0x20` (to skip the length) and the second word correctly captures any remaining bytes. 

---

## [L-02] - Incorrect Salt Validation in `_deployMarket`

### Description:
The [`_deployMarket` function](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L393-L489) enforces that the first 20 bytes of the `salt` must match the `msg.sender` or be zero. This restrictive check limits the flexibility of salt usage and tightly couples the salt to the caller's address, which might not be necessary and could lead to unintended restrictions.

**Exact Code Snippet:**
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L393-L489
```solidity
if (!(address(bytes20(salt)) == msg.sender || bytes20(salt) == bytes20(0))) {
    revert SaltDoesNotContainSender();
}
```

### Impact :
Restricts salt usage, potentially preventing valid market deployments or enabling predictable addresses, which could be exploited.

### Recommendation:
Remove or revise the restrictive salt validation to allow more flexibility in salt generation. If associating the salt with the sender is required for business logic, ensure it's done securely without unnecessary constraints.

---

## [L-03] - Potential Initialization Code Length Miscalculation in `_deployHooksInstance`

### Description:
In the [`_deployHooksInstance` function](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L289-L328), the `initCodeSize` is calculated by subtracting 1 from the `extcodesize` of the `hooksTemplate`. This subtraction likely results in an incomplete copy of the hooks template's bytecode, leading to deployment failures or malfunctioning hooks instances.

**Exact Code Snippet:**
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L303
```solidity
let initCodeSize := sub(extcodesize(hooksTemplate), 1)
```

### Impact :
Could cause the deployed hooks instance to have incomplete or incorrect bytecode, leading to contract deployment failure or malfunctioning hooks.

### Recommendation:
Remove the `-1` from the `extcodesize` calculation to ensure the full bytecode of the hooks template is copied accurately. The correct line should be:
```solidity
let initCodeSize := extcodesize(hooksTemplate)
```

---

## [L-04] - Missing Access Control on `pushProtocolFeeBipsUpdates` Function

### Description:
The [`pushProtocolFeeBipsUpdates` function](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L553-L587) lacks access control modifiers, allowing any external user to call it. This can lead to unauthorized modifications of protocol fee configurations across multiple markets associated with a hooks template.

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L553-L587
**Exact Code Snippet:**
```solidity
function pushProtocolFeeBipsUpdates(
    address hooksTemplate,
    uint marketStartIndex,
    uint marketEndIndex
) public override nonReentrant {
    // Function logic
}
```

### Impact :
Unauthorized users can update protocol fee configurations across multiple markets, potentially altering fees to unintended values and disrupting the protocol's revenue model.

### Recommendation:
Restrict access to the `pushProtocolFeeBipsUpdates` function by adding appropriate access control, such as the `onlyArchControllerOwner` modifier, to ensure that only authorized entities can perform fee updates.


---

## [L-05] - Potential Storage Collision with Transient Storage

### Description:
The `HooksFactory` contract uses a transient storage variable `_tmpMarketParameters` with a hashed key derived from `keccak256('Transient:TmpMarketParametersStorage') - 1`. If another contract or library uses the same key or a similar hashing scheme, it could lead to storage collisions, resulting in corrupted data or unexpected behavior.

**Exact Code Snippet:**
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L37-L38
```solidity
TransientBytesArray internal constant _tmpMarketParameters =
    TransientBytesArray.wrap(uint256(keccak256('Transient:TmpMarketParametersStorage')) - 1);
```

### Impact :
Data stored in transient storage could be overwritten or read incorrectly, leading to corrupted market parameters or security vulnerabilities.

### Recommendation:
Ensure that the transient storage keys are unique and unlikely to collide with other contracts or libraries. Consider using more specific or project-unique identifiers for hashing

---




## [L-06} - Potential Misplacement of Account in `addAllowedSenderOnChainCalldata`

### Description:
Within the [`_updateSphereXEngineOnRegisteredContractsInSet` function](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatArchController.sol#L116-L142), when `engineAddress` is not `address(0)`, the contract modifies the `addAllowedSenderOnChainCalldata` by storing the `account` at an offset of `0x24`. This approach assumes a specific calldata structure, which may not align with the actual function signature of `ISphereXEngine.addAllowedSenderOnChain`. Incorrectly positioning the `account` parameter can lead to malformed calldata, causing the external call to fail or behave unexpectedly.

**Exact Code Snippet:**
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatArchController.sol#L135-L137
```solidity
assembly {
  mstore(add(addAllowedSenderOnChainCalldata, 0x24), account)
}
```

### Impact :
Incorrect calldata formatting may cause the `addAllowedSenderOnChain` function to receive invalid parameters, leading to failed transactions or unintended behavior.

### Recommendation:
Ensure that the `account` parameter is correctly positioned within the calldata according to the function's ABI. Typically, function selectors occupy the first 4 bytes, followed by encoded parameters. To accurately place the `account`, calculate the correct offset based on the function's parameter encoding. 




---

## [L-08] - Improper Handling of Expired Credentials Without Cleanup

### Description:
When a lender's credential expires and cannot be refreshed, the contract unsets the credential in the `LenderStatus` struct but does not perform any additional cleanup or state updates beyond reverting access. Over time, this can lead to stale data within the [`_lenderStatus` mapping](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/AccessControlHooks.sol#L89) , increasing storage costs and potentially impacting the efficiency of credential validations.

**Exact Code Snippet:**
```solidity
if (credentialTimestamp == 0 || credentialTimestamp > block.timestamp) {
    return false;
}
// If credential is still valid, update credential
if (provider.calculateExpiry(credentialTimestamp) >= block.timestamp) {
    status.setCredential(provider, credentialTimestamp);
    return true;
}
// Else, credential is invalid
status.unsetCredential();
```

### Impact :
Accumulation of stale `LenderStatus` entries increases storage costs and may degrade the performance of credential validations.

### Recommendation:
Implement a mechanism to clean up or efficiently manage stale credentials. 

---


## [L-09] - Misaligned Data Handling in Assembly Causing Incorrect Calldata Formatting

### Description:
The contract uses low-level assembly to construct calldata for external function calls, such as in [`_tryGetCredential`](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/FixedTermLoanHooks.sol#L507-L539) and [`_tryValidateCredential`](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/FixedTermLoanHooks.sol#L562-L622). However, it incorrectly calculates memory offsets (e.g., starting at `0x1c` instead of `0x00`), leading to improperly formatted calldata. This misalignment can cause external calls to fail or behave unexpectedly, disrupting credential validation processes.

**Exact Code Snippet:**
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/FixedTermLoanHooks.sol#L517-L526
```solidity
assembly {
    mstore(0x00, getCredentialSelector)
    mstore(0x20, accountAddress)
    // Call the provider and check if the return data is valid
    if and(gt(returndatasize(), 0x1f), staticcall(gas(), providerAddress, 0x1c, 0x24, 0, 0x20)) {
        credentialTimestamp := and(mload(0), 0xffffffff)
    }
}
```

### Impact
Incorrect calldata formatting leads to failed external calls, preventing proper credential validation and potentially blocking legitimate lender actions.

**Recommendation:**
Avoid using low-level assembly for calldata construction unless absolutely necessary. Utilize Solidity's high-level `abi.encodeWithSelector` and `abi.decode` functions to ensure correct and safe calldata formatting. If assembly is required for optimization, meticulously calculate memory offsets and ensure alignment with Solidity's ABI specifications.


---

## [L-10] - Potential Data Overflow in `fixedTermEndTime`

### Description:
The `fixedTermEndTime` is stored as a `uint32`, representing a Unix timestamp. While `uint32` can accommodate timestamps up to approximately the year 2106, it's prudent to consider future-proofing by using larger data types to avoid potential overflows or limitations as blockchain timestamps evolve.

**Exact Code Snippet:**
```solidity
uint32 public constant MaximumLoanTerm = 365 days;
...
uint32 fixedTermEndTime;
assembly {
    fixedTermEndTime := calldataload(hooksData.offset)
}
```

### Impact:
Limits the maximum representable timestamp to the year 2106, which may be insufficient for long-term deployments or future protocol extensions.

### Recommendation:
Use a larger data type, such as `uint64` or `uint256`, for `fixedTermEndTime` to accommodate a broader range of timestamps and enhance future compatibility.


---


## [L-11] - Memory Corruption in `_isSanctioned` Function

### Description:
The [`_isSanctioned` function ](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketBase.sol#L254-L273) incorrectly stores the `account` address at memory position `0x40`, which is reserved for the free memory pointer in Solidity. Overwriting this can lead to memory corruption and unpredictable behavior in subsequent memory operations.

**Exact Code Snippet:**
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketBase.sol#L254-L273
```solidity
function _isSanctioned(address account) internal view returns (bool result) {
  address _borrower = borrower;
  address _sentinel = address(sentinel);
  assembly {
    let freeMemoryPointer := mload(0x40)
    mstore(0, 0x06e74444)
    mstore(0x20, _borrower)
    mstore(0x40, account) // Overwrites the free memory pointer
    if iszero(
      and(eq(returndatasize(), 0x20), staticcall(gas(), _sentinel, 0x1c, 0x44, 0, 0x20))
    ) {
      returndatacopy(0, 0, returndatasize())
      revert(0, returndatasize())
    }
    result := mload(0)
    mstore(0x40, freeMemoryPointer)
  }
}
```

### Impact:
Overwriting the free memory pointer can corrupt the contract's memory state, leading to unexpected behaviors or vulnerabilities.

### Recommendation:
Use a different memory location for storing temporary data to avoid corrupting the free memory pointer. For example:
```solidity
mstore(0x80, account)
```

---

## [L-12] - Incorrect Calldata Construction in `_isSanctioned` Function

**Description:**
The [`_isSanctioned` function](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketBase.sol#L254-L273) constructs calldata for the `isSanctioned` function by setting the calldata starting at `0x1c` with a size of `0x44`. This offset is incorrect and does not align with Solidity's ABI encoding, leading to improper parameter passing.

**Exact Code Snippet:**
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketBase.sol#L265
```solidity
staticcall(gas(), _sentinel, 0x1c, 0x44, 0, 0x20)
```

### Impact:
Incorrect calldata offsets can cause the `isSanctioned` call to pass wrong parameters, resulting in false sanction checks and potential unauthorized access.

### Recommendation:
Construct calldata starting at memory position `0x00` with correct parameter encoding. For example:
```solidity
mstore(0x00, 0x06e74444) // function selector
mstore(0x04, _borrower)
mstore(0x24, account)
staticcall(gas(), _sentinel, 0x00, 0x44, 0x00, 0x20)
```
Ensure the function selector and parameter encoding align with the `isSanctioned` function's signature.

---


## [L-13] - Incorrect Storage Packing in `_writeState` Function

### Description:
The [`_writeState` function](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketBase.sol#L540-L618) manually packs multiple state variables into storage slots using bitwise operations. Any mismatch between the expected storage layout and the actual packing can lead to corrupted state variables.

**Exact Code Snippet:**
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketBase.sol#L558-L564
```solidity
assembly {
  // Slot 0 Storage Layout:
  // [15:31] | state.maxTotalSupply
  // [31:32] | state.isClosed
  let slot0 := or(isClosed, shl(0x08, maxTotalSupply))
  sstore(_state.slot, slot0)
}
```

### Impact:
Incorrect storage packing can corrupt the contract's state, leading to unintended behaviors, loss of funds, or security vulnerabilities.


---

## [L-14] - Lack of Sanctions Check in `_createEscrowForUnderlyingAsset` Function

### Description:
The [`_createEscrowForUnderlyingAsset` function](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketBase.sol#L769-L792) does not perform a sanctions check on the `accountAddress` before creating an escrow. This oversight allows sanctioned accounts to create escrows and interact with the market, potentially bypassing intended restrictions.

**Exact Code Snippet:**
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketBase.sol#L769-L792
```solidity
function _createEscrowForUnderlyingAsset(
  address accountAddress
) internal returns (address escrow) {
  // ... assembly code to create escrow without sanctions check
}
```

### Impact:
Sanctioned accounts can create escrows and participate in the market, undermining the contract's compliance and security measures.

### Recommendation:
Integrate a sanctions check within the `_createEscrowForUnderlyingAsset` function to ensure that only non-sanctioned accounts can create escrows. For example:
```solidity
function _createEscrowForUnderlyingAsset(
  address accountAddress
) internal returns (address escrow) {
    if (_isSanctioned(accountAddress)) {
        revert AccountIsSanctioned();
    }
    // Proceed with escrow creation
}
```
Ensure `AccountIsSanctioned` is a defined error in the `IMarketEventsAndErrors` interface.



---

## [L-14] - Overwriting Memory in `_createEscrowForUnderlyingAsset` Function

### Description:
The [`_createEscrowForUnderlyingAsset` function](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketBase.sol#L769-L792) writes to memory locations `0x40` and `0x60` without restoring them after use. This can corrupt the free memory pointer and other memory data, leading to unpredictable contract behavior.

**Exact Code Snippet:**
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketBase.sol#L789-L790
```solidity
assembly {
  mstore(0x40, freeMemoryPointer)
  mstore(0x60, 0)
}
```

### Impact:
Overwriting and not restoring memory pointers can lead to memory corruption, causing functions to behave unpredictably or revert unexpectedly.

### Recommendation:
Ensure that any modifications to memory pointers like `0x40` are properly restored after use.

---


## [L-15] - Missing Checks in `_processUnpaidWithdrawalBatch` Function

### Description:
In the [`_processUnpaidWithdrawalBatch` function,](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketWithdrawals.sol#L319-L345) after processing a batch, it shifts the `unpaidBatches` array without verifying that the batch was indeed fully paid. This could lead to removing a batch that is still partially unpaid.

**Exact Code Snippet:**
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketWithdrawals.sol#L341-L343
```solidity
if (batch.scaledTotalAmount == batch.scaledAmountBurned) {
  _withdrawalData.unpaidBatches.shift();
  emit_WithdrawalBatchClosed(expiry);
}
```

### Impact:
Removing a batch that is not fully paid can result in incomplete processing of withdrawals, potentially leaving users unable to withdraw their funds.

### Recommendation:
Ensure that only fully paid batches are removed from `unpaidBatches`. Verify that `batch.scaledAmountBurned >= batch.scaledTotalAmount` before shifting. 

---


## [L-16] - Missing Input Validation in `queueWithdrawal` and `queueFullWithdrawal` Functions

### Description:
The [`queueWithdrawal` ](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketWithdrawals.sol#L80-L130) and [`queueFullWithdrawal` functions](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketWithdrawals.sol#L135-L148) convert user-specified amounts to `scaledAmount` without validating that the resulting scaled amounts are within acceptable ranges. This could allow users to queue withdrawals with excessively large or invalid amounts.

**Exact Code Snippet:**
```solidity
uint104 scaledAmount = state.scaleAmount(amount).toUint104();
if (scaledAmount == 0) revert_NullBurnAmount();
```

### Impact:
Allowing excessively large or invalid scaled amounts could lead to incorrect state updates or unintended fund locking.

### Recommendation:
Implement comprehensive validation of the `scaledAmount` to ensure it is within expected ranges. For example:
```solidity
require(scaledAmount > 0 && scaledAmount <= account.scaledBalance, "Invalid withdrawal amount");
```
---



## [L-17] - Incorrect Bitmasking and Shifting in `encodeHooksDeploymentConfig` Function

### Description:
The [`encodeHooksDeploymentConfig` function](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/HooksConfig.sol#L18-L27) attempts to encode `optionalFlags` and `requiredFlags` into a `HooksDeploymentConfig` type using inline assembly. However, the bitmasking and shifting operations are incorrectly applied, potentially leading to improper encoding of flags.

**Exact Code Snippet:**
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/HooksConfig.sol#L18-L27
```solidity
function encodeHooksDeploymentConfig(
  HooksConfig optionalFlags,
  HooksConfig requiredFlags
) pure returns (HooksDeploymentConfig flags) {
  assembly {
    let cleanedOptionalFlags := and(0xffff, shr(0x50, optionalFlags))
    let cleanedRequiredFlags := and(0xffff0000, shr(0x40, requiredFlags))
    flags := or(cleanedOptionalFlags, cleanedRequiredFlags)
  }
}
```

### Impact:
Incorrect encoding of deployment flags may result in misconfiguration of hooks, enabling or disabling unintended functionalities.

### Recommendation:
Verify and correct the bitmasking and shifting operations to ensure that `optionalFlags` and `requiredFlags` are accurately positioned within the `HooksDeploymentConfig`. For example:
```solidity
flags := or(cleanedOptionalFlags, shr(0x40, cleanedRequiredFlags))
```
Ensure that the masks (`0xffff` and `0xffff0000`) align with the intended bit positions.


---

## [L-18] - Overwriting Free Memory Pointer (`0x40`) in Hook Functions

### Description:
In multiple hook functions, memory is allocated starting at `0x40` without preserving the original free memory pointer. Overwriting the free memory pointer can lead to memory corruption and unpredictable behavior in subsequent operations.

**Exact Code Snippet:**
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/HooksConfig.sol#L274-L275
```solidity
let cdPointer := mload(0x40)
let headPointer := add(cdPointer, 0x20)
// ...
// No restoration of free memory pointer
```

### Impact 
Overwriting the free memory pointer can corrupt Solidity's memory management, leading to unpredictable behavior and potential vulnerabilities.

### Recommendation:
Preserve the original free memory pointer before modifying it and restore it after operations. For example:
```solidity
let originalFreeMem := mload(0x40)
// ... perform memory operations
mstore(0x40, originalFreeMem)
```
Alternatively, use higher memory locations for temporary data to avoid interfering with the free memory pointer.

---

## [L-19] - Incorrect Function Selector Offset in Hook Calls

### Description:
In the assembly `call` instructions within hook functions, the calldata pointer is offset by `0x1c` bytes (`add(cdPointer, 0x1c)`). This offset is incorrect and misaligns the function selector and parameters, causing hook calls to fail.

**Exact Code Snippet:**
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/HooksConfig.sol#L300-L303
```solidity
if iszero(call(gas(), target, 0, add(cdPointer, 0x1c), size, 0, 0)) {
  returndatacopy(0, 0, returndatasize())
  revert(0, returndatasize())
}
```

### Impact:
Incorrect calldata offset will cause hook calls to fail or pass malformed data, leading to unexpected behaviors or reverted transactions.

### Recommendation:
Use the correct calldata pointer without unnecessary offsets. Replace `add(cdPointer, 0x1c)` with `cdPointer` to ensure the function selector and parameters are correctly aligned. For example:
```solidity
if iszero(call(gas(), target, 0, cdPointer, size, 0, 0)) {
  returndatacopy(0, 0, returndatasize())
  revert(0, returndatasize())
}
```

---

## [L-20] - Misaligned Calldata Structure in Hook Functions**

### Description:
The hook functions construct calldata by writing selectors and parameters at specific memory offsets. However, the offsets used for writing parameters and copying `extraData` may not align correctly with the expected structure of the hook functions, leading to malformed calldata.

**Exact Code Snippet:**
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/HooksConfig.sol#L277-L283
```solidity
// Example from onDeposit
mstore(cdPointer, onDepositSelector)
mstore(headPointer, lender)
mstore(add(headPointer, DepositHook_ScaledAmount_Offset), scaledAmount)
mcopy(add(headPointer, DepositHook_State_Offset), state, MarketStateSize)
// ...
calldatacopy(
  add(headPointer, DepositHook_ExtraData_TailOffset),
  DepositCalldataSize,
  extraCalldataBytes
)
```

### Impact:
Misaligned calldata can cause hook functions to receive incorrect parameters, leading to failures or unintended behaviors.


---

## [L-21] - Incorrect Calculation of `size` Variable in Hook Calls

### Description:
In several hook functions, the `size` variable is calculated using an incorrect base size constant (e.g., using `RepayHook_Base_Size` in the `onBorrow` function). This mismatch can lead to incorrect calldata sizes being sent to hook contracts.

**Exact Code Snippet:**
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/HooksConfig.sol#L583
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/HooksConfig.sol#L533
```solidity
let size := add(RepayHook_Base_Size, extraCalldataBytes) // In onBorrow
```

### Impact:
Using an incorrect base size for calldata calculation will result in malformed hook calls, causing them to fail or process incorrect data.

### Recommendation:
Ensure that each hook function uses the correct base size constant when calculating `size`. For example, in the `onBorrow` function, use `BorrowHook_Base_Size` instead of `RepayHook_Base_Size`:
```solidity
let size := add(BorrowHook_Base_Size, extraCalldataBytes)
```



---

## [L-22] - Potential Misalignment in `mergeFlags` Function

### Description:
The [`mergeFlags` function](https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/HooksConfig.sol#L127-L144) attempts to merge `HooksConfig` with `HooksDeploymentConfig`. However, the bit shifting and masking operations may not correctly align the optional and required flags, leading to improper flag configurations.

**Exact Code Snippet:**
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/HooksConfig.sol#L108-L109
```solidity
mergedFlags := shr(0xa0, and(flagsA, flagsB))
merged := or(addressA, mergedFlags)
```

### Impact:
Misaligned flag merging can result in unintended hooks being enabled or disabled, compromising the contract's functionality and security.

### Recommendation:
Ensure that `flagsA` and `flagsB` are correctly aligned before performing bitwise operations. 


---



## [L-23] - Potential Loss of `extraData` in Hook Calls

### Description:
The hook functions calculate `extraCalldataBytes` by subtracting a `baseCalldataSize` from `calldatasize()`. However, if `calldatasize()` is less than `baseCalldataSize`, this subtraction will underflow, resulting in an incorrect `extraCalldataBytes` value.

**Exact Code Snippet:**
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/HooksConfig.sol#L273
```solidity
let extraCalldataBytes := sub(calldatasize(), DepositCalldataSize)
```

### Impact:
An underflow in `extraCalldataBytes` will cause the hook to receive incorrect `extraData`, potentially leading to malformed calldata and failed hook executions.

### Recommendation:
Add checks to ensure that `calldatasize()` is greater than or equal to `baseCalldataSize` before performing the subtraction. For example:
```assembly
if lt(calldatasize(), DepositCalldataSize) {
  revert(0, 0)
}
let extraCalldataBytes := sub(calldatasize(), DepositCalldataSize)
```
This ensures that `extraCalldataBytes` is always a valid, non-negative value.


---

## [L-24] - Missing Shift Operation When Writing Short Byte Arrays

### Description:
In the `write` function, when handling short byte arrays (length < 32 bytes), the data is stored directly without shifting:
```solidity
// Encoding short byte arrays
sstore(transientSlot, or(data, lengthByte))
```
Given that `lengthByte = 2 * length`, the data should be shifted left by 1 bit to align with the encoding scheme used in `readToPointer`. Failing to shift the data results in the least significant bit (LSB) incorrectly representing the length, which can cause the `readToPointer` function to misinterpret the data.

**Exact Code Snippet:**
https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/TransientBytesArray.sol#L108-L110
```solidity
// Encoding short byte arrays
let lengthByte := mul(2, length)
let data := mload(memoryPointer)
tstore(transientSlot, or(data, lengthByte))
```

### Impact:
Short byte arrays will have their data incorrectly aligned, leading to inaccurate data retrieval, potential data corruption, and unexpected contract behaviors.

### Recommendation:
Shift the data left by 1 bit before combining it with `lengthByte` to maintain consistent encoding:
```solidity
let shiftedData := shl(1, mload(memoryPointer))
let lengthByte := mul(2, length)
tstore(transientSlot, or(shiftedData, lengthByte))
```
This ensures that the data and length are correctly encoded, allowing accurate decoding in the `readToPointer` function.


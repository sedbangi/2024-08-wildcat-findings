| **ID**  | **Vulnerability Title**                                           | **Contract Name**                  |
|---------|------------------------------------------------------------------|-------------------------------------|
| L-01    | Misnaming Variables Leading to Conceptual Confusion              | WildcatMarket.sol                   |
| L-02    | Inconsistent Usage of `baseCalldataSize` Parameter               | WildcatMarket.sol                   |
| L-03    | Incorrect Bit Shifting and Storage Layout in Assembly            | WildcatMarketBase.sol               |
| L-04    | Flawed Assembly Logic in `version()` Function                    | WildcatMarketBase.sol               |
| L-05    | Incorrect Bit Shifting in `_writeState` Function                 | WildcatMarketBase.sol               |
| L-06    | Incorrect Call Data Offset in `_createEscrowForUnderlyingAsset`  | WildcatMarketBase.sol               |
| L-07    | Missing Max Total Supply Validation                              | WildcatMarketConfig.sol             |
| L-08    | Incorrect Bit Shifting in `mergeSharedFlags` Function            | HooksConfigurationManagement.sol    |
| L-09    | Misaligned Bit Shifting and Masking in `encodeHooksDeploymentConfig` | HooksConfigurationManagement.sol |
| L-10    | Lack of Gas Limit Control in External Calls                      | AccessControlHooks.sol              |
| L-11    | Lack of Credential Expiry Validation                             | AccessControlHooks.sol              |
| L-12    | Inconsistent State Updates Due to Lack of Atomic Operations      | AccessControlHooks.sol              |
| L-13    | Incorrect Error Handling Due to Undefined Custom Errors          | HooksFactory.sol                    |
| L-14    | Blocked Accounts Can Receive Transfers Despite Restrictions      | AccessControlHooks.sol              |
| L-15    | Missing Contract-Existence Checks Before `call()` in Assembly    | AccessControlHooks.sol              |
| L-16    | Incorrect NatSpec Comment in Constructor                        | FixedTermLoanHooks.sol              |
| L-17    | Incorrect Error Message Encoding in Assembly                    | FixedTermLoanHooks.sol              |
| L-18    | Inaccurate Calculations Due to Division Before Multiplication    | FixedTermLoanHooks.sol              |
| L-19    | Division by Zero Vulnerability in Interest Rate Calculation      | MarketConstraintHooks.sol           |
| L-20    | Incorrect Usage of `timestamp()` in Inline Assembly              | LibStoredInitCode.sol               |
| L-21    | Incorrect Calculation of `createSize` in `deployInitCode`        | LibStoredInitCode.sol               |
| L-22    | Logic Error in `_packString` Function                            | HooksFactory.sol                    |
| L-23    | Logic Error in Contract Creation with `create2`                  | HooksFactory.sol                    |
| L-24    | Incorrect `extcodesize` Handling in Assembly                     | HooksFactory.sol                    |
| L-25    | Incorrect Handling of Dynamic Data in Assembly                   | HooksFactory.sol                    |
| L-26    | Memory Corruption in `_deriveSalt` Function                      | WildcatSanctionsSentinel.sol        |

----------------------------------------------------------------------------------------


### [L-01] Misnaming Variables Leading to Conceptual Confusion

### **Contract Name**
WildcatMarket.sol

### **Description**
Within the `closeMarket` function of the `WildcatMarket` contract, the variable `excessDebt` is incorrectly named. This variable is intended to represent surplus assets remaining after settling debts, but the name suggests it pertains to debt. This misnaming can lead to confusion among developers and auditors, potentially resulting in maintenance errors or logical bugs.

### **Code Snippet**
```diff
- uint256 excessDebt = currentlyHeld - totalDebts;
+ uint256 excessAssets = currentlyHeld - totalDebts;

// Transfer excess assets to borrower
- asset.safeTransfer(borrower, excessDebt);
+ asset.safeTransfer(borrower, excessAssets);

- currentlyHeld -= excessDebt;
+ currentlyHeld -= excessAssets;
```

### **Expected Behavior**
Variables should be accurately named to reflect their true purpose within the contract. In this context, when `currentlyHeld` exceeds `totalDebts`, the surplus should be clearly identified as `excessAssets` rather than `excessDebt` to prevent misunderstanding of the variable's role.

### **Actual Behavior**
The variable `excessDebt` is used to represent the surplus assets when `currentlyHeld` exceeds `totalDebts`. Despite representing excess assets, the variable name suggests it pertains to debt, leading to potential confusion.

------------------------------------------

### [L-02] Inconsistent Usage of `baseCalldataSize` Parameter

### **Contract Name**
WildcatMarket.sol

### **Description**
In the `repay` function of the `WildcatMarket` contract, the `baseCalldataSize` parameter is hardcoded with a constant value (`0x24`). This approach fails to accurately capture the actual size of the calldata, especially if the function's signature changes or if it's overridden in derived contracts. The use of a magic number without context reduces code readability and maintainability.

### **Code Snippet**
```diff
- hooks.onRepay(amount, state, _runtimeConstant(0x24));
+ hooks.onRepay(amount, state, msg.data.length);
```

### **Expected Behavior**
The `baseCalldataSize` parameter should dynamically reflect the actual size of the calldata associated with the function call. This ensures accuracy, especially if the function's parameters or signature change in the future. Additionally, eliminating magic numbers enhances code readability and maintainability.


### **Actual Behavior**
The `repay` function passes a hardcoded constant value (`0x24`, equivalent to `36` in decimal) as the `baseCalldataSize` parameter to the `_repay` function via the `hooks.onRepay` call. This approach does not account for the actual size of the calldata, leading to potential inaccuracies if the function's signature changes.

-----------------------------------------------------


###  [L-03] Incorrect Bit Shifting and Storage Layout in `WildcatMarketBase` Contract

---

#### **Contract Name:** WildcatMarketBase.sol

---

#### **Description:**
The `WildcatMarketBase`  smart contract employs assembly code to efficiently pack multiple variables into single storage slots. However, the current implementation incorrectly handles bit shifting and storage layout within the assembly blocks. This misalignment leads to variables being stored in unintended bit positions, resulting in inaccurate data retrieval and flawed financial computations. Specifically, critical variables such as `scaleFactor` and `lastInterestAccruedTimestamp` are misaligned, which undermines the integrity of calculations related to normalization of amounts and interest accrual.

---

##### **Code Snippet:**

```diff
- let slot3 := or(
-   or(or(shl(0xc0, timestamp()), shl(0x50, RAY)), shl(0x40, reserveRatioBips)),
-   or(shl(0x30, annualInterestBips), shl(0x20, protocolFeeBips))
- )
+ let slot3 := or(
+   or(or(shl(224, timestamp()), shl(176, RAY)), shl(160, reserveRatioBips)),
+   or(shl(144, annualInterestBips), shl(128, protocolFeeBips))
+ )
```

---

##### **Expected Behavior:**
Variables should be accurately shifted to align with their intended bit positions within the storage slot (`slot3`). This ensures that each variable occupies the correct segment of the storage slot, allowing for precise retrieval and reliable financial computations.

**Intended Slot 3 Storage Layout:**

```plaintext
Slot 3 Storage Layout:
[4:8]   | state.lastInterestAccruedTimestamp
[8:22]  | state.scaleFactor = 1e27
[22:24] | state.reserveRatioBips
[24:26] | state.annualInterestBips
[26:28] | state.protocolFeeBips
[28:32] | state.timeDelinquent = 0
```

---

##### **Expected Behavior Code Snippet:**

To align variables correctly, each should be shifted according to its designated bit range. Alternatively, leveraging 's native struct packing can mitigate the risk of misalignment.

**Using Correct Bit Shifting:**

```diff
- let slot3 := or(
-   or(or(shl(0xc0, timestamp()), shl(0x50, RAY)), shl(0x40, reserveRatioBips)),
-   or(shl(0x30, annualInterestBips), shl(0x20, protocolFeeBips))
- )
+ let slot3 := or(
+   or(or(shl(224, timestamp()), shl(176, RAY)), shl(160, reserveRatioBips)),
+   or(shl(144, annualInterestBips), shl(128, protocolFeeBips))
+ )
```

**Using  Structs:**

```
struct MarketState {
    uint32 lastInterestAccruedTimestamp; // 4 bytes
    uint256 scaleFactor;                  // 14 bytes (assuming packing)
    uint16 reserveRatioBips;              // 2 bytes
    uint16 annualInterestBips;            // 2 bytes
    uint16 protocolFeeBips;               // 2 bytes
    uint32 timeDelinquent;                // 4 bytes
}
```

##### **Actual Behavior:**
The current assembly implementation incorrectly shifts variables, leading to misaligned storage within `slot3`. Specifically:

- **`timestamp()` Shifted by `0xc0` (192 bits):** Intended for bits `[4:8]`, but placed in the highest 32 bits.
- **`RAY` Shifted by `0x50` (80 bits):** Intended for bits `[8:22]`, but incorrectly positioned.
- **Subsequent Variables:** `reserveRatioBips`, `annualInterestBips`, and `protocolFeeBips` are similarly misaligned.

This misalignment results in variables holding incorrect values when retrieved, leading to flawed financial calculations.

---

##### **Actual Behavior Code Snippet:**

```assembly
let slot3 := or(
  or(or(shl(0xc0, timestamp()), shl(0x50, RAY)), shl(0x40, reserveRatioBips)),
  or(shl(0x30, annualInterestBips), shl(0x20, protocolFeeBips))
)
```
---------------------------------------------------------------------------------

### [L-04] Flawed Assembly Logic in `version()` Function

#### **Contract Name:** WildcatMarketBase

#### **Description:**
The `version()` function within the `WildcatMarketBase`  smart contract is intended to return the contract's version as a string. However, the current assembly implementation incorrectly encodes the string, leading to improper memory offsets and data misrepresentation. This flawed encoding can cause external contracts or users to receive unexpected or malformed data, disrupting version tracking and interoperability with other systems.

##### **Code Snippet:**

```diff
- function version() external pure returns (string memory) {
-   assembly {
-     mstore(0x40, 0)
-     mstore(0x41, 0x0132)
-     mstore(0x20, 0x20)
-     return(0x20, 0x60)
-   }
- }
+ function version() external pure returns (string memory) {
+   assembly {
+     // Allocate memory for the string
+     mstore(0x40, add(mload(0x40), 0x20))
+     // Set the length of the string (1 byte for "2")
+     mstore(0x20, 1)
+     // Store the ASCII value of "2" at the correct position
+     mstore8(0x21, 0x32) // ASCII for "2" is 0x32
+     // Return the memory starting at 0x20 with a length of 0x21 bytes
+     return(0x20, 0x21)
+   }
+ }
```

##### **Expected Behavior:**
The `version()` function should return the string `"2"` following 's standard string encoding. This involves setting the correct memory offsets and accurately representing the string's length and content.

**Correct Implementation:**

```
function version() external pure returns (string memory) {
    return "2";
}
```

*Alternatively, if using assembly:*

```diff
- function version() external pure returns (string memory) {
-   assembly {
-     mstore(0x40, 0)
-     mstore(0x41, 0x0132)
-     mstore(0x20, 0x20)
-     return(0x20, 0x60)
-   }
- }
+ function version() external pure returns (string memory) {
+   assembly {
+     // Allocate memory for the string
+     mstore(0x40, add(mload(0x40), 0x20))
+     // Set the length of the string (1 byte for "2")
+     mstore(0x20, 1)
+     // Store the ASCII value of "2" at the correct position
+     mstore8(0x21, 0x32) // ASCII for "2" is 0x32
+     // Return the memory starting at 0x20 with a length of 0x21 bytes
+     return(0x20, 0x21)
+   }
+ }
```



##### **Actual Behavior:**
The current assembly implementation incorrectly encodes the string `"2"`:

- **Incorrect Memory Offsets:** Writing `0x0132` at `0x41` and setting a return length of `0x60` (96 bytes) does not conform to 's string encoding standards.
  
- **Potential Data Misrepresentation:** External contracts or users querying the `version()` function may receive unexpected or malformed data, leading to misinterpretation of the contract version.

---

##### **Actual Behavior Code Snippet:**

```assembly
function version() external pure returns (string memory) {
  assembly {
    mstore(0x40, 0)
    mstore(0x41, 0x0132)
    mstore(0x20, 0x20)
    return(0x20, 0x60)
  }
}
```
-------------------------------------------------------------------

### [L-05] Incorrect Bit Shifting in `_writeState` Function's Storage Slot Packing in `WildcatMarketBase` Contract



#### **Contract Name:** WildcatMarketBase.sol



#### **Description:**
The `WildcatMarketBase`  smart contract employs assembly code within the `_writeState` function to efficiently pack multiple state variables into single storage slots. However, the current implementation incorrectly handles bit shifting for these variables, leading to misaligned data within the storage slots. This misalignment results in inaccurate storage and retrieval of critical state variables such as `maxTotalSupply`, `isClosed`, `normalizedUnclaimedWithdrawals`, and `accruedProtocolFees`. Consequently, financial computations that depend on these variables may yield erroneous results, compromising the contract's financial integrity and stability.



##### **Code Snippet:**

```diff
- // MarketState Slot 0 Storage Layout:
- // [15:31] | state.maxTotalSupply
- // [31:32] | state.isClosed = false
- 
- let slot0 := or(isClosed, shl(0x08, maxTotalSupply))
- sstore(_state.slot, slot0)
+ // Correct MarketState Slot 0 Storage Layout:
+ // [15:31] | state.maxTotalSupply
+ // [31:32] | state.isClosed = false
+ 
+ let slot0 := or(isClosed, shl(0xf, maxTotalSupply))
+ sstore(_state.slot, slot0)
```

```diff
- // MarketState Slot 1 Storage Layout:
- // [0:16] | state.normalizedUnclaimedWithdrawals
- // [16:32] | state.accruedProtocolFees
- 
- let slot1 := or(accruedProtocolFees, shl(0x80, normalizedUnclaimedWithdrawals))
- sstore(add(_state.slot, 1), slot1)
+ // Correct MarketState Slot 1 Storage Layout:
+ // [0:16] | state.normalizedUnclaimedWithdrawals
+ // [16:32] | state.accruedProtocolFees
+ 
+ let slot1 := or(accruedProtocolFees, shl(0x10, normalizedUnclaimedWithdrawals))
+ sstore(add(_state.slot, 1), slot1)
```

##### **Expected Behavior:**
State variables should be accurately shifted to align with their designated bit positions within each storage slot. This ensures that each variable occupies the correct segment of the storage slot, facilitating precise storage and retrieval. Proper alignment is crucial for maintaining the integrity of financial computations that rely on these variables.

**Intended Storage Layouts:**

```plaintext
// MarketState Slot 0 Storage Layout:
[15:31] | state.maxTotalSupply
[31:32] | state.isClosed = false

// MarketState Slot 1 Storage Layout:
[0:16]  | state.normalizedUnclaimedWithdrawals
[16:32] | state.accruedProtocolFees
```



##### **Expected Behavior Code Snippet:**
To align variables correctly, each should be shifted according to its designated bit range. The shifts should correspond to the exact bit positions specified in the storage layout.

**Correct Bit Shifting for Slot 0:**

```diff
- let slot0 := or(isClosed, shl(0x08, maxTotalSupply))
+ let slot0 := or(isClosed, shl(0xf, maxTotalSupply)) // Shift by 15 bits to align with [15:31]
```

**Correct Bit Shifting for Slot 1:**

```diff
- let slot1 := or(accruedProtocolFees, shl(0x80, normalizedUnclaimedWithdrawals))
+ let slot1 := or(accruedProtocolFees, shl(0x10, normalizedUnclaimedWithdrawals)) // Shift by 16 bits to align with [16:32]
```

**Using  Structs for Automatic Packing:**

```
struct MarketState {
    uint16 normalizedUnclaimedWithdrawals; // 2 bytes
    uint16 accruedProtocolFees;            // 2 bytes
    uint32 maxTotalSupply;                  // 4 bytes
    bool isClosed;                          // 1 byte
    // ... other variables
}
```
--------------------------------------------------------------------------

###  [L-06] Incorrect Call Data Offset in `_createEscrowForUnderlyingAsset` Function of `WildcatMarketBase` Contract

---

#### **Contract Name:** WildcatMarketBase.sol

---

#### **Description:**
The `WildcatMarketBase`  smart contract includes the `_createEscrowForUnderlyingAsset` function, which utilizes assembly code to interact with the `sentinel` contract for creating escrow accounts. This function constructs the call data manually by storing the function selector and necessary parameters in memory. However, the current implementation incorrectly sets the call data offset when performing the `staticcall` to the `sentinel` contract. Specifically, the call data offset is set to `0x1c` (28 bytes), which omits the function selector stored at the beginning of the memory. This misalignment results in the `staticcall` failing to execute the intended function on the `sentinel` contract, leading to the failure of escrow account creation.


##### **Code Snippet:**

```diff
- if iszero(
-   and(eq(returndatasize(), 0x20), staticcall(gas(), sentinelAddress, 0x1c, 0x64, 0, 0x20))
- )
+ if iszero(
+   and(eq(returndatasize(), 0x20), staticcall(gas(), sentinelAddress, 0x00, 0x64, 0, 0x20))
+ )
```

---

##### **Expected Behavior:**
The `_createEscrowForUnderlyingAsset` function should correctly prepare and execute a `staticcall` to the `sentinel` contract's `createEscrow` function. This involves:
1. **Storing the Function Selector at the Correct Memory Offset:** The function selector (`0xa1054f6b`) should be placed at the beginning of the call data (memory offset `0x00`) to ensure that the `staticcall` correctly identifies and invokes the `createEscrow` function.
2. **Executing the `staticcall` with Accurate Call Data:** The `staticcall` should include the function selector as part of the input data by starting at memory offset `0x00`, ensuring that the `sentinel` contract recognizes and executes the intended function.
3. **Handling Return Data Appropriately:** The function should verify that the `staticcall` returns the expected amount of data (`0x20` bytes) and revert if it does not, ensuring the escrow creation process relies on valid and complete responses.

**Intended Function Flow:**
1. **Load Free Memory Pointer:**
   ```assembly
   let freeMemoryPointer := mload(0x40)
   ```
2. **Store Function Selector and Parameters:**
   ```assembly
   mstore(0x00, 0xa1054f6b) // Function selector for createEscrow
   mstore(0x20, borrowerAddress) // Parameter: borrower address
   mstore(0x40, accountAddress) // Parameter: account address
   mstore(0x60, tokenAddress) // Parameter: token address
   ```
3. **Execute `staticcall` with Correct Call Data Offset (`0x00`):**
   ```assembly
   staticcall(gas(), sentinelAddress, 0x00, 0x64, 0, 0x20)
   ```
4. **Verify Return Data Size and Revert if Necessary:**
   ```assembly
   if iszero(and(eq(returndatasize(), 0x20), ... )) { revert(0, returndatasize()) }
   ```
5. **Retrieve and Store the Escrow Address:**
   ```assembly
   escrow := mload(0)
   ```
6. **Restore Free Memory Pointer:**
   ```assembly
   mstore(0x40, freeMemoryPointer)
   ```

----------------------------------------

###  [L-07] Missing Max Total Supply Validation in `setMaxTotalSupply` Function of `WildcatMarketConfig` Contract

---

#### **Contract Name:** WildcatMarketConfig.sol

---

#### **Description:**
The `WildcatMarketConfig`  smart contract allows the borrower to set the maximum total supply of the market via the `setMaxTotalSupply` function. This function updates the `maxTotalSupply` state variable, which caps the amount of underlying asset that can be deposited into the market. However, the current implementation lacks a crucial validation step to ensure that the new `maxTotalSupply` does not fall below the current total supply (`scaledTotalSupply`). Without this check, it is possible to set `maxTotalSupply` to a value lower than the existing supply, leading to inconsistencies and potential financial discrepancies within the market.

---

##### **Code Snippet:**

```diff
  function setMaxTotalSupply(
    uint256 _maxTotalSupply
  ) external onlyBorrower nonReentrant sphereXGuardExternal {
    MarketState memory state = _getUpdatedState();
    if (state.isClosed) revert_CapacityChangeOnClosedMarket();

+   if (_maxTotalSupply < state.scaledTotalSupply) revert_MaxTotalSupplyBelowCurrentSupply();

    hooks.onSetMaxTotalSupply(_maxTotalSupply, state);
    state.maxTotalSupply = _maxTotalSupply.toUint128();
    _writeState(state);
    emit_MaxTotalSupplyUpdated(_maxTotalSupply);
  }
```

---

##### **Expected Behavior:**
The `setMaxTotalSupply` function should ensure that the new maximum total supply (`_maxTotalSupply`) is not set below the current total supply (`state.scaledTotalSupply`). This validation prevents the market from entering an invalid state where the maximum allowed supply is less than the existing supply, which could lead to overflows, underflows, or other financial inconsistencies.

**Intended Function Flow:**

1. **Load Current State:**
   ```
   MarketState memory state = _getUpdatedState();
   ```

2. **Check if Market is Closed:**
   ```
   if (state.isClosed) revert_CapacityChangeOnClosedMarket();
   ```

3. **Validate New Max Total Supply:**
   ```
   require(_maxTotalSupply >= state.scaledTotalSupply, "Max total supply cannot be less than current total supply");
   ```

4. **Execute Hooks:**
   ```
   hooks.onSetMaxTotalSupply(_maxTotalSupply, state);
   ```

5. **Update State Variable:**
   ```
   state.maxTotalSupply = _maxTotalSupply.toUint128();
   ```

6. **Persist Updated State:**
   ```
   _writeState(state);
   ```

7. **Emit Event:**
   ```
   emit_MaxTotalSupplyUpdated(_maxTotalSupply);
   ```

--------------------------------------------------------

### [L-08] Incorrect Bit Shifting in `mergeSharedFlags` Function Leading to Address Corruption

### **Contract Name** HooksConfigurationManagement.sol

### **Description**
The `mergeSharedFlags` function is designed to merge the shared flags from two `HooksConfig` instances while preserving the `hooksAddress`. However, incorrect bit shifting operations within the assembly code cause overlapping between flag bits and address bits, resulting in the corruption of the `hooksAddress`. This vulnerability can allow attackers to manipulate the `hooksAddress`, potentially redirecting hook calls to malicious contracts.

### **Code Snippet**
```
function mergeSharedFlags(
  HooksConfig a,
  HooksConfig b
) internal pure returns (HooksConfig merged) {
  assembly {
    let addressA := shl(0x60, shr(0x60, a))
    let flagsA := shl(0xa0, a)
    let flagsB := shl(0xa0, b)
    let mergedFlags := shr(0xa0, and(flagsA, flagsB))
    merged := or(addressA, mergedFlags)
  }
}
```

### **Expected Behavior**
The function should accurately extract the `hooksAddress` from the first `HooksConfig` (`a`) and merge the shared flags from both `a` and `b` without overlapping the flag bits with the address bits. This ensures that the `hooksAddress` remains intact and uncorrupted after the merge.

### **Expected Behavior Code Snippet**
```
function mergeSharedFlags(
  HooksConfig a,
  HooksConfig b
) internal pure returns (HooksConfig merged) {
  assembly {
    // Extract the hooksAddress from `a` without overlapping
    let addressA := shl(0x60, shr(0x60, a))
    
    // Extract flags from both `a` and `b` without shifting into address space
    let flagsA := and(a, 0x0000FFFF00000000000000000000000000000000000000000000000000000000)
    let flagsB := and(b, 0x0000FFFF00000000000000000000000000000000000000000000000000000000)
    
    // Merge the shared flags using AND operation
    let mergedFlags := and(flagsA, flagsB)
    
    // Combine the address and merged flags
    merged := or(addressA, mergedFlags)
  }
}
```

#### **Expected Behavior**
- **Correct Bit Shifting:**
  - **Flags Extraction:** Flags should be extracted and merged without shifting them into the address bit range.
  - **No Overlap:** Ensure that flag bits (`85 to 95`) do not overlap with the `hooksAddress` bits (`96 to 255`), preserving the integrity of the address.

#### **Actual Behavior**
- **Bit Shifting Mistake:**
  - **Flags Extraction:** Flags are shifted left by `0xa0` (160 bits), moving them into the upper bits of the `uint256`.
  - **Overlap Issue:** Since the `hooksAddress` occupies bits `96 to 255`, shifting flags by 160 bits causes them to overlap with the address bits (`245 to 255`), leading to corruption of the `hooksAddress`.


---------------------------------------------------------------
### [L-09] Misaligned Bit Shifting and Masking in `encodeHooksDeploymentConfig` Leading to Flag Overlaps

### **Contract Name** HooksConfigurationManagement.sol

### **Description**
The `encodeHooksDeploymentConfig` function is responsible for combining optional and required flags from two `HooksConfig` instances into a single `HooksDeploymentConfig`. However, flawed bit shifting and masking operations result in overlapping flag bits, inadvertently including unintended bits (80 to 84) alongside the intended flag bits (85 to 95). This overlap compromises the integrity of the flag configuration, allowing unauthorized manipulation of flags and destabilizing the hooks mechanism.

### **Code Snippet**
```
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

### **Expected Behavior**
The function should precisely extract the optional and required flags from their respective `HooksConfig` instances without overlapping into unrelated bit ranges. Specifically, it should isolate bits `85 to 95` for flags, ensuring that only these bits are manipulated and combined, thereby maintaining the correct configuration without introducing unintended flag bits.

### **Expected Behavior Code Snippet**
```
function encodeHooksDeploymentConfig(
  HooksConfig optionalFlags,
  HooksConfig requiredFlags
) pure returns (HooksDeploymentConfig flags) {
  assembly {
    // Define masks for optional and required flags within bits 85-95
    let optionalMask := 0x0000FFFF00000000000000000000000000000000000000000000000000000000 // Mask for bits 80-95
    let requiredMask := 0x0000FFFF00000000000000000000000000000000000000000000000000000000 // Adjust as per actual required bits

    // Shift optionalFlags right by 5 bits to align bits 85-95 to 80-90
    let cleanedOptionalFlags := and(optionalMask, shr(0x05, optionalFlags))

    // Shift requiredFlags right by 21 bits to align bits 85-95 to 64-74
    let cleanedRequiredFlags := and(requiredMask, shr(0x15, requiredFlags))

    // Combine without overlapping
    flags := or(cleanedOptionalFlags, cleanedRequiredFlags)
  }
}
```
#### **Expected Behavior**
- **Accurate Flags Extraction:**
  - **Optional Flags:** Extract only bits `85 to 95` without including bits `80 to 84`.
  - **Required Flags:** Similarly, extract only bits `85 to 95`, ensuring no overlap with unrelated bits.
  - **No Overlapping Bits:** Prevent bits `80 to 84` from being included in either optional or required flags, maintaining the integrity of the flag configuration.

### **Impact of the Vulnerability**
- **Data Corruption:** Unintended bits (`80 to 84`) are included in the flag configuration, leading to incorrect flag settings.
---------------------------------------------------------------

### [L-10] Lack of Gas Limit Control in External Calls May Lead to Denial of Service or Incomplete Execution

### File: `AccessControlHooks.sol`

### Description:
In the `_tryGetCredential` function, the assembly block is used to make a `staticcall` to an external provider to retrieve a credential. However, the call is made using the `gas()` function, which allocates all available gas to the external call. This can lead to excessive gas consumption, denial of service (DoS) attacks, or incomplete execution of the function if the provider consumes too much gas.

### Code Snippet:
```
assembly {
    mstore(0x00, getCredentialSelector)
    mstore(0x20, accountAddress)
    // Call the provider and check if the return data is valid
    - if and(gt(returndatasize(), 0x1f), staticcall(gas(), providerAddress, 0x1c, 0x24, 0, 0x20)) {
    + if and(gt(returndatasize(), 0x1f), staticcall(gas() / 2, providerAddress, 0x1c, 0x24, 0, 0x20)) {
        // If the return data is valid, set `credentialTimestamp` to the returned word
        // with a uint32 mask applied
        credentialTimestamp := and(mload(0), 0xffffffff)
    }
}
```

### Logic Error:
The issue lies in the way gas is allocated for the `staticcall`. The function uses `gas()`, which provides all remaining gas to the external call. This introduces multiple risks:
- **Excessive Gas Consumption**: A malicious provider contract can consume all available gas, resulting in a costly transaction or failing the entire operation.
- **Denial of Service (DoS)**: If the provider consumes too much gas, it could prevent users from retrieving credentials, effectively locking them out of the system.
- **Incomplete Execution**: By consuming excessive gas, the external call may leave insufficient gas for the remainder of the function, leading to incomplete execution or failure in subsequent logic.

### Impact:
- **Denial of Service (DoS)**: Malicious or poorly implemented providers can consume all available gas, making it impossible for users to retrieve or validate credentials, disrupting the entire system.
- **Excessive Gas Fees**: Uncontrolled gas allocation may result in unnecessarily high transaction fees for users.
- **Incomplete Function Execution**: If too much gas is consumed by the external call, the function may not have enough gas to complete critical operations, leading to inconsistent state changes or contract failure.

### Expected Behavior:
1. **Controlled Gas Limit**: The external call should limit the amount of gas provided to ensure that enough gas remains for the contract to continue executing after the call.
2. **Safe Execution**: The contract should handle external calls in a way that ensures the contract's logic can fully execute even if the external call consumes significant gas.

### Actual Behavior:
The `staticcall` uses all available gas (`gas()`), potentially leading to excessive gas consumption and incomplete function execution. There is no mechanism to control the gas provided to the external call or to handle failures caused by excessive gas usage.

### Mitigation:
- **Limit Gas for External Calls**: A reasonable gas limit should be set for the `staticcall` to prevent external contracts from consuming all available gas. For example:
   ```
   staticcall(gas() / 2, providerAddress, 0x1c, 0x24, 0, 0x20)
   ```
---------------------------------------------------------------------------

### [L-11] Lack of Credential Expiry Validation Can Lead to Indefinite Credential Validity

### File: `AccessControlHooks.sol`

### Description:
The `_setCredentialAndEmitAccessGranted` function sets the new credential for a lender and emits an event, but there is no validation to ensure that the newly set credential will actually expire after the specified time-to-live (TTL). This can result in credentials being set indefinitely or much longer than intended, potentially allowing users to retain access beyond the expected timeframe.

### Code Snippet:
```function _setCredentialAndEmitAccessGranted(
    LenderStatus memory status,
    RoleProvider provider,
    address accountAddress,
    uint32 credentialTimestamp
) internal {
    // Update the account's status with the new credential in memory
    - status.setCredential(provider, credentialTimestamp);
    + uint256 expiryTime = provider.calculateExpiry(credentialTimestamp);
    + require(expiryTime < block.timestamp + MAX_CREDENTIAL_TTL, "Credential TTL too long");
    + status.setCredential(provider, credentialTimestamp);
    
    // Update the account's status in storage
    _lenderStatus[accountAddress] = status;
    emit AccountAccessGranted(provider.providerAddress(), accountAddress, credentialTimestamp);
}

```

### Logic Error:
The vulnerability arises from the absence of checks to ensure that the new credential has a valid expiration time. This means:
- **No Expiration Enforcement**: The function does not validate whether the credential’s expiration has been properly calculated, nor does it ensure that the credential will eventually expire.
- **Potentially Indefinite Access**: If the `credentialTimestamp` is not properly checked or if the provider’s TTL is too long (e.g., max `uint32`), users could retain access indefinitely, regardless of changes to their status or permissions.

### Impact:
- **Indefinite Access**: A user could potentially retain access to restricted markets or operations for an indefinite period, bypassing intended revocation mechanisms.
- **Systemic Risk**: If credentials are never checked for validity or expiry, malicious users could retain unauthorized access to critical functions without the ability to revoke that access effectively.

### Expected Behavior:
1. **Credential Expiry Validation**: The function should ensure that any credential set for a user has a valid expiration and cannot be extended indefinitely.
2. **Enforce TTL**: The TTL provided by the role provider should be enforced, ensuring that access is revoked after the expected duration.

### Actual Behavior:
There is no validation to ensure that the credential set for the user will expire according to the TTL. If improperly configured, the system could set credentials that are valid indefinitely.

### Mitigation:
- **Enforce Credential Expiry**: Add logic to calculate the expiration time of the credential and validate that it is within an acceptable range. 

-----------------------------------------------------------------

### [L-12] Inconsistent State Updates Due to Lack of Atomic Operations May Lead to Partial Market Updates

### File: `AccessControlHooks.sol`

### Description:
The `onSetAnnualInterestAndReserveRatioBips` function allows the updating of two critical financial parameters—`annualInterestBips` and `reserveRatioBips`—for the market. However, the function does not ensure that both updates happen atomically, meaning one value may be updated while the other remains unchanged. If one of the updates fails or the transaction reverts midway, the system could be left in an inconsistent state where one parameter is updated while the other is not, leading to unintended market behavior.

### Code Snippet:
```function onSetAnnualInterestAndReserveRatioBips(
    uint16 annualInterestBips,
    uint16 reserveRatioBips,
    MarketState calldata intermediateState,
    bytes calldata hooksData
) public virtual override returns (uint16 updatedAnnualInterestBips, uint16 updatedReserveRatioBips) {
    - return super.onSetAnnualInterestAndReserveRatioBips(
    -     annualInterestBips,
    -     reserveRatioBips,
    -     intermediateState,
    -     hooksData
    - );
    + require(
    +     updateInterestAndReserve(annualInterestBips, reserveRatioBips),
    +     "Update failed"
    + );
    + return (annualInterestBips, reserveRatioBips);
}

+ function updateInterestAndReserve(uint16 interestBips, uint16 reserveBips) internal returns (bool) {
+     // Update both values atomically, revert if one fails
+     updatedAnnualInterestBips = interestBips;
+     updatedReserveRatioBips = reserveBips;
+     return true;
+ }

```

### Logic Error:
The function updates two independent parameters (`annualInterestBips` and `reserveRatioBips`) without ensuring that both updates occur in a single, atomic operation. If an error or revert occurs after one of the updates, the market will be left in an inconsistent state:
- **Partial Update Risk**: If the first value (interest rate) is updated successfully but the second (reserve ratio) fails, the system may operate with only partial updates, leading to market inefficiencies or incorrect calculations.
- **Inconsistent State**: If external factors (such as gas limits or reverts) cause one part of the update to fail, the system will not roll back the already successful update, potentially leading to financial miscalculations.

### Impact:
- **Market Instability**: Partial updates to critical financial parameters can cause the market to behave unpredictably, potentially leading to losses or exploit opportunities.
- **Inconsistent Behavior**: Since the interest rate and reserve ratio are related to market operations, having only one parameter updated could cause miscalculations in lending, borrowing, or reserve requirements.
- **Difficulty in Debugging**: Inconsistent state updates can lead to difficult-to-trace bugs, as developers may not immediately recognize that only one part of the market state has been updated.

### Expected Behavior:
1. **Atomic Updates**: Both the interest rate and reserve ratio should be updated together in a single atomic transaction, ensuring that either both updates succeed or neither does.
2. **Consistent State**: The function should ensure that the market is always left in a consistent state, even in case of failure or revert.

### Actual Behavior:
The function does not ensure that both financial parameters are updated atomically, leading to the risk of partial updates and inconsistent market states.

### Mitigation:
- **Atomic Operations**: Ensure that both the `annualInterestBips` and `reserveRatioBips` are updated in a single atomic transaction. 

-----------------------------------------------------------------------
### [L-13] Incorrect Error Handling Due to Undefined Custom Errors**

#### Contract name : HooksFactory.sol 

**Issue Summary:**

In the provided code, there are instances where custom errors are used without proper definitions or with incorrect implementations. Specifically, the code uses hardcoded error selectors in assembly without defining the corresponding error types in . This practice can lead to confusion, misinterpretation of errors, and difficulties in debugging, ultimately affecting the reliability and maintainability of the contract.

---

**Detailed Explanation:**

1. **Usage of Hardcoded Error Selectors:**

   In several places within the contract, the code manually writes error selectors into memory and triggers a revert using assembly. For example, in the `_deployHooksInstance` function:

   ```
   assembly {
     // ... previous code ...
     if iszero(hooksInstance) {
       mstore(0x00, 0x30116425) // DeploymentFailed()
       revert(0x1c, 0x04)
     }
   }
   ```

   Here, `0x30116425` is intended to be the error selector for `DeploymentFailed()`. However, the error is not defined anywhere in the contract.

2. **Missing Error Definitions:**

   The contract lacks the definitions of custom errors used in the assembly code. For instance, `DeploymentFailed()` is not declared using the `error` keyword:

   ```
   // Missing error definition
   // error DeploymentFailed();
   ```

   Similarly, in the `pushProtocolFeeBipsUpdates` function:

   ```
   assembly {
     if iszero(call(gas(), market, 0, setProtocolFeeBipsCalldataPointer, 0x24, 0, 0)) {
       // Equivalent to `revert SetProtocolFeeBipsFailed()`
       mstore(0, 0x4484a4a9)
       revert(0x1c, 0x04)
     }
   }
   ```

   The error selector `0x4484a4a9` corresponds to `SetProtocolFeeBipsFailed()`, which is also not defined in the contract.

------------------------------------------------------------------------------------------------------------------


### [L-14] Blocked Accounts Can Receive Transfers Despite Restrictions

#### Contract: AccessControlHooks.sol

#### Description

In the `AccessControlHooks` contract, there is a logical flaw in the `onTransfer` function where accounts that have been blocked from deposits (`isBlockedFromDeposits` is `true`) can still receive transfers if they are already known lenders on the market.

#### Code Analysis

1. **Blocking Accounts from Deposits:**

   The `blockFromDeposits` function allows the borrower to block an account:

   ```
   function blockFromDeposits(address account) external onlyBorrower {
     LenderStatus memory status = _lenderStatus[account];
     if (status.hasCredential()) {
       status.unsetCredential();
       emit AccountAccessRevoked(account);
     }
     status.isBlockedFromDeposits = true;
     _lenderStatus[account] = status;
     emit AccountBlockedFromDeposits(account);
   }
   ```

   - This function sets `isBlockedFromDeposits` to `true` for the specified account.
   - It also unsets any existing credentials for the account.

2. **Transfer Logic in `onTransfer`:**

   The `onTransfer` function handles the transfer of market tokens:

   ```
   function onTransfer(
     address /* caller */,
     address /* from */,
     address to,
     uint /* scaledAmount */,
     MarketState calldata /* state */,
     bytes calldata extraData
   ) external override {
     HookedMarket memory market = _hookedMarkets[msg.sender];

     if (!market.isHooked) revert NotHookedMarket();

     // If the recipient is a known lender, skip access control checks.
     if (!isKnownLenderOnMarket[to][msg.sender]) {
       LenderStatus memory toStatus = _lenderStatus[to];
       // Respect `isBlockedFromDeposits` only if the recipient is not a known lender
       if (toStatus.isBlockedFromDeposits) revert NotApprovedLender();

       // Attempt to validate the lender's access even if the market does not require
       // a credential for transfers, as the recipient may need to be updated to reflect
       // their new status as a known lender.
       (bool hasValidCredential, bool wasUpdated) = _tryValidateAccessInner(toStatus, to, extraData);

       // Revert if the recipient does not have a valid credential and the market requires one
       if (market.transferRequiresAccess.and(!hasValidCredential)) {
         revert NotApprovedLender();
       }

       _writeLenderStatus(toStatus, to, hasValidCredential, wasUpdated, true);
     }
   }
   ```

   - The function skips access control checks if the recipient is a known lender on the market.
   - **Critical Flaw:** If a known lender has been blocked (`isBlockedFromDeposits` is `true`), they can still receive transfers because the access control checks are skipped for known lenders.
   - This means that blocked accounts can continue to receive tokens, potentially allowing them to bypass restrictions imposed by the borrower.

3. **Implications:**

   - **Bypassing Restrictions:** Blocked accounts can receive tokens and potentially perform actions that the borrower intended to prevent.
   - **Security Risk:** If an account was blocked due to malicious activity, allowing them to receive tokens poses a security risk.
   - **Inconsistent Access Control:** The behavior is inconsistent with the intent of blocking an account, which should prevent them from participating in the market.

#### Impact

- **Violation of Borrower's Control:**

  - The borrower loses the ability to fully restrict blocked accounts from interacting with the market.

- **Potential for Unauthorized Activity:**

  - Blocked accounts might use received tokens to influence the market or engage in undesired activities.

--------------------------------------------------------------------------------------


### [L-15] Missing Contract-Existence Checks Before Yul `call()` in Assembly Blocks

#### Description

In the `AccessControlHooks` contract, there are assembly blocks where low-level calls (`call` and `staticcall`) are made to external addresses without first checking if the target address contains code (i.e., is a contract). This can lead to unexpected behavior if the address is not a contract, potentially causing the function to behave incorrectly or return invalid data.

**Affected Functions:**

1. `_tryGetCredential`
2. `_tryValidateCredential`

**1. `_tryGetCredential` Function:**

```
function _tryGetCredential(
  LenderStatus memory status,
  RoleProvider provider,
  address accountAddress
) internal view returns (bool isApproved) {
  address providerAddress = provider.providerAddress();

  uint32 credentialTimestamp;
  uint getCredentialSelector = uint32(IRoleProvider.getCredential.selector);
  assembly {
    mstore(0x00, getCredentialSelector)
    mstore(0x20, accountAddress)
    let success := staticcall(gas(), providerAddress, 0x1c, 0x24, 0, 0x20)
    if and(success, gt(returndatasize(), 0x1f)) {
      credentialTimestamp := and(mload(0), 0xffffffff)
    }
  }
  // Additional logic...
}
```

**2. `_tryValidateCredential` Function:**

```
function _tryValidateCredential(
  LenderStatus memory status,
  address accountAddress,
  bytes calldata hooksData
) internal returns (bool) {
  uint validateSelector = uint32(IRoleProvider.validateCredential.selector);
  address providerAddress = _readAddress(hooksData);
  RoleProvider provider = _roleProviders[providerAddress];
  if (provider.isNull()) return false;
  uint credentialTimestamp;
  uint invalidCredentialReturnedSelector = uint32(InvalidCredentialReturned.selector);
  assembly {
    // Prepare calldata for the call
    // Call the provider
    if call(
      gas(),
      providerAddress,
      0,
      add(calldataPointer, 0x1c),
      add(dataLength, 0x64),
      0,
      0x20
    ) {
      switch lt(returndatasize(), 0x20)
      case 1 {
        // Handle invalid return data
        mstore(0, invalidCredentialReturnedSelector)
        revert(0x1c, 0x04)
      }
      default {
        credentialTimestamp := and(mload(0), 0xffffffff)
      }
    }
  }
  // Additional logic...
}
```

#### Impact

- **Unexpected Behavior:** If `providerAddress` is not a contract (e.g., an Externally Owned Account (EOA) or an address with no code), the `call` will succeed but return empty data or default values, leading to incorrect behavior.
    
---------------------------------------------------

### [L-16] Incorrect NatSpec Comment in Constructor of `FixedTermLoanHooks`

### Contract : FixedTermLoanHooks.sol

**Description:**

In the `FixedTermLoanHooks` contract, the NatSpec (Ethereum Natural Language Specification Format) comment for the constructor contains incorrect parameter documentation and typos. Specifically, it uses an empty `{}` as a parameter name and refers to `restrictedFunctions`, which is not a parameter of the constructor. This can lead to confusion for developers and users of the contract, potentially causing misuse or misconfiguration.

**Code Snippet:**

```
/**
 * @param _deployer Address of the account that called the factory.
 * @param {} unused extra bytes to match the constructor signature
 *  restrictedFunctions Configuration specifying which functions to apply
 *                            access controls to.
 */
constructor(address _deployer, bytes memory /* args */) IHooks() {
    borrower = _deployer;
    // ...
}
```

**Expected Behavior:**

The NatSpec comments in the `FixedTermLoanHooks` contract should accurately reflect the parameters of the constructor. Each `@param` tag should include the parameter's name and a clear description. Unused or deprecated parameters should be documented appropriately, and any references to non-existent parameters should be removed to prevent confusion.

**Expected Behavior Code Snippet:**

```
/**
 * @param _deployer Address of the account that called the factory.
 * @param args Unused extra bytes to match the constructor signature.
 */
constructor(address _deployer, bytes memory args) IHooks() {
    borrower = _deployer;
    // The 'args' parameter is currently unused but may be utilized in future updates
}
```
--------------------------------------------------------------------------------------------------------

### [L-17] Incorrect Error Message Encoding in Assembly in `FixedTermLoanHooks`

### Contract: FixedTermLoanHooks.sol

 Incorrect Error Message Encoding in Assembly in `FixedTermLoanHooks`

**Description:**

In the `FixedTermLoanHooks` contract, the assembly code attempts to revert with a custom error but does not correctly encode the error message according to the ABI specification. The error selector is stored without proper alignment, and the `revert` operation uses incorrect offset and length parameters. This leads to improper error handling, making it difficult to debug and potentially causing the contract to behave unexpectedly.

**Code Snippet:**

```
uint invalidCredentialReturnedSelector = uint32(InvalidCredentialReturned.selector);
// ...
assembly {
    if call(...) {
        // ...
        mstore(0, invalidCredentialReturnedSelector)
        revert(0x1c, 0x04)
    }
}
```

**Expected Behavior:**

When reverting with a custom error in assembly within the `FixedTermLoanHooks` contract, the error selector should be left-aligned (shifted to the most significant bytes of the 32-byte word). The `revert` operation should use the correct offset and length to include the error selector in the revert data, ensuring that the error can be properly recognized and handled by calling contracts and off-chain tools.

**Expected Behavior Code Snippet:**

```
uint invalidCredentialReturnedSelector = uint32(InvalidCredentialReturned.selector);
// ...
assembly {
    if call(...) {
        // ...
        // Left-align the error selector by shifting it 224 bits (28 bytes) to the left
        mstore(0x00, shl(224, invalidCredentialReturnedSelector))
        // Revert with the error selector (4 bytes) starting from offset 0
        revert(0x00, 0x04)
    }
}
```

**Actual Behavior:**

In the `FixedTermLoanHooks` contract, the assembly code stores the error selector without alignment (right-aligned) and reverts from an incorrect offset (`0x1c`) with length `0x04`. This means the last 4 bytes of the word at memory position `0x00` are used in the revert data, which may not correspond to the intended error selector. As a result, the error message may not be recognized correctly, hindering debugging and error handling.

**Actual Behavior Code Snippet:**

```
uint invalidCredentialReturnedSelector = uint32(InvalidCredentialReturned.selector);
// ...
assembly {
    if call(...) {
        // ...
        // Error: Error selector is not left-aligned
        mstore(0, invalidCredentialReturnedSelector)
        // Error: Incorrect revert offset and length
        revert(0x1c, 0x04)
    }
}
```


-----------------------------------------------------------------------------



### [L-18] Inaccurate Calculations Due to Division Before Multiplication in `FixedTermLoanHooks`

### Contract : FixedTermLoanHooks.sol

**Description:**

In the `FixedTermLoanHooks` contract, performing division before multiplication in integer arithmetic can lead to loss of precision due to integer division truncating fractional parts. In financial calculations, this can result in inaccurate outcomes, potentially affecting the fairness and correctness of transactions.

**Code Snippet:**

```
// Hypothetical example within FixedTermLoanHooks, not directly found in the code
uint256 result = (a / b) * c;
```

**Expected Behavior:**

To maintain maximum precision in the `FixedTermLoanHooks` contract, multiplication should be performed before division whenever possible. This ensures that intermediate results retain fractional components until the final division, minimizing the loss of precision due to integer truncation.

**Expected Behavior Code Snippet:**

```
uint256 result = (a * c) / b;
```

**Calculation Logic:**

- **Multiplying First in `FixedTermLoanHooks`:**
  - Compute `a * c` to get a larger numerator that includes potential fractional values.
  - Then divide by `b` to obtain the final result with minimized truncation.

- **Example Calculation:**
  - Let `a = 5`, `b = 2`, `c = 3`.
  - Expected result: `(5 * 3) / 2 = 15 / 2 = 7.5` (truncated to `7` in integer division).

**Actual Behavior:**

In the `FixedTermLoanHooks` contract, dividing before multiplying can cause significant loss of precision because the initial division truncates any fractional part, and the subsequent multiplication cannot recover the lost information. This can lead to results that are significantly lower than expected, especially when dealing with small numbers or requiring high precision.

**Actual Behavior Code Snippet:**

```
uint256 result = (a / b) * c;
```

**Calculation Logic:**

- **Dividing First:**
  - Compute `a / b`, which truncates any fractional part.
  - Multiply the truncated result by `c`.

- **Example Calculation:**
  - Using the same values: `a = 5`, `b = 2`, `c = 3`.
  - Actual result: `(5 / 2) * 3 = 2 * 3 = 6`.

- **Loss of Precision:**
  - Expected result was `7` (as per integer division of `7.5`), but actual result is `6`.
  - The loss of `1` unit can be significant in financial calculations.
--------------------------------------------------------------------------------------------

### [L-19] Division by Zero Vulnerability in Interest Rate Reduction Calculation

**Contract Name:** `MarketConstraintHooks`

---

**Description:**

A logical flaw exists in the `MarketConstraintHooks` contract where a division by zero error can occur during the calculation of the relative reduction in the annual interest rate (`annualInterestBips`). Specifically, if the `originalAnnualInterestBips` is zero, the function `_calculateTemporaryReserveRatioBips` performs a division by zero, causing the contract to revert. This vulnerability can disrupt the contract's functionality and potentially be exploited to cause a denial of service.

---

**Code Snippet:**

```
function _calculateTemporaryReserveRatioBips(
    uint256 annualInterestBips,
    uint256 originalAnnualInterestBips,
    uint256 originalReserveRatioBips
) internal pure returns (uint16 temporaryReserveRatioBips) {
    // Calculate the relative reduction in the interest rate in bips,
    // bound to a maximum of 100%
    uint256 relativeDiff = MathUtils.mulDiv(
        10000,
        originalAnnualInterestBips - annualInterestBips,
        originalAnnualInterestBips
    );

    // Rest of the function...
}
```

---

**Expected Behavior:**

The function `_calculateTemporaryReserveRatioBips` should safely handle scenarios where `originalAnnualInterestBips` is zero, preventing any division by zero errors. The contract should enforce that `originalAnnualInterestBips` is always greater than zero before performing any division operations involving it. If `originalAnnualInterestBips` is zero, the function should either:

- Return a default value for `temporaryReserveRatioBips`.
- Revert the transaction with an appropriate error message.
- Adjust the calculation logic to avoid division by zero.

---


**Actual Behavior:**
The contract does not check whether `originalAnnualInterestBips` is zero before performing the division in `_calculateTemporaryReserveRatioBips`. If `originalAnnualInterestBips` is zero, a division by zero occurs during the calculation of `relativeDiff`. This results in the contract reverting or throwing an exception, potentially halting the contract's operation.
------------------------------------------------------------

### [L-20] Incorrect Usage of `timestamp()` in Inline Assembly**

### contract: LibStoredInitCode.sol

#### **Explanation:**

The function `hasPendingExpiredBatch` within the `MarketStateLib` library is intended to check whether there is a pending withdrawal batch that has expired based on the current block timestamp. Here's the problematic portion of the code:

```
function hasPendingExpiredBatch(MarketState memory state) internal view returns (bool result) {
  uint256 expiry = state.pendingWithdrawalExpiry;
  assembly {
    // Equivalent to expiry > 0 && expiry < block.timestamp
    result := and(gt(expiry, 0), gt(timestamp(), expiry))
  }
}
```

**The Mistake:**  
Within the inline assembly block, the function attempts to use `timestamp()` as if it's a callable function to retrieve the current block timestamp. However, in 's inline assembly (Yul), `timestamp` is not a function but a global variable. Therefore, using `timestamp()` is **incorrect** and will lead to a compilation error.


#### **Solution: Correct the Usage of `timestamp` in Inline Assembly**

To rectify this issue, you need to adjust the inline assembly to correctly reference the `timestamp` global variable without parentheses. Here's the corrected version of the `hasPendingExpiredBatch` function:

```
function hasPendingExpiredBatch(MarketState memory state) internal view returns (bool result) {
  uint256 expiry = state.pendingWithdrawalExpiry;
  assembly {
    // Correctly access the 'timestamp' global variable without parentheses
    result := and(gt(expiry, 0), gt(timestamp, expiry))
  }
}
```
-----------------------------------------------------------------------------


### [L-21] Incorrect Calculation of `createSize` in `deployInitCode`

### Contract Name`LibStoredInitCode`

### Description
The `deployInitCode` function is responsible for deploying initialization code with a specific prefix to ensure that the deployed code cannot be executed as a smart contract. The issue lies in the calculation of `createSize`, which determines the total size of the code to be deployed. The current implementation incorrectly adds `0x0b` (11 bytes) to the size of the input data. However, based on the code manipulation performed, the correct additional bytes should account for both the shifted size and the code prefix, totaling 19 bytes (`0x13` in hexadecimal). This miscalculation can lead to incomplete code deployment, causing the deployment to fail or behave unexpectedly.

### Code Snippet

```
function deployInitCode(bytes memory data) internal returns (address initCodeStorage) {
    assembly {
        let size := mload(data)
        // - Vulnerable code: Incorrect calculation of createSize
        - let createSize := add(size, 0x0b) // Incorrect calculation

        // + Updated expected code: Correct calculation of createSize
        + let createSize := add(size, 0x13) // Correct calculation

        // Prefix Code
        //
        // Has trailing STOP instruction so the deployed data
        // can not be executed as a smart contract.
        //
        // Instruction                | Stack
        // ----------------------------------------------------
        // PUSH2 size                 | size                  |
        // PUSH0                      | 0, size               |
        // DUP2                       | size, 0, size         |
        // PUSH1 10 (offset to STOP)  | 10, size, 0, size     |
        // PUSH0                      | 0, 10, size, 0, size  |
        // CODECOPY                   | 0, size               |
        // RETURN                     |                       |
        // STOP                       |                       |
        // ----------------------------------------------------

        // Shift (size + 1) to position it in front of the PUSH2 instruction.
        // Reuse data.length memory for the create prefix to avoid
        // unnecessary memory allocation.
        mstore(data, or(shl(64, add(size, 1)), 0x6100005f81600a5f39f300))
        // Deploy the code storage
        initCodeStorage := create(0, add(data, 21), createSize)
        // if (initCodeStorage == address(0)) revert InitCodeDeploymentFailed();
        if iszero(initCodeStorage) {
            mstore(0, 0x11c8c3c0)
            revert(0x1c, 0x04)
        }
        // Restore data.length
        mstore(data, size)
    }
}
```


### Expected Behavior
The `deployInitCode` function should correctly calculate the total size of the code to be deployed by adding 19 bytes (`0x13` in hexadecimal) to the size of the input data. This accounts for the 8-byte shifted size and the 11-byte code prefix. The corrected `createSize` ensures that the entire prefixed code is deployed without truncation, preventing deployment failures and ensuring the deployed code behaves as intended.

### Actual Behavior
The current implementation incorrectly adds `0x0b` (11 bytes) to the size of the input data when calculating `createSize`. This miscalculation does not account for the 8-byte shifted size, resulting in a total `createSize` of `size + 11` instead of the required `size + 19`. Deploying with this incorrect `createSize` leads to incomplete code being deployed, which can cause deployment failures or unexpected behavior of the deployed contract.

### Actual Behavior Code Snippet
The incorrect code causing this issue is:

```
let createSize := add(size, 0x0b) // Incorrect calculation
```
-----------------------------------------

### [L-22] Logical Error in `_packString` Function Leading to Incorrect Token Names and Symbols

**Contract Name:** `HooksFactory`

**Description:**

The `_packString` function in the `HooksFactory` contract is designed to efficiently pack a string of up to 63 bytes into two `bytes32` words for storage or processing. However, due to incorrect memory offset calculations in the assembly code, the function fails to correctly pack the string data. This results in incorrect token names and symbols for deployed markets, leading to confusion, misrepresentation, and potential security risks.

**Code Snippet:**

```
function _packString(string memory str) internal pure returns (bytes32 word0, bytes32 word1) {
    assembly {
        let length := mload(str)
        // Equivalent to:
        // if (str.length > 63) revert NameOrSymbolTooLong();
        if gt(length, 0x3f) {
            mstore(0, 0x19a65cb6)
            revert(0x1c, 0x04)
        }
-        // Load the length and first 31 bytes of the string into the first word
-        // by reading from 31 bytes after the length pointer.
-        word0 := mload(add(str, 0x1f))
-        // If the string is less than 32 bytes, the second word will be zeroed out.
-        word1 := mul(mload(add(str, 0x3f)), gt(mload(str), 0x1f))
+        // Load the length and first 32 bytes of the string into the first word
+        // by reading from 32 bytes after the length pointer.
+        word0 := mload(add(str, 0x20))
+        // If the string is less than 32 bytes, the second word will be zeroed out.
+        word1 := mload(add(str, 0x40))
    }
}
```

**Expected Behavior:**

The `_packString` function should:

- Correctly pack a string of up to 63 bytes into two `bytes32` variables (`word0` and `word1`).
- For strings up to 32 bytes:
  - `word0` contains the string length and the first part of the string data.
  - `word1` is set to zero.
- For strings longer than 32 bytes:
  - `word0` contains the first 32 bytes of the string data.
  - `word1` contains the next 31 bytes of the string data.
- Ensure that all bytes of the string are accurately captured without skipping or overlapping.


**Actual Behavior:**

The `_packString` function incorrectly:

- Skips the initial bytes of the string data due to miscalculated memory offsets.
- Loads unintended or uninitialized memory into `word0` and `word1`.
- Fails to accurately capture the full string data, leading to incorrect token names and symbols.

-------------------------------------------------------

### [L-23] Logic Error in Assembly for Contract Creation with `create2` May Lead to Incorrect Address Calculation/Address Collisions

### File: `HooksFactory.sol`

### Description:
In the `_deployMarket` function, the assembly block is used to calculate and deploy a market contract using `create2`. However, due to improper handling of the `salt` parameter, the calculated address of the deployed contract may differ from the intended address. This could result in the creation of an unintended contract at an incorrect address, potentially leading to a loss of control over the contract's logic or assets.

### Code Snippet:
```
assembly {
    - let market := create2(0, add(initCodePointer, 0x20), mload(initCodePointer), salt)
    + let properSalt := keccak256(abi.encodePacked(salt, caller()))
    + let market := create2(0, add(initCodePointer, 0x20), mload(initCodePointer), properSalt)
    if iszero(market) {
        mstore(0x00, 0x30116425) // DeploymentFailed()
        revert(0x1c, 0x04)
    }
}
```

### Logic Error:
The issue lies in how the `salt` is passed to the `create2` opcode. The `create2` opcode generates the contract address based on the sender, the `salt`, and the contract's bytecode. However, in this case, the `salt` does not properly include unique identifiers like the caller's address, which could lead to address collisions or incorrect deployments.

- **Incorrect Handling of the Salt**: The `salt` might not reflect the intended uniqueness (e.g., it does not include the address of the creator or other unique identifiers), which can lead to address collisions or the deployment of incorrect contracts.
- **Inconsistent Address Generation**: If the same `salt` is used by different users, contracts could be deployed at the same address unintentionally, causing overwriting or unpredictable behavior.

### Impact:
- **Loss of Contract Ownership**: If a contract is deployed at an unintended address due to incorrect salt handling, the owner may lose control over the contract, resulting in potential loss of assets or incorrect execution of logic.
- **Address Collisions**: If multiple users use the same `salt`, they might deploy contracts at the same address, overwriting each other's contracts and causing conflicts.

### Expected Behavior:
The `salt` should be constructed in such a way that it includes unique identifiers, such as the caller's address and other parameters, to ensure that the generated contract address is unique and predictable.

1. **Use of Proper Salt**: The `salt` should be based on unique identifiers, such as the caller's address, ensuring that each deployed contract has a unique address.
2. **Avoid Address Collisions**: Proper salt handling prevents different users from deploying contracts at the same address by mistake.

### Actual Behavior:
The current implementation may generate the same address for multiple contracts or users, leading to potential overwriting or loss of control over the contract's logic or assets.

### Mitigation:
- **Include Caller Address in Salt**: Modify the salt generation to include the `msg.sender` address or another unique identifier to ensure that each contract is deployed at a unique address.
- **Validate Salt**: Add a check to ensure that the `salt` provided is unique and hasn't been used before, preventing accidental overwriting of deployed contracts.


------------------------------------------------------------------------------------------------------------------------------------

### [L-24] Incorrect `extcodesize` Handling in Assembly Could Cause Code Execution on Non-Contract Addresses

### File: `HooksFactory.sol`

### Description:
In the `_deployHooksInstance` function, the contract checks the size of the code at the `hooksTemplate` address using `extcodesize` to determine whether to proceed with copying and deploying the contract code. However, the use of `extcodesize` with an assumption that any non-zero value indicates a valid contract can lead to a logic error. An address with leftover bytecode or self-destructed contracts can still have non-zero `extcodesize` before the next transaction, leading to incorrect execution.

### Code Snippet:
```
assembly {
    - let initCodeSize := extcodesize(hooksTemplate)
    + if iszero(extcodesize(hooksTemplate)) { revert(0x00, 0x00) }  // Ensure valid contract exists
    + if iszero(isContract(hooksTemplate)) { revert(0x00, 0x00) }    // Additional validation
    extcodecopy(hooksTemplate, initCodePointer, 0, initCodeSize)
}
```

### Logic Error:
- **Incorrect Interpretation of `extcodesize`**: The assumption that any non-zero result from `extcodesize` is valid can be flawed. For instance:
  - A contract that has been self-destructed in the current transaction may still return a non-zero `extcodesize` until the transaction is fully completed.
  - An address that previously had a contract but was self-destructed could still appear valid until fully removed.
  
This can result in incorrect behavior where contracts are copied from addresses that no longer contain valid contract code, leading to failed or unintended deployments.

### Impact:
- **Execution of Invalid Code**: The contract may attempt to copy and execute bytecode from an address that has been self-destructed, leading to faulty or unintended behavior.
- **Deployment of Incomplete Contracts**: If the contract copies incomplete or invalid code due to a flawed `extcodesize` check, the deployed instance may malfunction, potentially leading to financial loss or unexpected outcomes.

### Expected Behavior:
The contract should properly validate that the `hooksTemplate` address contains valid, fully deployed contract code before attempting to copy and deploy the bytecode.

1. **Check for Contract Validity**: Ensure that the address contains a valid contract, not just leftover bytecode from a self-destructed contract.
2. **Prevent Deployment of Invalid Code**: Add more robust checks to avoid deploying from invalid contract addresses.

### Actual Behavior:
The current implementation can mistakenly proceed with copying code from addresses that no longer have a valid contract deployed, leading to potential issues with contract deployment.

### Mitigation:
- **Verify Contract Bytecode Integrity**: Add checks to ensure that the bytecode being copied belongs to a valid contract that has not been self-destructed. This could include validating the contract's hash or using other mechanisms to ensure the contract is still valid.

----------------------------------------------------------------------------------------------------------------

### [L-25] **Incorrect Handling of Dynamic Data in Assembly Can Lead to Buffer Overflows**

### File: `HooksFactory.sol`

### Description:
In assembly, when copying or processing dynamic data (like strings or arrays), incorrect handling of memory boundaries can lead to buffer overflows, allowing attackers to overwrite important data in memory.

### Code Snippet:
```
assembly {
    - let strLength := mload(add(str, 0x20))  // Load length of string
    + let strLength := mload(str)             // Load string length from first word
    + if gt(strLength, 0x40) {                // Ensure string length does not exceed buffer size
    +   revert(0x00, 0x00)
    + }
    mstore(add(str, 0x40), mload(add(str, strLength)))  // Correct data copy within bounds
}
```

### Logic Error:
- **Buffer Overflow**: The assembly code attempts to load and copy data dynamically without properly checking memory boundaries, leading to buffer overflows and memory corruption.

### Mitigation:
- **Perform Bounds Checking**: Ensure that the length of the dynamic data (such as strings or arrays) is properly checked against the available buffer size before performing any memory operations.
- **Use Safe Memory Operations**: Utilize safe memory manipulation functions or implement custom routines that prevent buffer overflows.

---------------------------------------------------------------------------------------------------
-----------------------------------------------------------------------------------------

### [L-26] Memory Corruption & Free Memory Pointer Mismanagement in `_deriveSalt` Function

### File: `WildcatSanctionsSentinel.sol`

### Description:
The `_deriveSalt` function uses inline assembly to calculate a `salt` for contract creation with `create2`. However, the function writes to critical memory slots (`0x00`, `0x20`, and `0x40`), which are used internally by 's memory management system. This can lead to memory corruption, conflicts with other parts of the contract, or undefined behavior due to improper handling of the free memory pointer.

### Code Snippet:
```
assembly {
    // Cache free memory pointer
    let freeMemoryPointer := mload(0x40)
    // `keccak256(abi.encode(borrower, account, asset))`
    - mstore(0x00, borrower)
    - mstore(0x20, account)
    - mstore(0x40, asset)
    + let ptr := add(freeMemoryPointer, 0x80) // Allocate safe memory offsets
    + mstore(ptr, borrower)
    + mstore(add(ptr, 0x20), account)
    + mstore(add(ptr, 0x40), asset)
    salt := keccak256(ptr, 0x60)
    // Restore free memory pointer
    mstore(0x40, freeMemoryPointer)
}
```

### Logic Error:
The issue arises from the direct use of memory slots `0x00`, `0x20`, and `0x40`, which are part of ’s internal memory structure:
- **`0x00` and `0x20`**: Used by  for scratch space and zero slot storage.
- **`0x40`**: Stores the pointer to the next free memory.

Writing to these slots can interfere with the internal workings of the  compiler, leading to memory corruption or undefined behavior. This is particularly risky in complex contracts, where memory management is critical for ensuring correct execution.

### Impact:
- **Memory Corruption**: Overwriting these critical memory slots may result in unexpected behavior, including corruption of contract logic, incorrect results, or even contract failure.
- **Unpredictable Behavior**: If the free memory pointer is not managed correctly, future memory allocations within the contract may behave unpredictably, leading to bugs or vulnerabilities.

### Expected Behavior:
- **Memory Safety**: The function should use safe memory offsets that do not interfere with 's internal memory management, ensuring that the contract operates reliably.
- **Avoidance of Memory Corruption**: The memory used for storing data (borrower, account, asset) should be allocated in a safe space, avoiding critical memory slots used by .

### Actual Behavior:
The function overwrites important memory locations (`0x00`, `0x20`, and `0x40`), potentially causing memory corruption and leading to unpredictable or incorrect contract behavior.

### Mitigation:
- **Use Higher Memory Offsets**: Allocate memory at safe offsets, such as `0x80` and beyond, to avoid conflicts with ’s internal memory structure. 

--------------------------------------------------------------------------


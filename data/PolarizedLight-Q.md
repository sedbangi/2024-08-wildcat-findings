
# Team Polarized Light WildCat Protocol QA Report 

## Low 1 - Off-by-One Vulnerability in Fixed Term Loan Withdrawal Timing

### Overview:

The FixedTermLoanHooks contract contains a vulnerability that allows users to withdraw funds exactly at the fixed term end time, rather than strictly after it. This off-by-one error could potentially be exploited to withdraw funds earlier than intended by the protocol design.

### Description:

The FixedTermLoanHooks contract is designed to enforce a fixed-term loan period, after which withdrawals are permitted. However, the current implementation allows withdrawals to occur at the exact moment the fixed term ends, rather than enforcing a strict "after end time" rule.

In the `onQueueWithdrawal` function, the check for withdrawal timing is implemented as follows:

```solidity
function onQueueWithdrawal(
    address lender,
    uint32 /* expiry */,
    uint /* scaledAmount */,
    MarketState calldata /* state */,
    bytes calldata hooksData
  ) external override {
    HookedMarket memory market = _hookedMarkets[msg.sender];
    if (!market.isHooked) revert NotHookedMarket();
    if (market.fixedTermEndTime > block.timestamp) {
      revert WithdrawBeforeTermEnd();
    }
    // ... rest of the function
  }
```

The condition `market.fixedTermEndTime > block.timestamp` allows withdrawals when `block.timestamp` is exactly equal to `market.fixedTermEndTime`. This is inconsistent with the expected behavior of a fixed-term loan, where withdrawals should only be permitted after the term has fully elapsed.

This vulnerability was identified through the following test:

```solidity
 function test_offByOneErrorInFixedTermWithdrawal() external {

        // Setup
        uint32 fixedTermEndTime = uint32(block.timestamp + 30 days);
        address marketAddress = address(1);
        address testAccount = address(this);


        // Set up the market with a fixed term
        hooks.setHookedMarket(
            marketAddress,
            HookedMarket({
                isHooked: true,
                transferRequiresAccess: false,
                depositRequiresAccess: false,
                withdrawalRequiresAccess: true,
                minimumDeposit: 0,
                fixedTermEndTime: fixedTermEndTime
            })
        );

        // Grant withdrawal access to the test account
        hooks.addRoleProvider(address(this), 365 days);
        hooks.grantRole(testAccount, uint32(block.timestamp));

        // Mark the test account as a known lender on the market
        hooks.setIsKnownLender(testAccount, marketAddress, true);

        // Test withdrawals
        MarketState memory state;
        vm.startPrank(marketAddress);

        // 1. Attempt to withdraw 1 second before fixedTermEndTime (should fail)
        vm.warp(fixedTermEndTime - 1);
        vm.expectRevert(FixedTermLoanHooks.WithdrawBeforeTermEnd.selector);
        hooks.onQueueWithdrawal(testAccount, 0, 1, state, "");

        // 2. Attempt to withdraw exactly at fixedTermEndTime (should fail, but doesn't due to the bug)
        vm.warp(fixedTermEndTime);
        bool successAtExactTime = true;
        try hooks.onQueueWithdrawal(testAccount, 0, 1, state, "") {
            // If this succeeds, it confirms the off-by-one error
            successAtExactTime = true;

        } catch {
            // If this fails, the off-by-one error is not present
            successAtExactTime = false;
        }

        assertTrue(successAtExactTime, "Withdrawal at exact end time should fail but succeeded (off-by-one error)");

        // 3. Attempt to withdraw 1 second after fixedTermEndTime (should succeed)
        vm.warp(fixedTermEndTime + 1);
        hooks.onQueueWithdrawal(testAccount, 0, 1, state, "");
        vm.stopPrank();
    }
}
```

The test output shows:

```
Ran 1 test for test/access/FixedTermLoanHooks.t.sol:FixedTermLoanHooksTest
[PASS] test_offByOneErrorInFixedTermWithdrawal() (gas: 102892)
```

This indicates that the `onQueueWithdrawal`  allows a withdrawal at the exact `fixedTermEndTime`.

### Code Location:

FixedTermLoanHooks.sol, `onQueueWithdrawal` function

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/FixedTermLoanHooks.sol#L857


### Impact:

This vulnerability allows users to withdraw funds slightly earlier than intended by the protocol design. While the time difference is minimal (1 second), it represents a deviation from the expected behavior of fixed-term loans and could potentially be exploited in combination with other timing-sensitive operations.

### Recommended Mitigation:

Modify the condition in the `onQueueWithdrawal` function to use a greater-than-or-equal-to comparison:

```solidity
if (market.fixedTermEndTime >= block.timestamp) {
  revert WithdrawBeforeTermEnd();
}
```

This change ensures that withdrawals are only permitted strictly after the fixed term end time, aligning with the expected behavior of fixed-term loans.

This off-by-one error in the FixedTermLoanHooks contract represents a subtle but important deviation from the intended behavior of fixed-term loans. By allowing withdrawals at the exact end time rather than strictly after it, the contract fails to fully enforce the fixed-term nature of the loans. 

## Low 2 - WETH9 Transfer Failure Risk in HooksFactory on Blast Chain

### Overview:

The HooksFactory contract in the Wildcat protocol may encounter transfer failures when using WETH9 as the origination fee asset on the Blast chain due to WETH9's non-standard implementation on this network.

### Description:

The `_deployMarket` function in HooksFactory attempts to transfer the origination fee using `safeTransferFrom`:

```solidity
if (originationFeeAsset != address(0)) {
  originationFeeAsset.safeTransferFrom(
    msg.sender,
    templateDetails.feeRecipient,
    originationFeeAmount
  );
}
```

On the Blast chain, WETH9 has a non-standard implementation that requires explicit approval even when `msg.sender` is the token owner. This could cause the transfer to fail if WETH9 is used as the `originationFeeAsset` on Blast.

### CodeLocation:

HooksFactory.sol#L393-L419
https://github.com/code-423n4/2024-08-wildcat/blob/main/src/HooksFactory.sol#L393-L419

### Impact:

When deploying markets on the Blast chain with WETH9 as the origination fee asset:
1. Market deployment could fail due to unsuccessful fee transfers.
2. This could prevent the protocol from functioning properly on the Blast chain when using WETH9.

The impact is specifically limited to the use of WETH9 on the Blast chain during the market deployment process.

### Recommended mitigations:

1. Implement a Blast-specific check and approval step for WETH9:

```solidity
if (originationFeeAsset == address(WETH9) && block.chainid == BLAST_CHAIN_ID) {
  IERC20(originationFeeAsset).approve(address(this), originationFeeAmount);
}
originationFeeAsset.safeTransferFrom(
  msg.sender,
  templateDetails.feeRecipient,
  originationFeeAmount
);
```

2. Create a wrapper contract for WETH9 on Blast:
Develop a Blast-specific WETH9 wrapper that handles the approval process internally, normalizing its behavior to match standard ERC20 tokens.

Implementing the suggested mitigations would enhance the robustness of the origination fee process on Blast, maintaining the protocol's functionality when using WETH9 on this network.

## Low 3 - Permissive Expiry Check in Credential Validation

### Overview:

The `FixedTermLoanHooks` contract allows credentials to be considered valid when their expiry time is exactly equal to the current block timestamp, potentially leading to unexpected behavior in time-sensitive operations.

### Description: 

In the `FixedTermLoanHooks` contract, the credential expiry check is implemented using a less-than comparison:

```solidity
if (newExpiry < block.timestamp) revert GrantedCredentialExpired();
```

This condition allows a credential to be considered valid when its expiry time is exactly equal to the current block timestamp. While this approach is more permissive, it could lead to unexpected behavior in scenarios where strict time bounds are crucial.

### CodeLocation: 

File: `FixedTermLoanHooks.sol
Function: `_grantRole
Line: [https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/FixedTermLoanHooks.sol#L451-L452]

### Impact: 

This permissive check could potentially allow actions to be performed with credentials that are technically expired. In highly time-sensitive operations or in systems where precise timing is critical, it could lead to unexpected behavior or security risks.

### Recommended mitigations:

1. If strict timing is required, consider changing the condition to:
   ```solidity
   if (newExpiry <= block.timestamp) revert GrantedCredentialExpired();
   ```
   This will ensure that credentials are considered expired exactly at their expiry time.

2. Consider adding a small buffer time (e.g., a few seconds) to the expiry check to account for potential block time variations:
   ```solidity
   if (newExpiry < block.timestamp + EXPIRY_BUFFER) revert GrantedCredentialExpired();
   ```

Addressing this finding will enhance the contract's clarity and security especially in time-critical applications. It's crucial to align the expiry check mechanism with the system's specific requirements.

## Low 4 - Zero-Value Token Transfers Not Handled

### Overview:

The `WildcatMarket` contract and the associated `WildcatSanctionsEscrow` contract do not check for zero-value token transfers before executing them. This oversight can lead to unnecessary gas consumption and potential transaction reverts with certain ERC20 token implementations.

### Description:

The contracts perform token transfers without verifying whether the transfer amount exceeds zero. While many ERC20 tokens allow zero-value transfers, some implementations may revert on such attempts. Additionally, executing zero-value transfers wastes gas and can occur unintentionally when dealing with fractional amounts that round down to zero.

This issue is present in at least two key functions:

1. The `borrow` function in WildcatMarket, which transfers tokens to the borrower.
2. The `releaseEscrow` function in WildcatSanctionsEscrow, which releases escrowed tokens.

Both functions use the `safeTransfer` method from the LibERC20 library but do not include a check for zero amounts before calling it.

### CodeLocation:

`WildcatMarket` contract:
```solidity
function borrow(uint256 amount) external onlyBorrower nonReentrant sphereXGuardExternal {
  // ... (other checks)
  asset.safeTransfer(msg.sender, amount);
}
```

`WildcatSanctionsEscrow` contract:
```solidity
function releaseEscrow() public override {
  // ... (other checks)
  uint256 amount = balance();
  asset.safeTransfer(_account, amount);
}
```

### Impact:

1. Potential transaction reverts: Some ERC20 tokens may revert on zero-value transfers, causing the entire transaction to fail.
2. Gas inefficiency: Attempting zero-value transfers unnecessarily consumes gas, increasing transaction costs.
3. Unintended behavior: Fractional amounts rounding down to zero could lead to no actual transfer occurring, potentially confusing users or disrupting expected contract behavior.

### Recommended mitigations:

1. Add a zero-value check before executing transfers:
```solidity
if (amount > 0) {
  asset.safeTransfer(recipient, amount);
}
```

2. For the `borrow` function, consider reverting if a zero amount is requested:
```solidity
if (amount == 0) revert ZeroBorrowAmount();
```

3. Implement a minimum transfer threshold to prevent issues with fractional amounts rounding down to zero.

4. For the `releaseEscrow` function, you might want to skip the transfer entirely if the balance is zero:
```solidity
uint256 amount = balance();
if (amount > 0) {
  asset.safeTransfer(_account, amount);
}
```

## Low 5 - Unbounded Growth of Hooks Templates Array

### Overview: 

The HooksFactory contract contains an array (`_hooksTemplates`) that can grow indefinitely without a mechanism to remove elements, potentially leading to gas inefficiencies and operational issues over time.

### Description:

The HooksFactory contract maintains an array `_hooksTemplates` to keep track of all hooks templates. New templates are added to this array in the `addHooksTemplate` function, but there is no corresponding functionality to remove templates from the array. While templates can be disabled via the `disableHooksTemplate` function, this only sets a flag in a separate mapping and does not remove the template from the array.

This design can lead to an ever-growing array, which could cause the following issues:

1. Increased gas costs for functions that iterate over or return the entire array.
2. Potential block gas limit issues if the array grows too large.
3. Inconsistencies between the array contents and the actual state of enabled templates.
4. Difficulty in managing or upgrading the system if it becomes necessary to remove obsolete templates.

### Code Location:

https://github.com/example/hooksFactory/blob/main/HooksFactory.sol

Relevant code snippets:
```solidity
address[] internal _hooksTemplates;

function addHooksTemplate(...) external override onlyArchControllerOwner {
    // ...
    _hooksTemplates.push(hooksTemplate);
    // ...
}

function disableHooksTemplate(address hooksTemplate) external override onlyArchControllerOwner {
    // ...
    _templateDetails[hooksTemplate].enabled = false;
    // Note: Does not remove from _hooksTemplates array
    // ...
}
```

### Impact:

While not immediately critical, this issue can lead to gradually increasing gas costs and potential system-wide inefficiencies. It may also complicate future upgrades or management of the system.

### Recommended Mitigations:

1. Implement a function to remove disabled templates from the `_hooksTemplates` array. This could be combined with the existing `disableHooksTemplate` function.

2. Consider using a different data structure, such as a mapping with a separate array for keys, which would allow for easier removal of elements.

Example mitigation (option 1):

```solidity
function removeHooksTemplate(address hooksTemplate) external onlyArchControllerOwner {
    uint256 index = _templateDetails[hooksTemplate].index;
    require(index < _hooksTemplates.length, "Template not found");
    
    // Move the last element to the position of the removed element
    _hooksTemplates[index] = _hooksTemplates[_hooksTemplates.length - 1];
    _templateDetails[_hooksTemplates[index]].index = index;
    
    // Remove the last element
    _hooksTemplates.pop();
    
    // Clear the template details
    delete _templateDetails[hooksTemplate];
    
    emit HooksTemplateRemoved(hooksTemplate);
}
```
By implementing a mechanism to remove the `_hooksTemplates` array, the contract will be better equipped to handle a growing number of templates without compromising on gas efficiency or system manageability. 

## Low 6 - Potential Failure of Large Transfers with Non-Standard ERC20 Tokens

### Overview:

The `WildcatMarket` and related contracts assume standard ERC20 token behavior for all operations. However, some ERC20 tokens may have built-in transfer restrictions or behave unexpectedly with large amounts, potentially causing operations to fail.

### Description:

The contracts use `safeTransfer` and `safeTransferFrom` for token transfers, which is a good practice. However, they don't account for potential limitations in the underlying token contracts, such as maximum transfer sizes or daily limits. This oversight could lead to failed operations when dealing with large amounts or non-standard ERC20 tokens.

Key vulnerable operations include:

1. Withdrawals
2. Borrowing
3. Repayments
4. Fee collection
5. Market closure

These operations assume that any amount of tokens can be transferred in a single transaction, which may not be true for all ERC20 implementations.

### CodeLocation:

1. WildcatMarketWithdrawals.sol:

```solidity
function _executeWithdrawal(
    MarketState memory state,
    address accountAddress,
    uint32 expiry,
    uint baseCalldataSize
  ) internal returns (uint256) {
    // ... (other code)
    if (_isSanctioned(accountAddress)) {
      address escrow = _createEscrowForUnderlyingAsset(accountAddress);
      asset.safeTransfer(escrow, normalizedAmountWithdrawn);
    } else {
      asset.safeTransfer(accountAddress, normalizedAmountWithdrawn);
    }
    // ... (other code)
  }
```

2. WildcatMarket.sol:

```solidity
function collectFees() external nonReentrant sphereXGuardExternal {
    // ... (other code)
    asset.safeTransfer(feeRecipient, withdrawableFees);
    // ... (other code)
  }

function borrow(uint256 amount) external onlyBorrower nonReentrant sphereXGuardExternal {
    // ... (other code)
    asset.safeTransfer(msg.sender, amount);
    // ... (other code)
  }

function _repay(MarketState memory state, uint256 amount, uint256 baseCalldataSize) internal {
    // ... (other code)
    asset.safeTransferFrom(msg.sender, address(this), amount);
    // ... (other code)
  }

function closeMarket() external onlyBorrower nonReentrant sphereXGuardExternal {
    // ... (other code)
    if (currentlyHeld < totalDebts) {
      uint256 remainingDebt = totalDebts - currentlyHeld;
      _repay(state, remainingDebt, 0x04);
    } else if (currentlyHeld > totalDebts) {
      uint256 excessDebt = currentlyHeld - totalDebts;
      asset.safeTransfer(borrower, excessDebt);
    }
    // ... (other code)
  }
```

3. WildcatSanctionsEscrow.sol:

```solidity
function releaseEscrow() public override {
    // ... (other code)
    asset.safeTransfer(_account, amount);
    // ... (other code)
  }
```

### Impact:

The impact of this issue could be:

1. Failed withdrawals could lock user funds in the contract.
2. Failed borrowing operations could prevent the borrower from accessing funds.
3. Failed repayments could incorrectly maintain debt levels.
4. Failed fee collections could result in loss of protocol revenue.
5. Failed market closure could leave the market in an inconsistent state.

These issues could lead to loss of funds and reduced protocol functionality.

### Recommended mitigations:

1. Implement a token compatibility check during market creation to ensure the underlying asset doesn't have unexpected transfer restrictions.
2. Add a mechanism to split large transfers into smaller chunks:

```solidity
function safeTransferLarge(address token, address to, uint256 amount) internal {
    uint256 balance = IERC20(token).balanceOf(address(this));
    require(balance >= amount, "Insufficient balance");

    uint256 batchSize = 1000000 * 10**IERC20(token).decimals(); // Example batch size
    uint256 remaining = amount;

    while (remaining > 0) {
        uint256 batch = remaining > batchSize ? batchSize : remaining;
        IERC20(token).safeTransfer(to, batch);
        remaining -= batch;
    }
}
```

3. Implement more robust error handling for transfer failures, possibly with retry mechanisms:

```solidity
function safeTransferWithRetry(address token, address to, uint256 amount) internal {
    uint8 attempts = 0;
    bool success = false;
    while (!success && attempts < 3) {
        try IERC20(token).safeTransfer(to, amount) {
            success = true;
        } catch {
            attempts++;
        }
    }
    require(success, "Transfer failed after multiple attempts");
}
```

While the current implementation uses safe transfer methods, it doesn't account for the full range of ERC20 tokens. The suggested mitigations would significantly enhance the reliability of the protocol.

## Low 7 - Enhancing Reentrancy Protection in Withdrawal Process

## Overview

The WildcatMarketWithdrawals and WildcatSanctionsEscrow contracts have opportunities for improving their reentrancy protection mechanisms in the withdrawal and escrow release processes. While some protective measures are in place, certain areas could benefit from additional safeguards to further strengthen the system's security.

## Description

The main area for improvement centers on the execution of withdrawals and the release of funds from escrow. The primary `executeWithdrawal` function is protected by a reentrancy guard, but the internal `_executeWithdrawal` function and the `releaseEscrow` function in the escrow contract could benefit from explicit reentrancy protection. Additionally, optimizing the order of operations in `_executeWithdrawal` could further enhance security, particularly regarding hook functions.

## Code Location

1. WildcatMarketWithdrawals contract:
   - `executeWithdrawal` function (protected by `nonReentrant` modifier)
   - `_executeWithdrawal` function (internal function called by `executeWithdrawal`)
2. WildcatSanctionsEscrow contract:
   - `releaseEscrow` function (lines 34-43)

## Impact

Implementing these improvements could prevent:
- Potential unauthorized multiple withdrawals
- Unintended manipulation of contract state
- Possible drain of funds
- Disruption of the withdrawal process

Enhancing security measures is particularly important given the financial nature of the contracts.

## Recommended Mitigations

1. Implement a reentrancy guard in the `_executeWithdrawal` function.
2. Reorder operations in `_executeWithdrawal` to ensure all state changes occur before external calls, including hook functions.
3. Add a reentrancy guard to the `releaseEscrow` function in the WildcatSanctionsEscrow contract.
4. Consider implementing a reentrancy guard at the contract level for WildcatSanctionsEscrow.

Implementing these improvements will significantly enhance the robustness and security of the protocol, ensuring better protection for both the system and its users.

## Low 8 - Unbounded Solidity Version Pragma

### Overview:

The contract uses a floating pragma statement `pragma solidity >=0.8.20;` which does not specify an upper bound for the Solidity compiler version. This practice can lead to potential vulnerabilities if the contract is compiled with a future version of Solidity that introduces breaking changes or new features that may affect the contract's behavior.

### Description:

New versions of solidity may introduce changes that could alter the behavior of existing code. By using a floating pragma without an upper bound, the contract becomes vulnerable to potential issues arising from future compiler versions. This can lead to unexpected behavior, security vulnerabilities, or deployment problems if the contract is recompiled with a newer, incompatible version of Solidity.

### CodeLocation:

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketToken.sol#L2-L2

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/libraries/LibStoredInitCode.sol#L2-L2

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/TransientBytesArray.sol#L2-L2

### Impact:

The impact of this vulnerability is potentially severe. It may lead to:
1. Unexpected contract behavior if compiled with a future, incompatible Solidity version.
2. Introduction of new security vulnerabilities due to changes in language semantics.
3. Deployment issues or increased gas costs if the contract is recompiled with a significantly different compiler version.
4. Difficulties in auditing and maintaining the contract across different environments or over time.

### Recommended mitigations:

1. Specify both a lower and upper bound for the Solidity version:

   ```solidity
   pragma solidity >=0.8.20 <0.9.0;
   ```

2. Alternatively, lock the contract to a specific Solidity version:

   ```solidity
   pragma solidity 0.8.20;
   ```

By implementing a bounded Solidity version pragma the protocol can significantly reduce the risk of unexpected behavior and potential vulnerabilities introduced by future compiler versions.

## Low 9 - Reduced Compatibility Due to PUSH0 Opcode Usage in Solidity 0.8.20

### Overview:

The contract uses Solidity version 0.8.20, which introduces the PUSH0 opcode from the Shanghai EVM upgrade. This may limit the contract's compatibility with certain chains or Layer 2 solutions that haven't implemented this upgrade.

### Description:

Solidity 0.8.20 utilizes the PUSH0 opcode, which was introduced in the Ethereum Shanghai upgrade. While this opcode is supported on the Ethereum mainnet and many testnets, it may not be implemented on all EVM-compatible chains or Layer 2 solutions. This could lead to deployment failures or unexpected behavior on networks that haven't adopted the Shanghai upgrade.

The use of this newer Solidity version, while beneficial for its latest features and optimizations, potentially reduces the portability of the smart contract across different blockchain environments.

### Code Location:

```solidity
// SPDX-License-Identifier: Apache-2.0
pragma solidity >=0.8.20;
```

### Impact:

- Reduced compatibility with older EVM-compatible chains and some Layer 2 solutions
- Potential deployment failures on unsupported networks
- Possible unexpected behavior if deployed on networks without PUSH0 support
- Limited portability of the smart contract across diverse blockchain environments

### Recommended Mitigations:

1. Consider using an earlier Solidity version (e.g., 0.8.19) that doesn't rely on the PUSH0 opcode for broader compatibility.
2. If specific features from 0.8.20 are required, carefully evaluate the target deployment environments to ensure they support the Shanghai upgrade.
3. Implement a multi-version contract strategy, maintaining separate versions for networks with and without PUSH0 support.
4. Clearly document the required EVM version and potential compatibility issues in the project documentation.

The choice of Solidity version 0.8.20 introduces a trade-off between leveraging the latest language features and maintaining broad compatibility. 

## Low 10 - Unbounded Mapping Array `_marketsByHooksTemplate`

### Overview

The contract uses an unbounded mapping array `_marketsByHooksTemplate` which is iterated upon in certain functions. While mitigations are in place, this design could potentially lead to gas inefficiency or Denial of Service scenarios if the number of markets grows excessively large.

## Description

The `_marketsByHooksTemplate` mapping in the contract is an unbounded array that stores markets for each hooks template. This array is appended to in the `_deployMarket` function:

```solidity
_marketsByHooksTemplate[hooksTemplate].push(market);
```

Functions like `getMarketsForHooksTemplate` and `pushProtocolFeeBipsUpdates` iterate over this array. If the number of markets for a specific hooks template becomes very large, these iterations could consume excessive gas or potentially hit block gas limits, rendering the functions unusable.

Mitigating factors include:
1. Pagination in `getMarketsForHooksTemplate`
2. Bounded iterations in `pushProtocolFeeBipsUpdates`

These design choices indicate awareness of potential issues with large arrays and attempts to mitigate them. However, the underlying risk of an unbounded array remains.

## Code Location

- `_marketsByHooksTemplate` mapping
- `_deployMarket` function
- `getMarketsForHooksTemplate` function
- `pushProtocolFeeBipsUpdates` function

## Impact

The impact of this issue is currently low due to implemented mitigations. However, as the number of markets grows, potential impacts include:
1. Increased gas costs for operations involving these functions
2. Possible DoS scenarios if array sizes exceed block gas limits
3. Degraded user experience due to higher transaction costs or failed transactions

## Recommended Mitigations

While current mitigations reduce severity, consider implementing the following to further optimize gas usage and prevent future issues:

1. Implement a maximum limit on the number of markets per hooks template.
2. Use a more gas-efficient data structure for storing markets, such as a mapping with a separate index-tracking array.
3. Develop a mechanism to remove markets from the array when they're no longer needed.
4. Implement additional checks to prevent operations on excessively large arrays.

While the current implementation includes mitigations for large arrays, the use of an unbounded mapping array presents a potential risk as the protocol scales. 

## Non-Critical Findings

## NC 1 - Arithmetic in Array Index Poses Potential Risk in Protocol Fee Update Function

### Overview:

The `pushProtocolFeeBipsUpdates` function in the HooksFactory contract performs arithmetic operations directly within an array index. This practice can lead to readability issues, potential bugs in more complex scenarios, and difficulties in debugging.

### Description:

In the `pushProtocolFeeBipsUpdates` function, the contract accesses array elements using arithmetic operations directly in the index:

```solidity
address market = markets[marketStartIndex + i];
```

This practice is generally discouraged in smart contract development. It can make the code harder to read and maintain, and in more complex scenarios, could potentially lead to index out of bounds errors or unexpected behavior if the arithmetic operation results in an unintended value.

### Code Location:

HooksFactory contract, within the `pushProtocolFeeBipsUpdates` function:

```solidity
function pushProtocolFeeBipsUpdates(
    address hooksTemplate,
    uint marketStartIndex,
    uint marketEndIndex
  ) public override nonReentrant {
    // ... (earlier code omitted for brevity)
    for (uint256 i = 0; i < count; i++) {
      address market = markets[marketStartIndex + i];
      // ... (rest of the function)
    }
  }
```

### Impact:

This practice could lead to more significant issues if the surrounding code is modified without careful consideration, or if similar patterns are adopted in more complex parts of the codebase.

### Recommended Mitigations:
1. Separate the index calculation from the array access:

```solidity
uint256 marketIndex = marketStartIndex + i;
address market = markets[marketIndex];
```

2. Consider using SafeMath or Solidity 0.8.x's built-in overflow checking for the index calculation in more complex scenarios.

3. Add explicit bounds checking before accessing the array, if not already present elsewhere in the function.

Implementing the suggested changes would improve code readability, maintainability, and set a positive precedent for handling array accesses safely. 

## NC 2 - Potential Gas Optimization and Scalability Issue in Role Provider Iteration

### Overview:

The `_loopTryGetCredential` function in the FixedTermLoanHooks contract iterates over an unbounded `_pullProviders` array, which could lead to excessive gas consumption and scalability issues if the number of role providers grows significantly.

### Description:

The `_loopTryGetCredential` function iterates over all elements in the `_pullProviders` array, excluding a specified index. This array is managed by `addRoleProvider` and `removeRoleProvider` functions without an explicit size limit. While not immediately problematic for small arrays, if the number of pull providers grows substantially over time, it could lead to increased gas costs and potential transaction failures due to block gas limits.

### Key points:

1. The function is internal and called within `_tryValidateAccessInner`, which is used in various hooks like `onDeposit`, `onQueueWithdrawal`, and `onTransfer`.
2. Each iteration calls `_tryGetCredential`, potentially consuming significant gas.

### CodeLocation:

```solidity
function _loopTryGetCredential(
    LenderStatus memory status,
    address accountAddress,
    uint256 pullProviderIndexToSkip
  ) internal view returns (bool foundCredential) {
    uint256 providerCount = _pullProviders.length;
    for (uint256 i = 0; i < providerCount; i++) {
      if (i == pullProviderIndexToSkip) continue;
      RoleProvider provider = _pullProviders[i];
      if (_tryGetCredential(status, provider, accountAddress)) return (true);
    }
  }
```
### Impact:

The impact is the following:

- For a small number of providers, the impact is minimal.
- As the number of providers increases, gas costs for operations involving credential checks will rise.
- In scenarios with a very large number of providers, some operations could become expensive or impossible to execute within a single block's gas limit.

### Recommended mitigations:

1. Implement a maximum limit on the number of pull providers that can be added to the system.
2. Consider a more gas-efficient data structure for storing and querying pull providers, such as a mapping with a separate array for keys.

The unbounded nature of the `_pullProviders` array, combined with the iteration in `_loopTryGetCredential`, could become a significant bottleneck as the system scales. Implementing appropriate mitigations now would ensure long-term efficiency and gas-friendliness, providing a better user experience and maintaining the protocol's viability as it grows. 









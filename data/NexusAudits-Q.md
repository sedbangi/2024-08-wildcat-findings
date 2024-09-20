# QA Report: Wildcat Protocol

## L-01: Memory Safety Violations

### 1. Memory Safety Violation in calculateCreate2Address (LibStoredInitCode)

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/libraries/LibStoredInitCode.sol#L55

The LibStoredInitCode contract uses inline assembly for memory manipulation, particularly in the deployInitCode, calculateCreate2Address, and various create2WithStoredInitCode functions. While it generally adheres to Solidity's memory model, there's a potential concern in calculateCreate2Address:

The calculateCreate2Address function uses the memory area from 0x00 to 0x55 as scratch space to prepare data for the keccak256 hash calculation. While this is technically within the allowed 64-byte scratch space, it's close to the boundary and could potentially lead to issues if other parts of the code inadvertently use the same memory region.

### Recommendation

Explicit Memory Allocation in calculateCreate2Address: Allocate memory using mload(0x40) and update the free memory pointer for the scratch space used in this function to enhance clarity and avoid potential conflicts.

### 2. Memory Safety Violation in symbol and name functions (WildcatMarketBase)

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketBase.sol#L77

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketBase.sol#L97

These functions use inline assembly to construct strings from bytes32 values stored in immutable storage. They directly write to memory locations 0x00 to 0x80, which overlaps with the scratch space area (0x00 to 0x40).

If other parts of the contract use the scratch space concurrently with calls to symbol or name, it could lead to unexpected behavior or data corruption.

### Recommendation

Use explicit memory allocation using mload(0x40) and update the free memory pointer before writing the string data to memory.

### 3. Memory Safety Violation in _isSanctioned function (WildcatMarketBase)

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketBase.sol#L254

This function uses inline assembly to call isSanctioned on the sentinel contract. It stores the returned data at memory location 0.

Writing to memory location 0 can potentially conflict with other code using the scratch space.

### Recommendation

Allocate memory explicitly for storing the return data from isSanctioned.

### 4. Memory Safety Violation in _createEscrowForUnderlyingAsset function (WildcatMarketBase)

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketBase.sol#L769

This function uses inline assembly to call createEscrow on the sentinel contract and stores the returned escrow address at memory location 0.

Writing to memory location 0 could lead to conflicts with other code using the scratch space.

### Recommendation

Explicitly allocate memory for storing the returned escrow address.

### 5. Memory Safety Violation in version function (WildcatMarketBase)

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarketBase.sol#L34

This function uses inline assembly to return a hardcoded version string ("2"). It writes directly to memory locations 0x20, 0x40, and 0x41.

Writing to memory location 0x40 could overwrite the free memory pointer, disrupting Solidity's memory management if not handled carefully.

### Recommendation

It's generally safer to allocate memory explicitly using mload(0x40) and update the free memory pointer before writing data. However, given the very specific and limited nature of this function, the risk is likely minimal.


### 6. Memory Safety Violation in _updateSphereXEngineOnRegisteredContractsInSet function (WildcatArchController)

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatArchController.sol#L116

This function uses inline assembly to handle errors when an account is not in a set. It directly writes an error selector to memory location 0 and then reverts.

Writing to memory location 0 can conflict with other code using the scratch space, leading to unexpected behavior.

### Recommendation

Use a local variable within the assembly block to store the error selector before reverting.

### 7. Memory Safety Violation in _tryGetCredential Function

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/AccessControlHooks.sol#L470

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/FixedTermLoanHooks.sol#L507

This function uses inline assembly to call the getCredential function on role provider contract. It stores the returned credentialTimestamp at memory location 0. Writing to memory location 0 might overwrite the scratch space, leading to unexpected behavior if other code uses the scratch space concurrently.

### Recommendation

Allocate memory explicitly using mload(0x40) and update the free memory pointer to store the credentialTimestamp.

### 8. Memory Safety Violation in _tryValidateCredential Function

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/FixedTermLoanHooks.sol#L562

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/AccessControlHooks.sol#L525

This function employs inline assembly to call validateCredential on a role provider. It prepares the calldata by writing to memory starting at calldataPointer.

The code assumes that the hooksData contains at least 20 bytes (for the provider address). If this assumption is violated, _readAddress(hooksData) could lead to out-of-bounds reads from calldata.

The calldatacopy operation copies dataLength bytes from the calldata into the memory buffer starting at add(calldataPointer, 0x80). There's no explicit check to ensure that this copy operation won't overwrite memory beyond the allocated bounds.

### Recommendation

Add a check to ensure that hooksData.length is at least 20 bytes before calling _readAddress(hooksData).

Add a check to make sure that add(calldataPointer, 0x80) + dataLength doesn't exceed the safe memory bounds before the calldatacopy operation.

## L-02: Potential Calldata Corruption in LibHooksConfig

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/HooksConfig.sol

There is an issue in the way LibHooksConfig handles "extra calldata" when preparing calldata for hook functions. While it aims to append extra calldata from the original function call to the hook calldata, there's a potential for this extra data to overwrite critical information in the hook calldata if its size is larger than expected.

The calldatacopy instruction used in functions like onDeposit, onQueueWithdrawal, etc., blindly copies extraCalldataBytes from the end of the original calldata to the memory allocated for the hook calldata.

There's no check to ensure that the copied extraCalldataBytes fit within the allocated memory space. If the extra calldata is larger than expected, it can overwrite the preceding data in the hook calldata, leading to corruption.

### Attack Scenario

An attacker could craft a malicious transaction with excessive extra calldata, causing it to overwrite crucial parameters like the MarketState or other data within the hook calldata. This can lead to the hook contract receiving and acting upon incorrect information, potentially allowing the attacker to manipulate the protocol's state or behavior.

### Code Snippet

```solidit
function onDeposit(
    // ... other parameters
) internal {
    // ...

    assembly {
        // ...

        // Vulnerable `calldatacopy`
        calldatacopy(
            add(headPointer, DepositHook_ExtraData_TailOffset), 
            DepositCalldataSize,
            extraCalldataBytes
        )

        // ...
    }
}
```

### Impact

The hook contract may receive corrupted data, leading to unpredictable and potentially malicious actions.


## L-03: Centralization Risk in Asset Blacklisting

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatArchController.sol#L200

The `WildcatArchController` contract includes functionality to blacklist assets, which is controlled solely by the contract owner. The contract allows the owner to add and remove assets from a blacklist without any checks or balances.

### Code Snippet
```solidity
function addBlacklist(address asset) external onlyOwner {
  if (!_assetBlacklist.add(asset)) {
    revert AssetAlreadyBlacklisted();
  }
  emit AssetBlacklisted(asset);
}

function removeBlacklist(address asset) external onlyOwner {
  if (!_assetBlacklist.remove(asset)) {
    revert AssetNotBlacklisted();
  }
  emit AssetPermitted(asset);
}
```

### Impact
1. Single point of failure: If the owner's private key is compromised, an attacker could arbitrarily blacklist or unblacklist assets.
2. Centralization: This gives significant power to a single entity, which may go against the principles of decentralization.
3. Potential for market manipulation: The owner could theoretically blacklist competing assets or unblacklist favored assets.

### Scenario
1. The owner (or an attacker who has gained control of the owner's private key) decides to manipulate the market.
2. They blacklist a popular asset, causing disruption in the protocol.
3. Users lose trust in the protocol due to this centralized control.

### Potential Fix
Implement a time-lock and/or multi-signature mechanism for blacklisting actions:

```solidity
struct BlacklistAction {
    address asset;
    bool isBlacklisting;
    uint256 effectiveTime;
}

mapping(address => BlacklistAction) public pendingBlacklistActions;
uint256 public constant TIMELOCK_PERIOD = 2 days;

function proposeBlacklistAction(address asset, bool isBlacklisting) external onlyOwner {
    pendingBlacklistActions[asset] = BlacklistAction(asset, isBlacklisting, block.timestamp + TIMELOCK_PERIOD);
    emit BlacklistActionProposed(asset, isBlacklisting);
}

function executeBlacklistAction(address asset) external {
    BlacklistAction memory action = pendingBlacklistActions[asset];
    require(action.effectiveTime != 0 && block.timestamp >= action.effectiveTime, "Action not ready");
   
    if (action.isBlacklisting) {
        _addBlacklist(asset);
    } else {
        _removeBlacklist(asset);
    }
   
    delete pendingBlacklistActions[asset];
}
```


## L-04: Re-entrancy Risk in HooksFactory._deployMarket

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/HooksFactory.sol#L393

The _deployMarket function in HooksFactory makes an external call to IHooks.onCreateMarket before transferring origination fees and deploying the market contract itself.
This sequence creates a potential re-entrancy vulnerability. A malicious hook contract could re-enter the _deployMarket function, potentially manipulating the market deployment process or the transfer of origination fees.

### Recommendation

Consider restructuring _deployMarket to transfer origination fees and deploy the market contract before calling the onCreateMarket hook.
Alternatively, explore using a re-entrancy guard or other mechanisms to prevent re-entrancy during the vulnerable window.


## L-05: Unbounded Iteration in AccessControlHooks._loopTryGetCredential

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/AccessControlHooks.sol#L588

The _loopTryGetCredential function iterates over all pull providers to find a valid credential for a lender.
If the number of pull providers becomes very large, this loop could consume excessive gas, potentially leading to transaction failures or denial-of-service attacks.

### Recommendation

Introduce a maximum iteration limit or gas cost threshold within _loopTryGetCredential to prevent excessive gas consumption in scenarios with many pull providers.s

## L-06: Lack of Input Validation in AccessControlHooks.grantRole

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/FixedTermLoanHooks.sol#L413

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/AccessControlHooks.sol#L376

The grantRole function, which allows role providers to grant access credentials, doesn't explicitly validate the roleGrantedTimestamp provided. A malicious or erroneous role provider could supply an invalid timestamp, potentially leading to unexpected access control behavior.


### Recommendation

Add explicit checks in grantRole to ensure that the roleGrantedTimestamp is within reasonable bounds (e.g., not in the future).


## L-07: Insufficient Handling of Dust Accumulation

The devs highlighted concerns about rounding errors in various functions dealing with normalized and scaled amounts.

### Fix

Implement mechanisms to handle dust accumulation. One approach could involve a periodic "dust sweep" function callable by the borrower or a designated address.

## L-08: Missing ERC20 Validation in rescueTokens()

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarket.sol#L37

The rescueTokens function is designed to allow the borrower to recover tokens accidentally sent to the market contract. However, the function lacks validation to confirm that the provided token address is a valid ERC20 contract. This omission creates a risk as a borrower could call the function with an arbitrary address.

### Vulnerable Code:

```solidity
function rescueTokens(address token) external onlyBorrower {
    if ((token == asset).or(token == address(this))) {
        // ...
    }
}
```

If the provided address doesn't represent an ERC20 contract, it could interact with the contract's logic in unexpected ways, potentially causing errors or unexpected behavior. By adding a check to ensure that the token address corresponds to a contract, we mitigate this risk.

### Proposed Fix

This verification can be done by checking if the code exists at the provided address using token.code.length > 0. This ensures that the address interacts with the intended ERC20 contract, improving the function's security.

```solidity
function rescueTokens(address token) external onlyBorrower {
    // Ensure 'token' is a contract address
    require(token.code.length > 0, "Invalid token address");
    if ((token == asset).or(token == address(this))) {
        // ...
    }
}
```

## L-09: Unauthorized Escrow Release Trigger

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatSanctionsEscrow.sol#L34

### Issue
The WildcatSanctionsEscrow system is designed to isolate assets of sanctioned lenders. The release of these assets should ideally occur only when the lender initiates it, after they are no longer sanctioned or when the borrower explicitly overrides the sanction.

The `releaseEscrow` function in the `WildcatSanctionsEscrow` contract can be called by any address, not just the account owner. While the funds are always sent to the correct account, this allows unauthorized parties to trigger the release.

### Code Snippet
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

### Impact
While funds cannot be redirected, an unauthorized party could trigger the release at an inopportune time for the account owner. This could potentially interfere with the account owner's financial plans or strategies related to the timing of the fund release.

### Scenario
1. A lender is sanctioned, and their assets are moved to an escrow contract.
2. The borrower decides to override the sanction or the lender is removed from the sanctions list.
3. An external observer notices this change and calls `releaseEscrow()` before the account owner does.
4. The funds are released to the correct account, but at a time not chosen by the account owner.

### Fix
Modify the `releaseEscrow` function to only allow calls from the `account` owner:

```solidity
function releaseEscrow() public override {
    require(msg.sender == account, "Only account owner can release escrow");
    if (!canReleaseEscrow()) revert CanNotReleaseEscrow();
    uint256 amount = balance();
    address _account = account;
    address _asset = asset;
    asset.safeTransfer(_account, amount);
    emit EscrowReleased(_account, _asset, amount);
}
```

## L-10: isKnownLenderOnMarket Status Not Updated on First Deposit With onTransfer Hook

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/AccessControlHooks.sol#L713

### Summary

The Wildcat protocol allows for flexible lending markets with customizable rules enforced through hooks. The AccessControlHooks contract enables a borrower to set restrictions on deposits and transfers, including minimum deposit amounts and approval requirements for lenders. To track lenders who have successfully made their first deposit, the protocol uses a isKnownLenderOnMarket mapping. This mapping is crucial for optimizing withdrawal checks, as known lenders are exempt from certain restrictions. However, a critical issue arises when a user first receives market tokens through a transfer rather than a direct deposit, as the isKnownLenderOnMarket status is not updated correctly in this scenario.

### Details

The _writeLenderStatus function in the AccessControlHooks contract is responsible for updating the isKnownLenderOnMarket mapping. This function is designed to set the status to true for an account if: a) the function call is triggered by a deposit (canSetKnownLender is true), b) the account has a valid credential (hasValidCredential is true), and c) the account is not already marked as a known lender for the specific market.

The onDeposit hook explicitly calls _writeLenderStatus with canSetKnownLender set to true, ensuring that the isKnownLenderOnMarket status is updated correctly when a user makes a direct deposit.

The onTransfer hook, while designed to handle transfers of market tokens between accounts, does not call _writeLenderStatus with canSetKnownLender set to true. This oversight means that when a user receives market tokens for the first time through a transfer, their isKnownLenderOnMarket status remains false, even though they should be considered a known lender.

### Impact

This bug can lead to several inconsistencies and potential issues:

The most immediate impact is the persistent inaccuracy of the isKnownLenderOnMarket status for users who receive their first market tokens via transfers. This contradicts the intended behavior of the protocol, which aims to track all lenders who have successfully interacted with the market.

Known lenders benefit from relaxed withdrawal requirements in certain scenarios. Due to this bug, users who become lenders through transfers might face unnecessary delays or complications during withdrawals, as the protocol incorrectly identifies them as not having made a prior deposit.


### Scenario

- A borrower deploys a new market with the AccessControlHooks contract and enables the onTransfer hook.
- A user (User A) makes an initial deposit, becoming a known lender (isKnownLenderOnMarket is set to true for User A).
- User A transfers some of their market tokens to another user (User B), who has never interacted with the market before.
- Despite receiving market tokens, User B's isKnownLenderOnMarket status remains false, as the onTransfer hook fails to update it.

### Fix

To address this issue, the onTransfer hook must also update the isKnownLenderOnMarket status for recipients who are not yet recognized as known lenders.

Within the onTransfer function, after validating the recipient's credential, call _writeLenderStatus with the canSetKnownLender argument set to true. This will ensure that the recipient's status is updated correctly, marking them as a known lender if they successfully pass the credential check.

## L-11: Lack of Fallback Mechanism for Sanctions Checks During Chainalysis Downtime

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/WildcatSanctionsSentinel.sol#L77

The Wildcat protocol relies on a Chainalysis sanctions list to identify and restrict sanctioned addresses. The isFlaggedByChainalysis function interacts with the chainalysisSanctionsList contract to determine if an address is flagged. However, the current implementation lacks a fallback mechanism to handle situations where the Chainalysis service might be unavailable, potentially leading to a disruption of the sanctions enforcement.

### Details

The protocol solely depends on the availability and responsiveness of the Chainalysis sanctions list contract. Any downtime or unresponsiveness from this external service could render the entire sanctions checking mechanism inoperable.

The code doesn't include an alternative method or data source for verifying sanctions if the Chainalysis contract is inaccessible.

This lack of redundancy could disrupt or halt crucial market operations that rely on sanctions checks. For instance, deposits, withdrawals, transfers, or even borrower actions like borrowing or setting parameters might be blocked entirely if the Chainalysis service is down, even for non-sanctioned addresses.

### Impact

The absence of a fallback mechanism for sanctions checks during Chainalysis downtime could severely hinder the protocol's functionality and security:

Denial of Service: Legitimate users might be unable to interact with the market if the Chainalysis service experiences temporary downtime or network issues.
Security Vulnerability: A prolonged outage of the Chainalysis API could create a window for sanctioned addresses to bypass checks and potentially interact with the protocol, undermining the intended security measures.


### Fix

To address this single point of failure, implement a fallback mechanism that ensures the continuity of sanctions checks even when the Chainalysis service is unavailable:

Local Cache: Maintain a local cache of recently flagged addresses within the WildcatMarket contract. This cache can be updated periodically and used as a secondary source of truth when the Chainalysis API is inaccessible.

Circuit Breaker Pattern: Implement a circuit breaker pattern that monitors the responsiveness of the Chainalysis contract. If repeated failures or timeouts occur, the circuit breaker can trip, temporarily switching to a more lenient mode, perhaps relying on the local cache or allowing transactions with additional verification steps.

Secondary Data Source: Consider integrating a secondary sanctions list provider as a backup. This provider could be another on-chain service or a decentralized oracle network that provides similar data points.

## L-12: Inconsistent Market Closure Checks

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarket.sol

The Wildcat Market contract employs the state.isClosed flag to manage actions when a market is closed. While some functions, such as deposit, borrow, collectFees, _repay, repayOutstandingDebt, and repayDelinquentDebt, diligently verify this flag, several other functions that interact with the market do not consistently enforce this check.

### Details

The following functions lack an isClosed check:

rescueTokens: Allows the borrower to recover tokens sent to the market contract by mistake. A closed market should ideally prevent further token interactions, even by the borrower.

setMaxTotalSupply: Enables the borrower to modify the maximum token supply. Allowing this action on a closed market contradicts the concept of a closed market, which typically implies fixed parameters.

setAnnualInterestAndReserveRatioBips: Allows adjustments to interest rates and reserve ratios, typically controlled by the borrower. Similar to setMaxTotalSupply, this action seems incongruous with a closed market.

setProtocolFeeBips: Permits updating the protocol fee, controlled by the factory. While fees might not be directly relevant in a closed market, preventing changes ensures consistency.

### Impact

This inconsistency in market closure checks introduces ambiguity and potential risks:

Unintended Token Flows: rescueTokens on a closed market might allow the borrower to manipulate token balances in unintended ways, potentially impacting outstanding obligations or withdrawals.

Conflicting Market State: Allowing parameter changes like setMaxTotalSupply, setAnnualInterestAndReserveRatioBips, and setProtocolFeeBips after closure creates a discrepancy between the isClosed status and the market's actual state, potentially confusing participants and complicating accounting.

### Fix

To ensure consistent behavior and mitigate risks, implement the isClosed check in all relevant functions. One approach is to introduce a modifier:

```solidity
modifier onlyOpenMarket() {
    MarketState memory state = _getUpdatedState();
    if (state.isClosed) revert_MarketClosed();
    _;
}

// Example of applying the modifier
function rescueTokens(address token) external onlyBorrower onlyOpenMarket {
    // ... function logic ...
}
```

Apply this modifier, or equivalent inline checks, to rescueTokens, setMaxTotalSupply, setAnnualInterestAndReserveRatioBips, and setProtocolFeeBips to enforce the isClosed condition.

## L-13: Missing Event Emission for rescueTokens

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarket.sol#L37

### Summary

The rescueTokens function in the Wildcat Market contract allows the borrower to recover tokens sent to the contract in error. However, this function currently lacks an event emission, which is crucial for transparency and monitoring.

### Details

The rescueTokens function does not emit an event after a successful token transfer. Event emissions serve as on-chain records, enabling off-chain systems and observers to track significant contract actions.

### Impact

The absence of an event emission for rescueTokens hinders the ability to monitor and audit token recoveries:

Limited Transparency: Without an event, identifying when, why, and which tokens were rescued becomes challenging. This lack of transparency could raise concerns about potential misuse of the function, even if used as intended.

Monitoring Difficulties: Off-chain systems relying on events to track token movements and balances within the market will miss these rescue operations, leading to inaccurate records and potentially complicating accounting or analysis.

Security Implications: While the function is restricted to the borrower, a lack of event records makes it difficult to investigate suspicious activity retroactively if a borrower's key is compromised.

### Fix

To enhance transparency and security, emit an event after a successful token rescue in the rescueTokens function. The event should include relevant details about the operation:

```solidity
// ... other contract code ...

event TokensRescued(address indexed token, address indexed rescuer, uint256 amount);

function rescueTokens(address token) external onlyBorrower onlyOpenMarket {
    // ... existing function logic ...

    token.safeTransferAll(msg.sender);

    emit TokensRescued(token, msg.sender, amount);
}
```

The TokensRescued event includes the token address, rescuer address, and the amount rescued. You can customize these parameters based on your monitoring and auditing requirements.

## L-14: Asset Transfer Before Market Closure Check in repay Function

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarket.sol#L202

The repay function in the Wildcat Market contract allows borrowers to repay their debts. However, the current implementation exhibits a potential vulnerability where the asset transfer occurs before the market closure status is verified.

### Impact

This sequence of actions introduces a risk of processing repayments even when the market is closed:

Unintended Asset Transfers: If a borrower initiates a repayment after a market has closed but before the transaction is mined, the asset transfer might still succeed. This could result in assets being sent to a closed market where they cannot be utilized as intended.

Accounting Discrepancies: Processing a repayment on a closed market could lead to inconsistencies between the market's internal state (debts, balances) and its closed status.

Complications in Market Closure: If a market is closed with outstanding repayments that were processed due to this bug, it could complicate procedures for settling final balances and distributing remaining assets.

### Fix

To mitigate this risk, the market closure check should be performed before executing the asset transfer:

```solidity
function repay(uint256 amount) external nonReentrant sphereXGuardExternal {
    if (amount == 0) revert_NullRepayAmount();

    // Check market state FIRST
    MarketState memory state = _getUpdatedState();
    if (state.isClosed) revert_RepayToClosedMarket();

    // Proceed with the transfer if the market is open
    asset.safeTransferFrom(msg.sender, address(this), amount);

    emit_DebtRepaid(msg.sender, amount);

    // ... rest of the function logic ...
}
```

## L-15: closeMarket Lacks Check for Unpaid Withdrawals

https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/market/WildcatMarket.sol#L226

The closeMarket function in the Wildcat Market contract aims to gracefully shut down a market, settling outstanding debts and distributing remaining assets. However, the current implementation does not explicitly ensure that all pending withdrawal batches are fully paid off before the market is marked as closed.

### Details

Depending on whether the market holds excess assets or debts, the function settles the difference by transferring assets to the appropriate party (borrower or contract).

The function resets key parameters such as interest rate (annualInterestBips), delinquency status (timeDelinquent), and sets the isClosed flag to true, effectively closing the market.

The function attempts to process the pending withdrawal batch, if any, followed by iterating and processing unpaid withdrawal batches.

The function only includes a check to revert if state.scaledPendingWithdrawals is not zero after attempting to process withdrawal batches.

### Impact

The current logic presents a potential flaw. While the function attempts to process pending and unpaid withdrawals, the absence of an upfront check before marking the market as closed creates a risk of incomplete withdrawal settlements. The sole reliance on state.scaledPendingWithdrawals after the processing loop might not guarantee all withdrawals are settled, especially if insufficient liquidity exists to cover them.

### Fix

To enhance the robustness of the closeMarket function and guarantee all pending withdrawals are settled before closure:

Introduce upfront check for the existence of any non-empty unpaid withdrawal batches. If found, revert the closure process, preventing premature closure.

// ... Inside the closeMarket function ...

// Check for unpaid withdrawal batches BEFORE marking the market as closed
if (_withdrawalData.unpaidBatches.length() > 0) {
    revert_CloseMarketWithUnpaidWithdrawals();
}

// Proceed with market parameter updates and closure...

### Considerations

Liquidity Requirements: Enforcing full withdrawal settlement might necessitate the borrower injecting additional liquidity into the market before closure if existing assets are insufficient.

Borrower Responsibilities: Document the requirement for borrowers to ensure adequate liquidity for pending withdrawals before initiating market closure.

By integrating these checks and potentially enhancing reporting, the Wildcat Market contract can ensure a more reliable and transparent market closure process, instilling greater confidence among participants regarding the proper handling of their funds, even as a market concludes its operations.
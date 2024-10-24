---
sponsor: "The Wildcat Protocol"
slug: "2024-08-wildcat"
date: "2024-10-24"
title: "The Wildcat Protocol"
findings: "https://github.com/code-423n4/2024-08-wildcat-findings/issues"
contest: 434
---

# Overview

## About C4

Code4rena (C4) is an open organization consisting of security researchers, auditors, developers, and individuals with domain expertise in smart contracts.

A C4 audit is an event in which community participants, referred to as Wardens, review, audit, or analyze smart contract logic in exchange for a bounty provided by sponsoring projects.

During the audit outlined in this document, C4 conducted an analysis of the Wildcat Protocol smart contract system written in Solidity. The audit took place between August 31—September 18 2024.

## Wardens

24 Wardens contributed reports to the Wildcat Protocol audit:

  1. [deadrxsezzz](https://code4rena.com/@deadrxsezzz)
  2. [Infect3d](https://code4rena.com/@Infect3d)
  3. [0xpiken](https://code4rena.com/@0xpiken)
  4. [pfapostol](https://code4rena.com/@pfapostol)
  5. [gesha17](https://code4rena.com/@gesha17)
  6. [falconhoof](https://code4rena.com/@falconhoof)
  7. [0xNirix](https://code4rena.com/@0xNirix)
  8. [Bauchibred](https://code4rena.com/@Bauchibred)
  9. [josephxander](https://code4rena.com/@josephxander)
  10. [0x1771](https://code4rena.com/@0x1771)
  11. [K42](https://code4rena.com/@K42)
  12. [kutugu](https://code4rena.com/@kutugu)
  13. [Bigsam](https://code4rena.com/@Bigsam)
  14. [Udsen](https://code4rena.com/@Udsen)
  15. [Takarez](https://code4rena.com/@Takarez)
  16. [0xriptide](https://code4rena.com/@0xriptide)
  17. [NexusAudits](https://code4rena.com/@NexusAudits) ([cheatc0d3](https://code4rena.com/@cheatc0d3) and [Zanna](https://code4rena.com/@Zanna))
  18. [air\_0x](https://code4rena.com/@air_0x)
  19. [Atarpara](https://code4rena.com/@Atarpara)
  20. [0xastronatey](https://code4rena.com/@0xastronatey)
  21. [saneryee](https://code4rena.com/@saneryee)
  22. [inh3l](https://code4rena.com/@inh3l)
  23. [shaflow2](https://code4rena.com/@shaflow2)

This audit was judged by [3docSec](https://code4rena.com/@3docsec).

Final report assembled by [liveactionllama](https://twitter.com/liveactionllama).

# Summary

The C4 analysis yielded an aggregated total of 9 unique vulnerabilities. Of these vulnerabilities, 1 received a risk rating in the category of HIGH severity and 8 received a risk rating in the category of MEDIUM severity.

Additionally, C4 analysis included 22 reports detailing issues with a risk rating of LOW severity or non-critical.

All of the issues presented here are linked back to their original finding.

# Scope

The code under review can be found within the [C4 The Wildcat Protocol repository](https://github.com/code-423n4/2024-08-wildcat), and is composed of 19 smart contracts written in the Solidity programming language and includes 3,784 lines of Solidity code.

# Severity Criteria

C4 assesses the severity of disclosed vulnerabilities based on three primary risk categories: high, medium, and low/non-critical.

High-level considerations for vulnerabilities span the following key areas when conducting assessments:

- Malicious Input Handling
- Escalation of privileges
- Arithmetic
- Gas use

For more information regarding the severity criteria referenced throughout the submission review process, please refer to the documentation provided on [the C4 website](https://code4rena.com), specifically our section on [Severity Categorization](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization).

# High Risk Findings (1)
## [[H-01] User could withdraw more than supposed to, forcing last user withdraw to fail](https://github.com/code-423n4/2024-08-wildcat-findings/issues/64)
*Submitted by [deadrxsezzz](https://github.com/code-423n4/2024-08-wildcat-findings/issues/64)*

Within Wildcat, withdraw requests are put into batches. Users first queue their withdraws and whenever there's sufficient liquidity, they're filled at the current rate. Usually, withdraw requests are only executable after the expiry passes and then all users within the batch get a cut from the `batch.normalizedAmountPaid` proportional to the scaled amount they've requested a withdraw for.

```solidity
    uint128 newTotalWithdrawn = uint128(
      MathUtils.mulDiv(batch.normalizedAmountPaid, status.scaledAmount, batch.scaledTotalAmount)
    );
```

This makes sure that the sum of all withdraws doesn't exceed the total `batch.normalizedAmountPaid`.

However, this invariant could be broken, if the market is closed as it allows for a batch's withdraws to be executed, before all requests are added.

Consider the market is made of 3 lenders - Alice, Bob and Laurence.

1.  Alice queues a larger withdraw with an expiry time 1 year in the future.
2.  Market gets closed.
3.  Alice executes her withdraw request at the current rate.
4.  Bob makes queues multiple smaller requests. As they're smaller, the normalized amount they represent suffers higher precision loss. Because they're part of the whole batch, they also slightly lower the batch's overall rate.
5.  Bob executes his requests.
6.  Laurence queues a withdraw for his entire amount. When he attempts to execute it, it will fail. This is because Alice has executed her withdraw at a higher rate than the current one and there's now insufficient `state.normalizedUnclaimedWithdrawals`

Note: marking this as High severity as it both could happen intentionally (attacker purposefully queuing numerous low-value withdraws to cause rounding down) and also with normal behaviour in high-value closed access markets where a user's withdraw could easily be in the hundreds of thousands.

Also breaks core invariant:

> The sum of all transfer amounts for withdrawal executions in a batch must be less than or equal to batch.normalizedAmountPaid

Adding a PoC to showcase the issue:

```solidity
  function test_deadrosesxyzissue() external {
    parameters.annualInterestBips = 3650;
    _deposit(alice, 1e18);
    _deposit(bob, 0.5e18);
    address laurence = address(1337);
    _deposit(laurence, 0.5e18);
    fastForward(200 weeks);

    vm.startPrank(borrower);
    asset.approve(address(market), 10e18);
    asset.mint(borrower, 10e18);

    vm.stopPrank();
    vm.prank(alice);
    uint32 expiry = market.queueFullWithdrawal();       // alice queues large withdraw

    vm.prank(borrower);
    market.closeMarket();                               // market is closed

    market.executeWithdrawal(alice, expiry);     // alice withdraws at the current rate

    vm.startPrank(bob);
    for (uint i; i < 10; i++) {
      market.queueWithdrawal(1);        // bob does multiple small withdraw requests just so they round down the batch's overall rate
    }
    market.queueFullWithdrawal();
    vm.stopPrank();
    vm.prank(laurence);
    market.queueFullWithdrawal();

    market.executeWithdrawal(bob, expiry);     // bob can successfully withdraw all of his funds

    vm.expectRevert();
    market.executeWithdrawal(laurence, expiry);    // laurence cannot withdraw his funds. Scammer get scammed.

  }
```

### Recommended Mitigation Steps

Although it's not a clean fix, consider adding a `addNormalizedUnclaimedRewards` function which can only be called after a market is closed. It takes token from the user and increases the global variable `state.normalizedUnclaimedRewards`. The invariant would remain broken, but it will make sure no funds are permanently stuck.

**[laurenceday (Wildcat) confirmed and commented](https://github.com/code-423n4/2024-08-wildcat-findings/issues/64#issuecomment-2388311632):**
 > We're going to have to dig into this, but we're confirming. Thank you!

**[3docSec (judge) commented](https://github.com/code-423n4/2024-08-wildcat-findings/issues/64#issuecomment-2391436961):**
 > I am confirming as High, under the assumption that funds can't be recovered (didn't see a `cancelWithdrawal` or similar option).

**[laurenceday (Wildcat) commented](https://github.com/code-423n4/2024-08-wildcat-findings/issues/64#issuecomment-2406135269):**
 > Fixed by the mitigation for [M-01](https://github.com/code-423n4/2024-08-wildcat-findings/issues/121#issuecomment-2406134173):<br>
 > [wildcat-finance/v2-protocol@b25e528](https://github.com/wildcat-finance/v2-protocol/commit/b25e528420617504f1ae6393b4be281a967c3e41).


***
 
# Medium Risk Findings (8)
## [[M-01] Users are incentivized  to not withdraw immediately after the market is closed](https://github.com/code-423n4/2024-08-wildcat-findings/issues/121)
*Submitted by [deadrxsezzz](https://github.com/code-423n4/2024-08-wildcat-findings/issues/121)*

Within a withdraw batch, all users within said batch are paid equally - at the same rate, despite what exactly was the rate when each individual one created their withdraw.

While this usually is not a problem as it is a way to reward users who queue the withdrawal and start the expiry cooldown, it creates a problematic situation when the market is closed with an outstanding expiry batch.

The problem is that up until the expiry timestamp comes, all new withdraw requests are added to this old batch where the rate of the previous requests drags the overall withdraw rate down.

Consider the following scenario:

1.  A withdraw batch is created and its expiry time is 1 year.
2.  6 months in, the withdraw batch has half of the markets value in it and the market is closed. The current rate is `1.12` and the batch is currently filled at `1.06`
3.  Now users have two choices - to either withdraw their funds now at `~1.06` rate or wait 6 months to be able to withdraw their funds at `1.12` rate.

This creates a very unpleasant situation as the users have an incentive to hold their funds within the contract, despite not providing any value.

Looked from slightly different POV, these early withdraw requesters force everyone else to lock their funds for additional 6 months, for the APY they should've usually received for just holding up until now.

### Recommended Mitigation Steps

After closing a market and filling the current expiry, delete it from `pendingWithdrawalExpiry`. Introduce a `closedExpiry` variable so you later make sure a future expiry is not made at that same timestamp to avoid collision.

**[d1ll0n (Wildcat) confirmed and commented](https://github.com/code-423n4/2024-08-wildcat-findings/issues/121#issuecomment-2403567138):**
 > Thanks for this, good find! Will adopt the proposed solution and see if it fixes [H-01](https://github.com/code-423n4/2024-08-wildcat-findings/issues/64).

**[laurenceday (Wildcat) commented](https://github.com/code-423n4/2024-08-wildcat-findings/issues/121#issuecomment-2406134173):**
 > Fixed with [wildcat-finance/v2-protocol@b25e528](https://github.com/wildcat-finance/v2-protocol/commit/b25e528420617504f1ae6393b4be281a967c3e41).


***

## [[M-02] `FixedTermLoanHooks` allow Borrower to update Annual Interest before end of the "Fixed Term Period"](https://github.com/code-423n4/2024-08-wildcat-findings/issues/77)
*Submitted by [Infect3d](https://github.com/code-423n4/2024-08-wildcat-findings/issues/77), also found by [0xpiken](https://github.com/code-423n4/2024-08-wildcat-findings/issues/95) and [falconhoof](https://github.com/code-423n4/2024-08-wildcat-findings/issues/23)*

### Summary

While the documentation states that in case of 'fixed term' market the APR cannot be changed until the term ends, nothing prevents this in `FixedTermLoanHooks`.

### Vulnerability details

In Wildcat markets, lenders know in advance how much `APR` the borrower will pay them. In order to allow lenders to exit the market swiftly, the market must always have at least a `reserve ratio` of the lender funds ready to be withdrawn.

If the borrower decides to [reduce the `APR`](https://docs.wildcat.finance/using-wildcat/day-to-day-usage/borrowers#reducing-apr), in order to allow lenders to 'ragequit', a new `reserve ratio` is calculated based on the variation of the APR as described in the link above.

Finally, is a market implement a fixed term (date until when withdrawals are not possible), it shouldn't be able to reduce the APR, as this would allow the borrower to 'rug' the lenders by reducing the APR to 0% while they couldn't do anything against that.

The issue here is that while lenders are (as expected) prevented to withdraw before end of term:<br>
<https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/FixedTermLoanHooks.sol#L857-L859>

```solidity
File: src/access/FixedTermLoanHooks.sol
848:   function onQueueWithdrawal(
849:     address lender,
850:     uint32 /* expiry */,
851:     uint /* scaledAmount */,
852:     MarketState calldata /* state */,
853:     bytes calldata hooksData
854:   ) external override {
855:     HookedMarket memory market = _hookedMarkets[msg.sender];
856:     if (!market.isHooked) revert NotHookedMarket();
857:     if (market.fixedTermEndTime > block.timestamp) {
858:       revert WithdrawBeforeTermEnd();
859:     }
```

this is not the case for the borrower setting the annual interest:<br>
<https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/FixedTermLoanHooks.sol#L960-L978>

```solidity
File: src/access/FixedTermLoanHooks.sol
960:   function onSetAnnualInterestAndReserveRatioBips(
961:     uint16 annualInterestBips,
962:     uint16 reserveRatioBips,
963:     MarketState calldata intermediateState,
964:     bytes calldata hooksData
965:   )
966:     public
967:     virtual
968:     override
969:     returns (uint16 updatedAnnualInterestBips, uint16 updatedReserveRatioBips)
970:   {
971:     return
972:       super.onSetAnnualInterestAndReserveRatioBips(
973:         annualInterestBips,
974:         reserveRatioBips,
975:         intermediateState,
976:         hooksData
977:       );
978:   }
979: 
```

### Impact

Borrower can rug the lenders by reducing the APR while they cannot quit the market.

### Proof of Concept

Add this test to `test/access/FixedTermLoanHooks.t.sol`

      function testAudit_SetAnnualInterestBeforeTermEnd() external {
        DeployMarketInputs memory inputs;

    	// "deploying" a market with MockFixedTermLoanHooks
    	inputs.hooks = EmptyHooksConfig.setHooksAddress(address(hooks));
    	hooks.onCreateMarket(
    		address(this),				// deployer
    		address(1),				// dummy market address
    		inputs,					// ...
    		abi.encode(block.timestamp + 365 days, 0) // fixedTermEndTime: 1 year, minimumDeposit: 0
    	);

    	vm.prank(address(1));
    	MarketState memory state;
    	// as the fixedTermEndTime isn't past yet, it's not possible to withdraw
    	vm.expectRevert(FixedTermLoanHooks.WithdrawBeforeTermEnd.selector);
    	hooks.onQueueWithdrawal(address(1), 0, 1, state, '');

    	// but it is still possible to reduce the APR to zero
    	hooks.onSetAnnualInterestAndReserveRatioBips(0, 0, state, "");
      }

### Recommended Mitigation Steps

When `FixedTermLoanHooks::onSetAnnualInterestAndReserveRatioBips` is called, revert if `market.fixedTermEndTime > block.timestamp`.

**[laurenceday (Wildcat) disputed and commented via duplicate issue \#23](https://github.com/code-423n4/2024-08-wildcat-findings/issues/23#issuecomment-2368151696):**
 > This is a valid finding, thank you - an embarrassing one for us at that, we clearly just missed this when writing the hook templates!
> 
> However, we're a bit torn internally on whether this truly classifies as a High. We've definitely specified in documentation that this is a rug pull mechanic, but there are no funds directly or indirectly at risk here, unless you classify the potential of _earning_ less than expected when you initially deposited as falling in that bucket.
> 
> So, we're going to kick this one to the judge: does earning 10,000 on a 100,000 deposit rather than 15,000 count as funds at risk if there's no way to ragequit for the period of time where that interest should accrue? Or is this more of a medium wherein protocol functionality is impacted?
> 
> It's definitely a goof on our end, and we're appreciative that the warden caught it, so thank you. With that said, we're trying to be fair to you (the warden) while also being fair to everyone else that's found things. This is a _very_ gentle dispute for the judge to handle: sadly the 'disagree with severity' tag isn't available to us anymore!

**[3docSec (judge) decreased severity to Medium and commented via duplicate issue \#23](https://github.com/code-423n4/2024-08-wildcat-findings/issues/23#issuecomment-2393321534):**
 > Hi @laurenceday thanks for adding context.
> 
> > "So, we're going to kick this one to the judge: does earning 10,000 on a 100,000 deposit rather than 15,000 count as funds at risk if there's no way to ragequit for the period of time where that interest should accrue? Or is this more of a medium wherein protocol functionality is impacted?"
> 
> I consider this a Medium issue: because it's only future "interest" gains that are at risk, I see this more like an availability issue where the lender's funds are locked at conditions they didn't necessarily sign up for; the problem is the locking (as you said if there was a ragequit option, it would be a different story).
> 
> I admit this is a subjective framing, but at the same time, it's consistent with how severity is assessed in bug bounty programs, where missing out on future returns generally has lower severity than having present funds at risk.


***

## [[M-03] Inconsistency across multiple repaying functions causing lender to pay extra fees](https://github.com/code-423n4/2024-08-wildcat-findings/issues/62)
*Submitted by [deadrxsezzz](https://github.com/code-423n4/2024-08-wildcat-findings/issues/62), also found by [Bigsam](https://github.com/code-423n4/2024-08-wildcat-findings/issues/100), [0xNirix](https://github.com/code-423n4/2024-08-wildcat-findings/issues/96), [Udsen](https://github.com/code-423n4/2024-08-wildcat-findings/issues/72), [Takarez](https://github.com/code-423n4/2024-08-wildcat-findings/issues/47), and [Infect3d](https://github.com/code-423n4/2024-08-wildcat-findings/issues/73)*

Within functions such as `repay` and `repayAndProcessUnpaidWithdrawalBatches`, funds are first pulled from the user in order to use them towards the currently expired, but not yet unpaid batch, and then the updated state is fetched.

```solidity
  function repay(uint256 amount) external nonReentrant sphereXGuardExternal {
    if (amount == 0) revert_NullRepayAmount();

    asset.safeTransferFrom(msg.sender, address(this), amount);
    emit_DebtRepaid(msg.sender, amount);

    MarketState memory state = _getUpdatedState();
    if (state.isClosed) revert_RepayToClosedMarket();

    // Execute repay hook if enabled
    hooks.onRepay(amount, state, _runtimeConstant(0x24));

    _writeState(state);
  }
```

However, this is not true for functions such as `closeMarket`, `deposit`, `repayOutstandingDebt` and `repayDelinquentDebt`, where the state is first fetched and only then funds are pulled, forcing borrower into higher fees.

```solidity
  function closeMarket() external onlyBorrower nonReentrant sphereXGuardExternal {
    MarketState memory state = _getUpdatedState();    // fetches updated state

    if (state.isClosed) revert_MarketAlreadyClosed();

    uint256 currentlyHeld = totalAssets();
    uint256 totalDebts = state.totalDebts();
    if (currentlyHeld < totalDebts) {
      // Transfer remaining debts from borrower
      uint256 remainingDebt = totalDebts - currentlyHeld;
      _repay(state, remainingDebt, 0x04);             // pulls user funds
      currentlyHeld += remainingDebt;
```

This inconsistency will cause borrowers to pay extra fees which they otherwise wouldn't.

**PoC:**

```solidity
  function test_inconsistencyIssue() external {
      parameters.annualInterestBips = 3650;
      _deposit(alice, 1e18);
      uint256 borrowAmount = market.borrowableAssets();
      vm.prank(borrower);
      market.borrow(borrowAmount);
      vm.prank(alice);
      market.queueFullWithdrawal();
      fastForward(52 weeks);

      asset.mint(borrower, 10e18);
      vm.startPrank(borrower);
      asset.approve(address(market), 10e18);
      uint256 initBalance = asset.balanceOf(borrower); 

      asset.transfer(address(market), 10e18);
      market.closeMarket();
      uint256 finalBalance = asset.balanceOf(borrower);
      uint256 paid = initBalance - finalBalance;
      console.log(paid);

  } 

    function test_inconsistencyIssue2() external {
      parameters.annualInterestBips = 3650;
      _deposit(alice, 1e18);
      uint256 borrowAmount = market.borrowableAssets();
      vm.prank(borrower);
      market.borrow(borrowAmount);
      vm.prank(alice);
      market.queueFullWithdrawal();
      fastForward(52 weeks);

      asset.mint(borrower, 10e18);
      vm.startPrank(borrower);
      asset.approve(address(market), 10e18);
      uint256 initBalance = asset.balanceOf(borrower); 


      market.closeMarket();
      uint256 finalBalance = asset.balanceOf(borrower);
      uint256 paid = initBalance - finalBalance;
      console.log(paid);

  }
```

and the logs:

    Ran 2 tests for test/market/WildcatMarket.t.sol:WildcatMarketTest
    [PASS] test_inconsistencyIssue() (gas: 656338)
    Logs:
      800455200405885337

    [PASS] test_inconsistencyIssue2() (gas: 680537)
    Logs:
      967625143234433533

### Recommended Mitigation Steps

Always pull the funds first and refund later if needed.

**[d1ll0n (Wildcat) acknowledged and commented](https://github.com/code-423n4/2024-08-wildcat-findings/issues/62#issuecomment-2388275386):**
 > The listed functions which incur higher fees all require the current state of the market to accurately calculate relevant values to the transfer. Because of that, the transfer can't happen until after the state is updated, and it would be expensive (and too large to fit in the contract size) to redo the withdrawal payments post-transfer.
> 
> For the repay functions this is more of an issue than the others, as that represents the borrower specifically taking action to repay their debts, whereas the other functions are actions by other parties (and thus we aren't very concerned if they fail to cure the borrower's delinquency for them). We may end up just removing these secondary repay functions.

**[laurenceday (Wildcat) commented](https://github.com/code-423n4/2024-08-wildcat-findings/issues/62#issuecomment-2403678176):**
 > Resolved by [wildcat-finance/v2-protocol@e7afdc9](https://github.com/wildcat-finance/v2-protocol/commit/e7afdc9312ec672df2a9d03add18727a4c774b88).


***

## [[M-04] `FixedTermLoanHook` looks at `block.timestamp` instead of `expiry`](https://github.com/code-423n4/2024-08-wildcat-findings/issues/60)
*Submitted by [deadrxsezzz](https://github.com/code-423n4/2024-08-wildcat-findings/issues/60)*

The idea of `FixedTermLoanHook` is to only allow for withdrawals after a certain term end time. However, the problem is that the current implementation does not look at the expiry, but instead at the `block.timestamp`.

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
```

This creates inconsistencies such as forcing users not only to wait until term's end, but also having to wait an extra `withdrawalBatchDuration` before they're able to withdraw their funds.

### Recommended Mitigation Steps

Check the `expiry` instead of `block.timestamp`.

**[d1ll0n (Wildcat) confirmed](https://github.com/code-423n4/2024-08-wildcat-findings/issues/60#event-14486191444)**

**[laurenceday (Wildcat) acknowledged and commented](https://github.com/code-423n4/2024-08-wildcat-findings/issues/60#issuecomment-2403650325):**
 > We've reflected on this a little bit, and decided that we want to turn this from a confirmed into an acknowledge.
> 
> The reasoning goes as follows:
> 
> Imagine that a fixed market has an expiry of December 30th, but there's a withdrawal cycle of 7 days.
> 
> Presumably the borrower is anticipating [and may have structured things] such that they are expecting to be able to make full use of any credit extended to them until then, and not a day sooner.
> 
> Fixing this in the way suggested would permit people to place withdrawal requests on December 23rd, with the potential to tip a market into delinquent status (depending on grace period configuration) before the fixed duration has actually met.
> 
> Net-net we think it makes more sense to allow the market to revert back to a perpetual after that expiry and allow withdrawal requests to be processed per the conditions. The expectation here would be that the withdrawal cycle would actually be quite short.

**[Infect3d (warden) commented](https://github.com/code-423n4/2024-08-wildcat-findings/issues/60#issuecomment-2405458024):**
 > May I comment on this issue: can we really consider this a bug rather than a feature and a design improvement, also considering sponsor comment?<br>
> Expiry mechanism is known by borrower and lender, so if borrower wants lenders to be able to withdraw on time, he can simply configure `fixedTermEndTime = value - withdrawalBatchDuration`.

**[3docSec (judge) commented](https://github.com/code-423n4/2024-08-wildcat-findings/issues/60#issuecomment-2411378025):**
 > Hi @Infect3d - I agree we are close to an accepted trade-off territory. Here I lean on the sponsor who very transparently made it clear this trade-off is not something they had deliberately thought of.
> 
> Therefore, because the impact is compatible with Medium severity, "satisfactory Medium" plus "sponsor acknowledged" is a fair way of categorizing this finding.


***

## [[M-05] Role providers can bypass intended restrictions and lower expiry set by other providers](https://github.com/code-423n4/2024-08-wildcat-findings/issues/57)
*Submitted by [deadrxsezzz](https://github.com/code-423n4/2024-08-wildcat-findings/issues/57), also found by [0x1771](https://github.com/code-423n4/2024-08-wildcat-findings/issues/86) and [gesha17](https://github.com/code-423n4/2024-08-wildcat-findings/issues/10)*

https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/FixedTermLoanHooks.sol#L413

If we look at the code comments, we'll see that role providers can update a user's credential only if at least one of the 3 is true:

*   the previous credential's provider is no longer supported, OR
*   the caller is the previous role provider, OR
*   the new expiry is later than the current expiry

```solidity
  /**
   * @dev Grants a role to an account by updating the account's status.
   *      Can only be called by an approved role provider.
   *
   *      If the account has an existing credential, it can only be updated if:
   *      - the previous credential's provider is no longer supported, OR
   *      - the caller is the previous role provider, OR
   *      - the new expiry is later than the current expiry
   */
  function grantRole(address account, uint32 roleGrantedTimestamp) external {
    RoleProvider callingProvider = _roleProviders[msg.sender];

    if (callingProvider.isNull()) revert ProviderNotFound();

    _grantRole(callingProvider, account, roleGrantedTimestamp);
  }
```

This means that a role provider should not be able to reduce a credential set by another role provider.

However, this could easily be bypassed by simply splitting the call into 2 separate ones:

1.  First one to set the expiry slightly later than the currently set one. This would set the role provider to the new one.
2.  Second call to reduce the expiry as much as they'd like. Since they're the previous provider they can do that.

### Recommended Mitigation Steps

Fix is non-trivial.

**[d1ll0n (Wildcat) disputed and commented](https://github.com/code-423n4/2024-08-wildcat-findings/issues/57#issuecomment-2384066409):**
 > This is a useful note to be aware of, but I'd categorize it low/informational as role providers are inherently trusted entities. The likelihood and impact of this kind of attack are pretty minimal.

**[3docSec (judge) commented](https://github.com/code-423n4/2024-08-wildcat-findings/issues/57#issuecomment-2393602205):**
 > There are a few factors to be considered:
> - it is a valid privilege escalation vector
> - the attacker has to be privileged already
> - the attack can have a direct impact on a lender
> 
> While the first two have me on the fence when choosing between Medium and Low severity, the third point is a tiebreaker towards Medium.
> 
> If we stick to the [C4 severity categorization](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization), I see a good fit with the Medium definition:
> 
> > "the function of the protocol or its availability could be impacted [...] with a hypothetical attack path with stated assumptions"


***

## [[M-06] No lender is able to exit even after the market is closed](https://github.com/code-423n4/2024-08-wildcat-findings/issues/52)
*Submitted by [0xpiken](https://github.com/code-423n4/2024-08-wildcat-findings/issues/52), also found by [josephxander](https://github.com/code-423n4/2024-08-wildcat-findings/issues/101) and [0xNirix](https://github.com/code-423n4/2024-08-wildcat-findings/issues/87)*

When a borrower creates a market hooked by a [fixed-term hook](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/FixedTermLoanHooks.sol), all lenders are prohibited from withdrawing their assets from the market before the fixed-term time has elapsed.

The borrower can close the market at any time. However, `fixedTermEndTime` of the market is not updated, preventing lenders from withdrawing their assets if `fixedTermEndTime` has not yet elapsed.

Copy below codes to [WildcatMarket.t.sol](https://github.com/code-423n4/2024-08-wildcat/blob/main/test/market/WildcatMarket.t.sol) and run forge test --match-test test_closeMarket_BeforeFixedTermExpired:

```solidity
  function test_closeMarket_BeforeFixedTermExpired() external {
    //@audit-info deploy a FixedTermLoanHooks template
    address fixedTermHookTemplate = LibStoredInitCode.deployInitCode(type(FixedTermLoanHooks).creationCode);
    hooksFactory.addHooksTemplate(
      fixedTermHookTemplate,
      'FixedTermLoanHooks',
      address(0),
      address(0),
      0,
      0
    );
    
    vm.startPrank(borrower);
    //@audit-info borrower deploy a FixedTermLoanHooks hookInstance
    address hooksInstance = hooksFactory.deployHooksInstance(fixedTermHookTemplate, '');
    DeployMarketInputs memory parameters = DeployMarketInputs({
      asset: address(asset),
      namePrefix: 'name',
      symbolPrefix: 'symbol',
      maxTotalSupply: type(uint128).max,
      annualInterestBips: 1000,
      delinquencyFeeBips: 1000,
      withdrawalBatchDuration: 10000,
      reserveRatioBips: 10000,
      delinquencyGracePeriod: 10000,
      hooks: EmptyHooksConfig.setHooksAddress(address(hooksInstance))
    });
    //@audit-info borrower deploy a market hooked by a FixedTermLoanHooks hookInstance
    address market = hooksFactory.deployMarket(
      parameters,
      abi.encode(block.timestamp + (365 days)),
      bytes32(uint(1)),
      address(0),
      0
    );
    vm.stopPrank();
    //@audit-info lenders can only withdraw their asset one year later
    assertEq(FixedTermLoanHooks(hooksInstance).getHookedMarket(market).fixedTermEndTime, block.timestamp + (365 days));
    //@audit-info alice deposit 50K asset into market
    vm.startPrank(alice);
    asset.approve(market, type(uint).max);
    WildcatMarket(market).depositUpTo(50_000e18);
    vm.stopPrank();
    //@audit-info borrower close market in advance
    vm.prank(borrower);
    WildcatMarket(market).closeMarket();
    //@audit-info the market is closed
    assertTrue(WildcatMarket(market).isClosed());
    //@audit-info however, alice can not withdraw her asset due to the unexpired fixed term.
    vm.expectRevert(FixedTermLoanHooks.WithdrawBeforeTermEnd.selector);
    vm.prank(alice);
    WildcatMarket(market).queueFullWithdrawal();
  }
```

### Recommended Mitigation Steps

When a market hooked by a fixed-term hook is closed, `fixedTermEndTime` should be set to `block.timestamp` if it has not yet elapsed:

```diff
  constructor(address _deployer, bytes memory /* args */) IHooks() {
    borrower = _deployer;
    // Allow deployer to grant roles with no expiry
    _roleProviders[_deployer] = encodeRoleProvider(
      type(uint32).max,
      _deployer,
      NotPullProviderIndex
    );
    HooksConfig optionalFlags = encodeHooksConfig({
      hooksAddress: address(0),
      useOnDeposit: true,
      useOnQueueWithdrawal: false,
      useOnExecuteWithdrawal: false,
      useOnTransfer: true,
      useOnBorrow: false,
      useOnRepay: false,
      useOnCloseMarket: false,
      useOnNukeFromOrbit: false,
      useOnSetMaxTotalSupply: false,
      useOnSetAnnualInterestAndReserveRatioBips: false,
      useOnSetProtocolFeeBips: false
    });
    HooksConfig requiredFlags = EmptyHooksConfig
      .setFlag(Bit_Enabled_SetAnnualInterestAndReserveRatioBips)
+     .setFlag(Bit_Enabled_CloseMarket);
      .setFlag(Bit_Enabled_QueueWithdrawal);
    config = encodeHooksDeploymentConfig(optionalFlags, requiredFlags);
  }
```

```diff
  function onCloseMarket(
    MarketState calldata /* state */,
    bytes calldata /* hooksData */
- ) external override {}
+ ) external override {
+   HookedMarket memory market = _hookedMarkets[msg.sender];
+   if (!market.isHooked) revert NotHookedMarket();
+   if (market.fixedTermEndTime > block.timestamp) {
+     _hookedMarkets[msg.sender].fixedTermEndTime = uint32(block.timestamp);
+   }
+ }
```

**[laurenceday (Wildcat) confirmed and commented](https://github.com/code-423n4/2024-08-wildcat-findings/issues/52#issuecomment-2364042816):**
 > This is a great catch.

**[laurenceday (Wildcat) commented](https://github.com/code-423n4/2024-08-wildcat-findings/issues/52#issuecomment-2403731338):**
 > Fixed by [wildcat-finance/v2-protocol@05958e3](https://github.com/wildcat-finance/v2-protocol/commit/05958e35995093bcf6941c82fc14f9b9acc7cea0).


***

## [[M-07] Role providers cannot be EOAs as stated in the documentation](https://github.com/code-423n4/2024-08-wildcat-findings/issues/49)
*Submitted by [pfapostol](https://github.com/code-423n4/2024-08-wildcat-findings/issues/49), also found by [Infect3d](https://github.com/code-423n4/2024-08-wildcat-findings/issues/74)*

### Lines of code

<https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/AccessControlHooks.sol#L220>

<https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/FixedTermLoanHooks.sol#L254>

### Impact

The [Documentation](https://docs.wildcat.finance/technical-overview/security-developer-dives/hooks/access-control-hooks#role-providers) suggests that a role provider can be a "push" provider (one that "pushes" credentials into the hooks contract by calling `grantRole`) and a "pull" provider (one that the hook calls via `getCredential` or `validateCredential`).

The documentation also states that:

> Role providers do not have to implement any of these functions - a role provider can be an EOA.

But in fact, only the initial deployer can be an EOA provider, since it is coded in the constructor. Any other EOA provider that the borrower tries to add via `addRoleProvider` will fail because it does not implement the interface.

### Proof of Concept

PoC will revert because EOA does not implement interface obviously:

<details>

```powershell
  [118781] AuditMarket::test_PoC_EOA_provider()
    ├─ [0] VM::startPrank(BORROWER1: [0xB193AC639A896a0B7a0B334a97f0095cD87427f2])
    │   └─ ← [Return]
    ├─ [29883] AccessControlHooks::addRoleProvider(RoleProvider: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 2592000 [2.592e6])
    │   ├─ [2275] RoleProvider::isPullProvider() [staticcall]
    │   │   └─ ← [Return] false
    │   ├─ emit RoleProviderAdded(providerAddress: RoleProvider: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], timeToLive: 2592000 [2.592e6], pullProviderIndex: 16777215 [1.677e7])
    │   └─ ← [Stop]
    ├─ [74243] AccessControlHooks::addRoleProvider(RoleProvider: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a], 2592000 [2.592e6])
    │   ├─ [2275] RoleProvider::isPullProvider() [staticcall]
    │   │   └─ ← [Return] true
    │   ├─ emit RoleProviderAdded(providerAddress: RoleProvider: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a], timeToLive: 2592000 [2.592e6], pullProviderIndex: 0)
    │   └─ ← [Stop]
    ├─ [5547] AccessControlHooks::addRoleProvider(EOA_PROVIDER1: [0x6aAfF89c996cAa2BD28408f735Ba7A441276B03F], 2592000 [2.592e6])
    │   ├─ [0] EOA_PROVIDER1::isPullProvider() [staticcall]
    │   │   └─ ← [Stop]
    │   └─ ← [Revert] EvmError: Revert
    └─ ← [Revert] EvmError: Revert
```

**PoC:**

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "forge-std/console2.sol";

import {WildcatArchController} from "../src/WildcatArchController.sol";
import {HooksFactory} from "../src/HooksFactory.sol";
import {LibStoredInitCode} from "src/libraries/LibStoredInitCode.sol";
import {WildcatMarket} from "src/market/WildcatMarket.sol";
import {AccessControlHooks} from "../src/access/AccessControlHooks.sol";
import {DeployMarketInputs} from "../src/interfaces/WildcatStructsAndEnums.sol";
import {HooksConfig, encodeHooksConfig} from "../src/types/HooksConfig.sol";

import {MockERC20} from "../test/shared/mocks/MockERC20.sol";
import {MockSanctionsSentinel} from "./shared/mocks/MockSanctionsSentinel.sol";
import {deployMockChainalysis} from "./shared/mocks/MockChainalysis.sol";
import {IRoleProvider} from "../src/access/IRoleProvider.sol";

contract AuditMarket is Test {
    WildcatArchController wildcatArchController;
    MockSanctionsSentinel internal sanctionsSentinel;
    HooksFactory hooksFactory;

    MockERC20 ERC0 = new MockERC20();

    address immutable ARCH_DEPLOYER = makeAddr("ARCH_DEPLOYER");
    address immutable FEE_RECIPIENT = makeAddr("FEE_RECIPIENT");
    address immutable BORROWER1 = makeAddr("BORROWER1");

    address immutable EOA_PROVIDER1 = makeAddr("EOA_PROVIDER1");
    address immutable PROVIDER1 = address(new RoleProvider(false));
    address immutable PROVIDER2 = address(new RoleProvider(true));

    address accessControlHooksTemplate = LibStoredInitCode.deployInitCode(type(AccessControlHooks).creationCode);

    AccessControlHooks accessControlHooksInstance;

    function _storeMarketInitCode() internal virtual returns (address initCodeStorage, uint256 initCodeHash) {
        bytes memory marketInitCode = type(WildcatMarket).creationCode;
        initCodeHash = uint256(keccak256(marketInitCode));
        initCodeStorage = LibStoredInitCode.deployInitCode(marketInitCode);
    }

    function setUp() public {
        deployMockChainalysis();
        vm.startPrank(ARCH_DEPLOYER);
        wildcatArchController = new WildcatArchController();
        sanctionsSentinel = new MockSanctionsSentinel(address(wildcatArchController));
        (address initCodeStorage, uint256 initCodeHash) = _storeMarketInitCode();
        hooksFactory =
            new HooksFactory(address(wildcatArchController), address(sanctionsSentinel), initCodeStorage, initCodeHash);

        wildcatArchController.registerControllerFactory(address(hooksFactory));
        hooksFactory.registerWithArchController();
        wildcatArchController.registerBorrower(BORROWER1);

        hooksFactory.addHooksTemplate(
            accessControlHooksTemplate, "accessControlHooksTemplate", FEE_RECIPIENT, address(ERC0), 1 ether, 500
        );
        vm.startPrank(BORROWER1);

        DeployMarketInputs memory marketInput = DeployMarketInputs({
            asset: address(ERC0),
            namePrefix: "Test",
            symbolPrefix: "TT",
            maxTotalSupply: uint128(100_000e27),
            annualInterestBips: uint16(500),
            delinquencyFeeBips: uint16(500),
            withdrawalBatchDuration: uint32(5 days),
            reserveRatioBips: uint16(500),
            delinquencyGracePeriod: uint32(5 days),
            hooks: encodeHooksConfig(address(0), true, true, false, true, false, false, false, false, false, true, false)
        });
        bytes memory hooksData = abi.encode(uint32(block.timestamp + 30 days), uint128(1e27));
        deal(address(ERC0), BORROWER1, 1 ether);
        ERC0.approve(address(hooksFactory), 1 ether);
        (address market, address hooksInstance) = hooksFactory.deployMarketAndHooks(
            accessControlHooksTemplate,
            abi.encode(BORROWER1),
            marketInput,
            hooksData,
            bytes32(bytes20(BORROWER1)),
            address(ERC0),
            1 ether
        );
        accessControlHooksInstance = AccessControlHooks(hooksInstance);
        vm.stopPrank();
    }

    function test_PoC_EOA_provider() public {
        vm.startPrank(BORROWER1);

        accessControlHooksInstance.addRoleProvider(PROVIDER1, uint32(30 days));
        accessControlHooksInstance.addRoleProvider(PROVIDER2, uint32(30 days));
        accessControlHooksInstance.addRoleProvider(EOA_PROVIDER1, uint32(30 days));
    }
}

contract RoleProvider is IRoleProvider {
    bool public isPullProvider;
    mapping(address account => uint32 timestamp) public getCredential;

    constructor(bool _isPullProvider) {
        isPullProvider = _isPullProvider;
    }

    function setCred(address account, uint32 timestamp) external {
        getCredential[account] = timestamp;
    }

    function validateCredential(address account, bytes calldata data) external returns (uint32 timestamp) {
        if (data.length != 0) {
            return uint32(block.timestamp);
        } else {
            revert("Wrong creds");
        }
    }
}
```

</details>

### Recommended Mitigation Steps

Replace the interface call with a low-level call and check if the user implements the interface in order to be a pull provider:

```solidity
(bool succes, bytes memory data) =
    providerAddress.call(abi.encodeWithSelector(IRoleProvider.isPullProvider.selector));
bool isPullProvider;
if (succes && data.length == 0x20) {
    isPullProvider = abi.decode(data, (bool));
} else {
    isPullProvider = false;
}
```

With this code all logic works as expected, for EOA providers `pullProviderIndex` is set to `type(uint24).max`, for contracts - depending on the result of calling `isPullProvider`:

```powershell
Traces:
  [141487] AuditMarket::test_PoC_EOA_provider()
    ├─ [0] VM::startPrank(BORROWER1: [0xB193AC639A896a0B7a0B334a97f0095cD87427f2])
    │   └─ ← [Return]
    ├─ [30181] AccessControlHooks::addRoleProvider(RoleProvider: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 2592000 [2.592e6])
    │   ├─ [2275] RoleProvider::isPullProvider()
    │   │   └─ ← [Return] false
    │   ├─ emit RoleProviderAdded(providerAddress: RoleProvider: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], timeToLive: 2592000 [2.592e6], pullProviderIndex: 16777215 [1.677e7])
    │   └─ ← [Stop]
    ├─ [74541] AccessControlHooks::addRoleProvider(RoleProvider: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a], 2592000 [2.592e6])
    │   ├─ [2275] RoleProvider::isPullProvider()
    │   │   └─ ← [Return] true
    │   ├─ emit RoleProviderAdded(providerAddress: RoleProvider: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a], timeToLive: 2592000 [2.592e6], pullProviderIndex: 0)
    │   └─ ← [Stop]
    ├─ [27653] AccessControlHooks::addRoleProvider(EOA_PROVIDER1: [0x6aAfF89c996cAa2BD28408f735Ba7A441276B03F], 2592000 [2.592e6])
    │   ├─ [0] EOA_PROVIDER1::isPullProvider()
    │   │   └─ ← [Stop]
    │   ├─ emit RoleProviderAdded(providerAddress: EOA_PROVIDER1: [0x6aAfF89c996cAa2BD28408f735Ba7A441276B03F], timeToLive: 2592000 [2.592e6], pullProviderIndex: 16777215 [1.677e7])
    │   └─ ← [Stop]
    └─ ← [Stop]
```

**[laurenceday (Wildcat) commented](https://github.com/code-423n4/2024-08-wildcat-findings/issues/49#issuecomment-2387986619):**
 > Not really a medium in that it doesn't 'matter' for the most part: this is sort of a documentation issue in that we'd never really expect an EOA that _wasn't_ the borrower (which is an EOA provider) to be a role provider.
> 
> It's vanishingly unlikely that a borrower is going to add some random arbiter that they don't control - possible that they add another address that THEY control but in that case they might as well use the one that's known to us.
> 
> Disputing, but with a light touch: we consider this a useful QA.

**[3docSec (judge) commented](https://github.com/code-423n4/2024-08-wildcat-findings/issues/49#issuecomment-2391862173):**
 > Thanks for the context. If we ignore the documentation, the fact that the initial role provider is the borrower, and the `NotPullProviderIndex` value that is used in this case makes it clear that the intention is allowing for EOAs to be there. 
> 
> While not the most requested feature, it's something a borrower may want to do, and given the above, may reasonably expect to see working. For this reason, I think a Medium is reasonable because we have marginal I admit, but still tangible, availability impact.


***

## [[M-08] `AccessControlHooks` `onQueueWithdrawal()` does not check if market is hooked which could lead to unexpected errors such as temporary DoS](https://github.com/code-423n4/2024-08-wildcat-findings/issues/11)
*Submitted by [gesha17](https://github.com/code-423n4/2024-08-wildcat-findings/issues/11), also found by [falconhoof](https://github.com/code-423n4/2024-08-wildcat-findings/issues/123), [kutugu](https://github.com/code-423n4/2024-08-wildcat-findings/issues/83), and [Infect3d](https://github.com/code-423n4/2024-08-wildcat-findings/issues/26)*

### Impact

The `onQueueWithdrawal()` function does not check if the caller is a hooked market, meaning anyone can call the function and attempt to verify credentials on a lender. This results in calls to registered pull providers with arbitrary hookData, which could lead to potential issues such as abuse of credentials that are valid for a short term, e.g. 1 block.

### Proof of Concept

The `onQueueWithdrawal()` function does not check if the msg.sender is a hooked market, which is standart in virtually all other hooks:

<https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L812>

```js
  /**
   * @dev Called when a lender attempts to queue a withdrawal.
   *      Passes the check if the lender has previously deposited or received
   *      market tokens while having the ability to deposit, or currently has a
   *      valid credential from an approved role provider.
   */
  function onQueueWithdrawal(
    address lender,
    uint32 /* expiry */,
    uint /* scaledAmount */,
    MarketState calldata /* state */,
    bytes calldata hooksData
  ) external override {
    LenderStatus memory status = _lenderStatus[lender];
    if (
      !isKnownLenderOnMarket[lender][msg.sender] && !_tryValidateAccess(status, lender, hooksData)
    ) {
      revert NotApprovedLender();
    }
  }
```

If the caller is not a hooked market, the statement `!isKnownLenderOnMarket[lender][msg.sender]`, will return true, because the lender will be unknown. As a result the `_tryValidateAccess()` function will be executed for any `lender` and any `hooksData` passed. The call to [`_tryValidateAccess()`](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L698) will forward the call to [`_tryValidateAccessInner()`](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L654). Choosing a lender of arbitrary address, say `address(1)` will cause the function to attempt to retrieve the credential via the call to [\_handleHooksData()](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L670), since the lender will have no previous provider or credentials.

As a result, the \_handleHooksData function will forward the call to the encoded provider in the hooksData and will forward the extra hooks data as well, say merkle proof, or any arbitrary malicious data.

<https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L617>

```js
  function _handleHooksData(
    LenderStatus memory status,
    address accountAddress,
    bytes calldata hooksData
  ) internal returns (bool validCredential) {
    // Check if the hooks data only contains a provider address
    if (hooksData.length == 20) {
      // If the data contains only an address, attempt to query a credential from that provider
      // if it exists and is a pull provider.
      address providerAddress = _readAddress(hooksData);
      RoleProvider provider = _roleProviders[providerAddress];
      if (!provider.isNull() && provider.isPullProvider()) {
        return _tryGetCredential(status, provider, accountAddress);
      }
    } else if (hooksData.length > 20) {
      // If the data contains both an address and additional bytes, attempt to
      // validate a credential from that provider
      return _tryValidateCredential(status, accountAddress, hooksData);
    }
  }
```

The function call will be executed in [tryValidateCredential()](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/AccessControlHooks.sol#L525), where the extra hookData will be forwarded. As described in the function comments, it will execute a call to `provider.(address account, bytes calldata data)`.

This means that anyone can call the function and pass arbitrary calldata. This can lead to serious vulnerabilities as the calldata is passed to the provider.

Consider the following scenario:

*   The pull provider is implemented to provide a short-term(say one block) approval timestamp.
*   A user of the protocol provides a merkle-proof which would grant the one-time approval to withdraw in a transaction.
*   A malicious miner frontruns the transaction submitting the same proof, but does not include the honest transaction in the mined block. Instead it is left for the next block.
*   In the next block, the credential is no longer valid and as a result the honest user has their transaction revert.
*   The miner does this continuosly essentially DoSing the entire market that uses this provider until it is removed and a new one added.

By following this scenario, a malicious user can essentially DoS a specific type pull provider.

Depending on implemenation of the pull provider, this can lead to other issues, as the malicious user can supply any arbitrary hookData in the function call.

### Recommended Mitigation Steps

Require the caller to be a registered hooked market, same as [onQueueWithdrawal()](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/FixedTermLoanHooks.sol#L848) in FixedTermloanHooks

**[3docSec (judge) commented via duplicate issue \#83](https://github.com/code-423n4/2024-08-wildcat-findings/issues/83#issuecomment-2404270998):**
 > I find this group compatible with the Medium severity for the following reasons:
> - access to a lender's signature is very feasible in the frontrunning scenario depicted in this finding
> - the hypothesis on `validateCredential` isn't really a speculation but rather a very reasonable implementation, one that was also assumed in the [previous audit (finding number 2)](https://hackmd.io/@geistermeister/BJk4Ekt90).

**[laurenceday (Wildcat) acknowledged and commented](https://github.com/code-423n4/2024-08-wildcat-findings/issues/11#issuecomment-2431829302):**
> We don't consider this a real issue, in that we’ve always wanted it to be possible for anyone to call the validate function to poke a credential update. This finding assumes that you have the signature someone else would be using as a credential and generally relies on a specific implementation of the provider that doesn’t actually exist, so there's no need to check `isHooked`.
> 
> It's been upgraded to a Medium, and we're not going to argue with this at this stage. As such, we're acknowledging rather than confirming or disputing simply to put a cap on the report.

***

# Low Risk and Non-Critical Issues

For this audit, 22 reports were submitted by wardens detailing low risk and non-critical issues. The [report highlighted below](https://github.com/code-423n4/2024-08-wildcat-findings/issues/119) by **Bauchibred** received the top score from the judge.

*The following wardens also submitted reports: [K42](https://github.com/code-423n4/2024-08-wildcat-findings/issues/116), [Infect3d](https://github.com/code-423n4/2024-08-wildcat-findings/issues/108), [0xriptide](https://github.com/code-423n4/2024-08-wildcat-findings/issues/117), [0xpiken](https://github.com/code-423n4/2024-08-wildcat-findings/issues/113), [falconhoof](https://github.com/code-423n4/2024-08-wildcat-findings/issues/17), [shaflow2](https://github.com/code-423n4/2024-08-wildcat-findings/issues/6), [Takarez](https://github.com/code-423n4/2024-08-wildcat-findings/issues/125), [0xNirix](https://github.com/code-423n4/2024-08-wildcat-findings/issues/115), [kutugu](https://github.com/code-423n4/2024-08-wildcat-findings/issues/114), [NexusAudits](https://github.com/code-423n4/2024-08-wildcat-findings/issues/111), [Udsen](https://github.com/code-423n4/2024-08-wildcat-findings/issues/109), [Bigsam](https://github.com/code-423n4/2024-08-wildcat-findings/issues/102), [air\_0x](https://github.com/code-423n4/2024-08-wildcat-findings/issues/88), [Atarpara](https://github.com/code-423n4/2024-08-wildcat-findings/issues/85), [0x1771](https://github.com/code-423n4/2024-08-wildcat-findings/issues/84), [0xastronatey](https://github.com/code-423n4/2024-08-wildcat-findings/issues/70), [deadrxsezzz](https://github.com/code-423n4/2024-08-wildcat-findings/issues/61), [pfapostol](https://github.com/code-423n4/2024-08-wildcat-findings/issues/50), [saneryee](https://github.com/code-423n4/2024-08-wildcat-findings/issues/41), [inh3l](https://github.com/code-423n4/2024-08-wildcat-findings/issues/36), and [gesha17](https://github.com/code-423n4/2024-08-wildcat-findings/issues/9).*

## [01] Market can immediately fall into deliquency

### Proof of Concept

First, note that the protocol allows borrowers to set a reserve ratio that they must maintain to avoid being charged a delinquency fee.

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

## [02] Use a pull pattern instead of push when collecting fees

### Proof of Concept

First, note that from the readMe, this has been stated: https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/README.md#L276

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

## [03] Borrowers would pay lesser APR in some supported assets

### Proof of Concept

First, per the readMe, we should assume rebasing tokens are in scope, see https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/README.md#L262

```markdown
| [Balance changes outside of transfers](https://github.com/d-xo/weird-erc20?tab=readme-ov-file#balance-modifications-outside-of-transfers-rebasingairdrops) | In scope    |
```

Where as this integration is welcome as the WIldMarketTokens themselves are somewhat rebasing in nature, this then means that users could pay lesser APR, which is because if they are used as underlying assets for markets, when the borrower/market contracts hold these tokens while they are lent, the newly accrued tokens may either be credited to the borrower, or inside the market itself, which in our case would count as the borrower adding liquidity. And result in the borrower _needing_ to pay a lower Annual Percentage Rate (APR) than initially set.

### Impact

Users would pay lesser APR for some to-be supported borrowable assets.
### Recommended Mitigation Steps

Since disallowing rebasing assets is not an option, either track the balance change for these assets or heavily document this behaviour to users.

## [04] Borrowers would lose a lot of funds if market is intentionally/unintentionally frequently updated

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

## [05] Deposits are broken in an edge case

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

## [06] Releasing escrowed funds might silently fail

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

This function is used to release the escrowed balance of an address, issue however is that it does not return any value on failure/success. This would cause external integrators to not be able to track the success status.

### Impact

QA, however if this silently fails, this is even more problematic as a faux event would be emitted.

### Recommended Mitigation Steps

Track the success of releasing the escrowed funds.

## [07] Apply some sort of AC to `WildcatMarketWithdrawals#executeWithdrawal()`

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

## [08] Setters don't have equality checkers 

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

These functions are used to set/unset the credential status of a lender, issue however is that they never check to ensure the value being set is not what's already stored.

### Impact

QA

### Recommended Mitigation Steps

Apply equality checkers.

## [09] `transferFrom` should only use allowance when  `spender != from`

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

QA, because currently if `sender==from` when calling transferFrom function, it will also deduct allowances, which should not be deducted.

> NB: In many protocols only transferFrom is used, this design will break the compatibility with many protocols.

### Recommended Mitigation Steps

Only use allowance when `spender != from`.

## [10] Total supply would be out of sync for WMTokens

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

## [11] Some assets can't be used as underlying 

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

## [12] Valid deposits would be bricked in an edge case

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

This implementation leads to unexpected reverts even when partial deposits are successfully processed, i.e deposits that are exactly = to the maximum deposits.

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

## [13] Market Settlements can be bricked

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

## [14] Unauthorized actors can sidestep their restrictions and still integrate with Wildcat

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

## [15] A borrower can remove their escrow address via `removeSanctionOverride()`

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

## [16] Admin could intentionally/unintentionally front-run borrower with the fee increasing 

### Proof of Concept

Protocol allows the borrower to deploy a new market and pay a fee to the protocol if it exists. This fee could be changed at any moment by admin.

Admin can accidentally/intentionally front-run borrowers the call to make a new market and set fee to bigger value, which borrower isn't expecting.

### Impact

QA, _Admin window_.

### Recommended Mitigation Steps

Consider adding a timelock to change fee parameters. This way, frontrunning will be impossible, and borrowers will know which fee they agree to.

## [17] Market list can easily get bloated

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

## [18] Potential Limitations of OpenZeppelin's EnumerableSet Usage

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

## [19] Import declarations should import specific identifiers, rather than the whole file

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

## [20] Credentials would no longer be grant-able after a while

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

Because in that case credentials can only be expired since no timestamp is considered after that, see https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/types/LenderStatus.sol#L29-L35

```solidity
  function credentialExpired(
    LenderStatus memory status,
    RoleProvider provider
  ) internal view returns (bool) {
    return provider.calculateExpiry(status.lastApprovalTimestamp) < block.timestamp;
  }

```

## [21] `hasCredential` does not factor in expiration

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

## [22] Remove linter errors from code

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

Make the line length less than 159.

## [23] PUSH0 Opcode is not supported on all to-deploy chains

### Proof of Concept

Per the readMe protocol is to also deploy on multiple optimistic chains see:<br>
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

**[laurenceday (Wildcat) acknowledged and commented](https://github.com/code-423n4/2024-08-wildcat-findings/issues/119#issuecomment-2365291876):**
 > Thank you for the effort involved in putting together this report. There's only one change we'll make here, a docs update based on number 05. The rest is either expected behaviour or things we aren't interested in fixing.
> 
> 1. This is expected behaviour: we don't envisage anyone ever actually creating a market with a 100% reserve ratio, but at the same time it's similarly unlikely that anyone would create one with 90%, so the boundary is somewhat illusory. If someone _did_ do the former, they'd be expected to immediately transfer some assets in upon first deposit to account for protocol fees anyway (indeed, they'd be required to): perhaps if a borrower wanted a vanity wrapped token at 0% APR, for example.
> 
> 2. We're not too fussed about this taking place (it's extremely unlikely that feeRecipient would get blocked by stablecoin issuers/WETH/cbBTC), and if someone created a market for a memecoin that blocked the address, that's just too bad for Wildcat really. One would anticipate that the market wouldn't react well to a token that made a decision like that in any event, so it's not as if the protocol would likely stand to lose much by way of revenue. We want to keep `feeRecipient` immutable, and a pull pattern would need a whitelisted set of addresses anyway that we don't want to have update power for.
> 
> 3. Rebasing tokens are explicitly mentioned in the audit repo and whitepaper for V1 as breaking the underlying interest model, and should not be used as underlying assets (Wildcat is likely to blacklist a few of the most popular ones just to hammer the point home). 
> 
>4. The impact on frequent updating is not nearly as severe as you might imagine - we are well aware of this, however: it is a design choice. (See image [here](https://github.com/user-attachments/assets/8b9363c1-9588-4b4b-8f67-f1d5f0c48772))
> 
> 5. We'll update the docs in time, cheers.
> 
> 6.  It is _extremely_ unlikely that an external protocol would want to integrate the result of a sanctioned escrow account being activated, but fair observation.
> 
> 7. Expected behaviour that we don't wish to fix: the point here is to allow sentinels or borrowers to execute things to keep the withdrawal queue clear. We can't reasonably account for things like wallet upgrades. We _can_ see a lender getting confused and angry that their funds aren't in their wallet because they forgot to execute a queued withdrawal themselves, and it'll just be easier for someone else to sweep things to them.
> 
> 8. Not a concern.
> 
> 9. Not a concern.
> 
> 10. Burning a claim on credit is a mad thing to do (read: we don't expect this to happen, and if it does, good luck to the lender). Adding this check would add unreasonable gas bloat to every single transfer.
> 
> 11. ERC-20s used as underlyings are expected to be well-formed/'standard'. Esoteric tokens such as this aren't a concern of ours.
> 
> 12. We want this in place: it's our observed experience that capped vaults don't have people trying to push right up to the wei, and if they're interested in doing so they should just be using `depositUpTo` instead of `deposit`. Our UI handles this.
> 
> 13. Not expecting this to happen: if it does, the borrower can transfer in from another address using `repay` as needed. More generally, the potential to update the borrower address would be a significant attack vector, and as such we made the early decision to not enable this.
> 
> 14. Market controller factories are deprecated in V2. Even if they weren't, the archcontroller owners would quite simply not be deploying MCFs that were owned by someone else except in the case of a mistake, and any borrower that tried to use them would find that they weren't registered with that new archcontroller address so couldn't deploy anyway.
> 
> 15. This may be precisely what is desired in the event that a borrower is targeted by a hostile Chainalysis oracle, and it's not as if we can distinguish between a 'real' and a 'fake' sanction. Besides, a borrower wouldn't have any market tokens to send into a sentinel escrow contract anyway: they aren't using withdrawals as lenders do.
> 
> 16. Not a concern: fees are read directly into the UI from a lens contract, and even if this happened, a borrower that disagreed would be free to terminate their market as soon as they noticed for minimal 'damage'.
> 
> 17. Expected behaviour - each market is its' own token contract as well, and allowing the 'resetting' of these would be extremely dangerous.
> 
> 18. No response needed.
> 
> 19. No response needed: compilation speed is not a concern of ours for this.
> 
> 20. Dawg, that's eighty years from now. I think we'll be fine. My grandchildren will not be inheriting a Wildcat market.
> 
> 21. Expired credentials are still credentials, and this is used for withdrawal permissions. Unset credentials (i.e. explicitly rescinded) are set to 0. This is expected behaviour.
> 
> 22. Not a concern.
> 
> 23. Not a concern: we'll deploy on chains once PUSH0 is supported.


***

# Disclosures

C4 is an open organization governed by participants in the community.

C4 audits incentivize the discovery of exploits, vulnerabilities, and bugs in smart contracts. Security researchers are rewarded at an increasing rate for finding higher-risk issues. Audit submissions are judged by a knowledgeable security researcher and solidity developer and disclosed to sponsoring developers. C4 does not conduct formal verification regarding the provided code but instead provides final verification.

C4 does not provide any guarantee or warranty regarding the security of this project. All smart contract software should be used at the sole risk and responsibility of users.

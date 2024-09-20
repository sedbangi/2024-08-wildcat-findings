## Impact
After a borrower revokes credentials for the lender, the lender can call the hook directly to re-grant himself credentials to the market as long as any provider has still granted him valid credentials.

No real impact or use of credentials at the moment, but logically if the borrower revokes credentials for a lender, I would think they would assume that they are actually revoked.

`https://github.com/code-423n4/2024-08-wildcat/blob/fe746cc0fbedc4447a981a50e6ba4c95f98b9fe1/src/access/AccessControlHooks.sol#L812`

## Proof of Concept

Add the following test to `WildcatMarketToken.t.sol`:
```
  function testRegainCreds() external {
    parameters.hooksConfig = parameters.hooksConfig.setFlag(Bit_Enabled_Transfer);
    BaseMarketTest.setUpContracts(true);
    token = IERC20(address(market));
    MarketState memory state;
    _mint(bob, 1 ether);
    _mint(alice, 1 ether);
    //bob's credentials removed & blocked from depositing
    _blockLender(bob);
    vm.prank(bob);
    //bob calls the hook directly and instantly regains credentials (but is still blocked from depositing)
    hooks.onQueueWithdrawal(bob,0,0,state,'');
  }
```

## Recommended Mitigation Steps
Clarify access control flow on revocation.
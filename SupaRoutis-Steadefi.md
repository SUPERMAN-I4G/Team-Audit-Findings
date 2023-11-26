# Steadefi - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## Medium Risk Findings
    - ### [M-01. Token Swap and `Repay` Revert Risk in `EmergencyClose` and `processWithdraw` Functions](#M-01)
    - ### [M-02. `addLiquidity` && `removeLiquidity` in `emergencyResume` and `emergencyPause` are prone to sandwich attacks](#M-02)
    - ### [M-03. `additionalCapacity()` function uses same weights for the maximum borrowable amounts in both tokens can overestimate/underestimate the amounts](#M-03)
    - ### [M-04. If an address gets Blacklisted by any asset tokens, there can be loss of funds](#M-04)
    - ### [M-05. `emergencyPause` does not check the state before running && can cause loss of funds for users](#M-05)

- ## Low Risk Findings
    - ### [L-01. Asset like UNI can revert on Large Approvals & Transfers](#L-01)
    - ### [L-02. `ChainlinkARBOracle` and `ChainlinkOracle` do not check for stale prices](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Steadefi

### Dates: Oct 26th, 2023 - Nov 6th, 2023

[See more contest details here](https://www.codehawks.com/contests/clo38mm260001la08daw5cbuf)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - Medium: 5
   - Low: 2
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Token Swap and `Repay` Revert Risk in `EmergencyClose` and `processWithdraw` Functions            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXProcessWithdraw.sol#L40

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXEmergency.sol#L136

## Summary
The identified vulnerability is related to two instances of token swapping within the functions `emergencyClose` and `processWithdraw`. These functions use the balance of the contract as the `amountIn` for the swaps, potentially resulting in swapped amounts that exceed the available amount the vault can repay. This can lead to a revert in the subsequent `repay` function.

## Vulnerability Details

1st Instance - `emergencyClose`
In the `emergencyClose` function, the vulnerability arises when the system checks if a swap is needed (before repay) and then proceeds to swap tokens.

```javascript

if (_swapNeeded) {
  ISwap.SwapParams memory _sp;
  _sp.tokenIn = _tokenFrom;
  _sp.tokenOut = _tokenTo;
  _sp.amountIn = IERC20(_tokenFrom).balanceOf(address(this)); // @audit might use more tokens for swap which will result in repay reverting because it tries to repay more than what the contract has
  _sp.amountOut = _tokenToAmt;
  _sp.slippage = self.minSlippage;
  _sp.deadline = deadline;
  GMXManager.swapTokensForExactTokens(self, _sp);
}
GMXManager.repay(self, _rp.repayTokenAAmt, _rp.repayTokenBAmt);

```

2nd Instance - "processWithdraw"
The same vulnerability pattern occurs in the `processWithdraw` function. 

```javascript
if (_swapNeeded) {
  ISwap.SwapParams memory _sp;
  _sp.tokenIn = _tokenFrom;
  _sp.tokenOut = _tokenTo;
  _sp.amountIn = IERC20(_tokenFrom).balanceOf(address(this)); // @audit might use more tokens for swap which will result in repay reverting because it tries to repay more than what the contract has
  _sp.amountOut = _tokenToAmt;
  _sp.slippage = self.minSlippage;
  _sp.deadline = block.timestamp;
  GMXManager.swapTokensForExactTokens(self, _sp);
}
```
If the amountIn used for the swap are more than expected, the resulting amounts of tokenA/tokenB might less be than what the repay function expects resulting in a revert.

```javascript
    GMXManager.repay(
      self,
      _rp.repayTokenAAmt,
      _rp.repayTokenBAmt
    );
```

MEV bots can also exploit this scenario to extract as much `amountIn` as possible.

## Impact
The impact of this vulnerability is two-fold:

1. The usage of the contract's balance as the "amountIn" for token swaps may lead to swapped amounts that exceed the available amount the vault can repay. This can result in the subsequent "repay" function reverting, leading to tx failures and DOS.
2. loss of opportunities for users

2. Possible loss of funds for the Vault since emergencyClose had to be executed to prevent against an extreme scenario.

## Proof of Concept
Consider this scenario;

A. Assuming the keeper initiates a "EmergencyClose" operation due to one or two crucial reasons 

B. The function determines whether token A or B swap is required before processing repay.

C. The swap function uses the entire contract balance as "amountIn" for the swap.

D. Due to market volatility, the swap results in an amount larger than the amount the vault can repay.

E. The subsequent "repay" function fails to execute, resulting in a revert of the transaction i.e EmergencyClose reverts and defeats the purpose of the emergency function whose goal is to be able to remove liquidity to try to protect against any extreme scenarios (leading to financial losses)

More info:
https://x.com/puputhrashing/status/1454030019223719937?s=46&t=ahuBu4vx0GHQr2UGnKTzKA

## Tools Used
Manual

## Recommendations
To mitigate this in short term this i would recommend Validating the Swap Amounts. Implement check to ensure that the `amountIn` used for swaps does not exceed the amount the vault can repay. 
## <a id='M-02'></a>M-02. `addLiquidity` && `removeLiquidity` in `emergencyResume` and `emergencyPause` are prone to sandwich attacks            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXEmergency.sol#L52-L61

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXEmergency.sol#L81-L90

## Summary

addLiquidity && removeLiquidity in the `emergencyResume` and `emergencyPause` functions do not check the minimum liquidity amounts. This makes vault funds vulnerable to sandwich attacks.

## Vulnerability Details

During an emergency pause, the `removeLiquidity` function is called without specifying the minimum amount for long and short tokens that must be received (minLongTokenAmount and minShortTokenAmount). This exposes the transaction to potential sandwich attacks, where an attacker can manipulate the market price by performing trades before and after the transaction. The same vulnerability is present in the `emergencyResume` function with `addLiquidity`, where liquidity is added back without checking for minimum amounts, making it possible for an attacker to extract value.

Exemple :
```javascript
  function emergencyPause(
    GMXTypes.Store storage self
  ) external {
    self.refundee = payable(msg.sender);

    GMXTypes.RemoveLiquidityParams memory _rlp;

    // Remove all of the vault's LP tokens
    _rlp.lpAmt = self.lpToken.balanceOf(address(this));
    _rlp.executionFee = msg.value;

    GMXManager.removeLiquidity(
      self,
@>      _rlp
    );
```
Since `minTokenAAmt` and `minTokenBAmt` are not defined, the `_rlp.minTokenBAmt` and `_rlp.minTokenAAmt` have the default value 0. Which makes this transaction vulnerable to MEV Bots. Same thing for for the `emergencyResume` function where `minMarketTokenAmt` is not defined for addingLiquidity.
## Impact
High. The lack of minimum amount checks can lead to significant financial loss as attackers can drain the value from the transactions.

## Tools Used
Manual Review
## Recommendations
Define and enforce minimum amounts for long, short tokens and LP tokens as function parameters when calling removeLiquidity and addLiquidity during emergency procedures to prevent exploitation through sandwich attacks.
## <a id='M-03'></a>M-03. `additionalCapacity()` function uses same weights for the maximum borrowable amounts in both tokens can overestimate/underestimate the amounts            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXReader.sol#L264-L270

## Summary

The `additionalCapacity()` function has been identified to potentially lead to inaccuracies in calculating the maximum borrowable amounts of assets due to the use of identical weights for different tokens.

## Vulnerability Details

In the lending protocol, the additionalCapacity() function is responsible for determining the total amounts that users are allowed to borrow depending on the weights of the tokens in the GMX liquidity Pool and leverage.

However, it does not account for the weights of tokenB and uses tokenA's weights for the calculation of this amount. 

```javascript
uint256 _maxTokenBLending = convertToUsdValue(
        self,
        address(self.tokenB),
        self.tokenBLendingVault.totalAvailableAsset()
      ) * SAFE_MULTIPLIER
@>        / (self.leverage * _tokenAWeight / SAFE_MULTIPLIER)
        - 1e18;
```
Also, the use of the magic number -1e18 seems unclear here. It seems like it is been used as a provision.
## Impact

The best case scenario is that this amount is underestimated which will just slow down the lending. But if the amount is overestimated then the following check will pass :

```javascript
  function beforeDepositChecks(
    GMXTypes.Store storage self,
    uint256 depositValue
  ) external view {
    if (self.status != GMXTypes.Status.Open)
      revert Errors.NotAllowedInCurrentVaultStatus();

    if (self.depositCache.depositParams.executionFee < self.minExecutionFee)
      revert Errors.InsufficientExecutionFeeAmount();

    if (!self.vault.isTokenWhitelisted(self.depositCache.depositParams.token))
      revert Errors.InvalidDepositToken();

    if (self.depositCache.depositParams.amt == 0)
      revert Errors.InsufficientDepositAmount();

    if (self.depositCache.depositParams.slippage < self.minSlippage)
      revert Errors.InsufficientSlippageAmount();

    if (depositValue == 0)
      revert Errors.InsufficientDepositAmount(); // @audit redundant check

    if (depositValue < MINIMUM_VALUE)
      revert Errors.InsufficientDepositAmount();

@>    if (depositValue > GMXReader.additionalCapacity(self)) 
      revert Errors.InsufficientLendingLiquidity();
  }
```


## Tools Used

Manual review

## Recommendations

```diff

uint256 _maxTokenBLending = convertToUsdValue(
        self,
        address(self.tokenB),
        self.tokenBLendingVault.totalAvailableAsset()
      ) * SAFE_MULTIPLIER
-      / (self.leverage * _tokenAWeight / SAFE_MULTIPLIER)
        - 1e18;

uint256 _maxTokenBLending = convertToUsdValue(
        self,
        address(self.tokenB),
        self.tokenBLendingVault.totalAvailableAsset()
      ) * SAFE_MULTIPLIER
+        / (self.leverage * _tokenBWeight / SAFE_MULTIPLIER)
        - 1e18;
```
## <a id='M-04'></a>M-04. If an address gets Blacklisted by any asset tokens, there can be loss of funds            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXWorker.sol#L28-L30

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXWorker.sol#L77-L79

## Summary

There exists a potential risk of fund loss if critical addresses involved in the protocol, such as the `Trove`, `depositVault`, `withdrawalVault`, or the admin/owner addresses, are blacklisted by any of the asset tokens like USDC used within the system.

## Vulnerability Details

Asset tokens like USDC might have built-in blacklisting capabilities that can restrict transactions from certain addresses. If critical system addresses are blacklisted, it may result in the inability to execute transactions involving these tokens like depositing/withdrawing/compounding rewards... Since smart contracts cannot react to or mitigate the effects of being blacklisted post-facto, this could lead to a situation where funds are effectively stuck without any recourse.

The depositor can still chose the token we wants to withdraw in, but loses amount equal to EXECUTION_FEE if he withdraws in a token where his address is blacklisted.

## Impact

The impact of such blacklisting could be severe:

- Operational Disruption: The protocol's normal operations, such as deposits, withdrawals, and internal compounding/rebalancing, could be halted.
- Loss of Funds: Users might lose access to their funds if they are held in addresses that are blacklisted.

## Tools Used

Manual review

## Recommendations

Allow every address used by the Vault to be updatable by a dedicated admin/Owner. For now, only the `Trove` address is updatable. 
Implement a multisig for owner access

## <a id='M-05'></a>M-05. `emergencyPause` does not check the state before running && can cause loss of funds for users            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXEmergency.sol#L47

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXEmergency.sol#L63

## Summary
The `emergencyPause` function in the GMX smart contract can be called by the keeper at any time without pre-transaction checks. In some cases this could result in financial loss for users if the function is executed before the callbacks have executed.

## Vulnerability Details
The emergencyPause function lacks a control mechanism to prevent execution before callbacks execution. While it is designed to halt all contract activities in an emergency, its unrestricted execution could disrupt ongoing transactions. For example, if a user calls a function like deposit which involves multiple steps and expects a callback, and emergencyPause is invoked before the callback is executed, the user might lose his funds as he will not be able to mint svTokens.

Since `emergencyPause` updates the state of the Vault to `GMXTypes.Status.Paused`, when the callback from GMX executes the `afterDepositExecution` nothing will happen since the conditions are not met. Which means that any deposit amount will not be met by a mint of svTokens.

```javascript
  function afterDepositExecution(
    bytes32 depositKey,
    IDeposit.Props memory /* depositProps */,
    IEvent.Props memory /* eventData */
  ) external onlyController {
    GMXTypes.Store memory _store = vault.store();

    if (
      _store.status == GMXTypes.Status.Deposit &&
      _store.depositCache.depositKey == depositKey
    ) {
      vault.processDeposit();
    } else if (
      _store.status == GMXTypes.Status.Rebalance_Add &&
      _store.rebalanceCache.depositKey == depositKey
    ) {
      vault.processRebalanceAdd();
    } else if (
      _store.status == GMXTypes.Status.Compound &&
      _store.compoundCache.depositKey == depositKey
    ) {
      vault.processCompound();
    } else if (
      _store.status == GMXTypes.Status.Withdraw_Failed &&
      _store.withdrawCache.depositKey == depositKey
    ) {
      vault.processWithdrawFailureLiquidityAdded();
    } else if (_store.status == GMXTypes.Status.Resume) {
      // This if block is to catch the Deposit callback after an
      // emergencyResume() to set the vault status to Open
      vault.processEmergencyResume();
    }
    

@ > // The function does nothing as the conditions are not met
  }
```

If by any chance, the `processDeposit` function is executed (or any other function from the callback) it will still revert in the beforeChecks (like the `beforeProcessDepositChecks`).
```javascript
  function beforeProcessDepositChecks(
    GMXTypes.Store storage self
  ) external view {
    if (self.status != GMXTypes.Status.Deposit)
@>      revert Errors.NotAllowedInCurrentVaultStatus();
  }
```

## Impact
If the emergency pause is triggered at an inopportune time, it could:
- Prevent the completion of in-progress transactions.
- Lead to loss of funds if the transactions are not properly rolled back.
- Erode user trust in the system due to potential for funds to be stuck without recourse.

### POC : 
You can copy this test in the file GMXEmergencyTest.t.sol then execute the test with the command forge test --mt

```javascript
  function test_UserLosesFundsAfterEmergencyPause() external {
    deal(address(WETH), user1, 20 ether);
    uint256 wethBalanceBefore = IERC20(WETH).balanceOf(user1);
    vm.startPrank(user1);
    _createDeposit(address(WETH), 10e18, 1, SLIPPAGE, EXECUTION_FEE);
     vm.stopPrank();

    vm.prank(owner);
    vault.emergencyPause();

    vm.prank(user1);
    mockExchangeRouter.executeDeposit(
      address(WETH),
      address(USDC),
      address(vault),
      address(callback)
    );
    uint256 wethBalanceAfter = IERC20(WETH).balanceOf(user1);
    //Check that no tokens have been minted to user while user loses funds = 10 eth
    assertEq(IERC20(vault).balanceOf(user1), 0);
    assertEq(wethBalanceAfter, wethBalanceBefore - 10 ether);

  }
```

## Tools Used
Manual review
## Recommendations
To mitigate this risk, the following recommendations should be implemented:

- Introduce a state check mechanism that prevents emergencyPause from executing if there are pending critical operations that must be completed to ensure the integrity of in-progress transactions.
- Implement a secure check that allows emergencyPause to queue behind critical operations, ensuring that any ongoing transaction can complete before the pause takes effect.

# Low Risk Findings

## <a id='L-01'></a>L-01. Asset like UNI can revert on Large Approvals & Transfers            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXVault.sol#L118C8-L119

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXVault.sol#L122-L123

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXVault.sol#L127-L128

## Summary

As per the protocol documentation, assets like UNI are to be used. However, these types of asset are programmed to revert transactions that involve large approvals and transfers. 

## Vulnerability Details

According to the documentation https://github.com/d-xo/weird-erc20. UNI reverts if the value passed to approve or transfer is larger than uint96. When constructing the Vault, many large approves are made :

```javascript
    _store.tokenA.approve(address(_store.router), type(uint256).max);
    _store.tokenB.approve(address(_store.router), type(uint256).max);
    _store.lpToken.approve(address(_store.router), type(uint256).max);

    _store.tokenA.approve(address(_store.depositVault), type(uint256).max);
    _store.tokenB.approve(address(_store.depositVault), type(uint256).max);

    _store.lpToken.approve(address(_store.withdrawalVault), type(uint256).max);

    _store.tokenA.approve(address(_store.tokenALendingVault), type(uint256).max);
    _store.tokenB.approve(address(_store.tokenBLendingVault), type(uint256).max);
```
There can also be large transfer amount at one point, for example if an emergencyPause, emergencyClose or emergencyResume happen.
## Impact
At the very least disrupting the contract creation or worse blocking the vault when large transfers happen.
## Tools Used

Manual review

## Recommendations
- limit max transfer/approve amounts to type(uint96).max
## <a id='L-02'></a>L-02. `ChainlinkARBOracle` and `ChainlinkOracle` do not check for stale prices            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/oracles/ChainlinkARBOracle.sol#L193

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/oracles/ChainlinkOracle.sol#L161

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/oracles/ChainlinkARBOracle.sol#L111

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/oracles/ChainlinkOracle.sol#L98

## Summary
`_badChainlinkResponse()` function in both the `ChainlinkARBOracle` and `ChainlinkOracle` contracts are missing the stale data check.

## Vulnerability Details
None of the oracle calls check for stale prices, for example

```javascript
    (
      uint80 _latestRoundId,
      int256 _latestAnswer,
      /* uint256 _startedAt */,
      uint256 _latestTimestamp,
@>      /* uint80 _answeredInRound */
    ) = AggregatorV3Interface(_feed).latestRoundData();

  function _badChainlinkResponse(ChainlinkResponse memory response) internal view returns (bool) {
    // Check for response call reverted
    if (!response.success) { return true; }
    // Check for an invalid roundId that is 0
    if (response.roundId == 0) { return true; }
    // Check for an invalid timeStamp that is 0, or in the future
    if (response.timestamp == 0 || response.timestamp > block.timestamp) { return true; }
    // Check for non-positive price
    if (response.answer == 0) { return true; }

    return false;
  }

```


## Impact
Oracle data feeds can return stale pricing data for a variety of reasons. If the returned pricing data is stale, the `getChainlinkResponse` function will execute with prices that don’t reflect the current pricing resulting in a potential loss of funds due to incorrect calculations.

## Tools Used
Manual 

## Recommendations
Read the _answeredInRound parameter from the calls to latestRoundData() and consider implementing the following changes in both `ChainlinkARBOracle` and `ChainlinkOracle` contracts :

function `_getChainlinkResponse` :
```diff

    (
      uint80 _latestRoundId,
      int256 _latestAnswer,
      /* uint256 _startedAt */,
      uint256 _latestTimestamp,
-      /* uint80 _answeredInRound */
    ) = AggregatorV3Interface(_feed).latestRoundData();

    (
      uint80 _latestRoundId,
      int256 _latestAnswer,
      /* uint256 _startedAt */,
      uint256 _latestTimestamp,
+      /* uint80 _answeredInRound */
    ) = AggregatorV3Interface(_feed).latestRoundData();

```

function `_badChainlinkResponse` :

```diff
  function _badChainlinkResponse(ChainlinkResponse memory response) internal view returns (bool) {
    // Check for response call reverted
    if (!response.success) { return true; }
    // Check for an invalid roundId that is 0
    if (response.roundId == 0) { return true; }
    // Check for an invalid timeStamp that is 0, or in the future
    if (response.timestamp == 0 || response.timestamp > block.timestamp) { return true; }
    // Check for non-positive price
    if (response.answer == 0) { return true; }
     // Check for stale price
+   if (response.answeredInRound >= response.roundId) { return true; }
    return false;
  }



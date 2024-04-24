# Axis Finance 

Type: CONTEST

Dates: 18 Mar 2024 - 30 Mar 2024

Axis is a modular auction protocol. It supports abstract atomic or batch auction formats, which can be added to the central auction house as modules. Additionally, it allows creating and auctioning derivatives of the base asset in addition to spot tokens

Code Under Review: https://github.com/sherlock-audit/2024-03-axis-finance

More contest details: https://audits.sherlock.xyz/contests/206

Rank: 2

# Findings Summary

- High : 7
- Medium : 4


|ID|Title|Severity|
|-|:-|:-:|
| [H-01](#H-01)| Lack of max field value check for coordinates allows bricking the decryption | High |
| [H-02](#H-02)| Downcasting to uint96 can cause assets to be lost for some tokens | High |
| [H-03](#H-03)| Incorrect prefundingRefund calculation will disallow claiming | High |
| [H-04](#H-04)| Overrly restrictive check for claimBid function disallows bidder's from claiming | High |
| [H-05](#H-05)| Gas is not configured to be claimable in Blast | High |
| [H-06](#H-06)| Inconsistent timestamp usage across `_revertIfLotActive` and `_revertIfLotConcluded` | High |
| [H-07](#H-07)| Bidder's payout claim will fail due to validation checks in LinearVesting | High |
| [M-01](#M-01)| Remaining funds of FMAP auctions cannot be recovered once auction is concluded | Medium |
| [M-02](#M-02)| Lot id is always set to 0 for new auctions | Medium |
| [M-03](#M-03)| Inaccurate value is used for partial fill quote amount when calculating fees | Medium |
| [M-04](#M-04)| User's can be griefed by not submitting the private key | Medium |


# Detailed Findings

# <a id='H-01'></a>H-01 Lack of max field value check for coordinates allows bricking the decryption

## Vulnerability Detail

In EMPAM, all the bids should be decrypted in order to settle the auction. In case the decryption of any bid reverts, auction cannot be settled and the entire assets (assets of bids + auction asset) is forever locked. The decryption involves performing elliptic curve scalar multiplication by making a call to the ecMul precompile (address 0x7).
To avoid any reverts, the bidder's input is validated at the time of bid creation. But the validation of public key is incomplete and allows an attacker to pass in a public key with co-ordinates greater than the field modulus. 

```solidity
    function _bid(
        uint96 lotId_,
        address bidder_,
        address referrer_,
        uint96 amount_,
        bytes calldata auctionData_
    ) internal override returns (uint64 bidId) {
        
        ....

        if (!ECIES.isValid(bidPubKey)) revert Auction_InvalidKey();
```

```solidity
    function isOnBn128(Point memory p) public pure returns (bool) {
        // check if the provided point is on the bn128 curve y**2 = x**3 + 3, which has generator point (1, 2)
        return _fieldmul(p.y, p.y) == _fieldadd(_fieldmul(p.x, _fieldmul(p.x, p.x)), 3);
    }


    // @audit doesn't check if the coordinates are greater than the FIELD_MODULUS
    function isValid(Point memory p) public pure returns (bool) {
        return isOnBn128(p) && !(p.x == 1 && p.y == 2) && !(p.x == 0 && p.y == 0);
    }


    function _fieldmul(uint256 a, uint256 b) private pure returns (uint256 c) {
        assembly {
            c := mulmod(a, b, FIELD_MODULUS)
        }
    }


    function _fieldadd(uint256 a, uint256 b) private pure returns (uint256 c) {
        assembly {
            c := addmod(a, b, FIELD_MODULUS)
        }
    }
```

This will cause the call to ecMul precompile to [revert](https://eips.ethereum.org/EIPS/eip-196#exact-semantics) at the time of decryption as coordinates greater than field modulus are considered invalid inputs

### POC
Add the following test to `test/lib/ECIES/decrypt.t.sol` and run `forge test --mt testHash_InvalidPubKeyAccepted`
It is asserted that the malicious pubkey is accepted by the library but will fail during decryption

```solidity
    function testHash_InvalidPubKeyAccepted() public {

        // point with coordinate greater than field modulus
        Point memory maliciousPubKey = Point({
            x:21888242871839275222246405745257275088696311157297823662689037894645226208584,y:2
        });        
        assert(maliciousPubKey.x >= ECIES.FIELD_MODULUS);

        // pubkey is considered valid by the lib
        bool valid = ECIES.isValid(maliciousPubKey);
        assert(valid);
   
        // but during decryption, will revert since ecMul revert on coordinates >= ECIES.FIELD_MODULUS, hence bricking the auction

        uint256 ciphertext = 0xf96d7675ae04b89c9b5a9b0613d3530bb939186d05959efba9b3249a461abbc4;
        uint256 recipientPrivateKey = 2;
        uint256 salt = 1;

        vm.expectRevert();
        ECIES.decrypt(ciphertext, maliciousPubKey, recipientPrivateKey, salt);
    }
```

## Impact

The decryption can be bricked leading to lost assets for the auctioner and bidder's

## Code Snippet

incomplete validation in ECIES library
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/lib/ECIES.sol#L131-L151

## Tool used

Manual Review

## Recommendation

Validate that both the coordinates are less than FIELD_MODULUS

# <a id='H-02'></a>H-02 Downcasting to uint96 can cause assets to be lost for some tokens

## Vulnerability Detail

After summing the individual bid amounts, the total bid amount is downcasted to uint96 without any checks

```solidity
            settlement_.totalIn = uint96(result.totalAmountIn);
```

uint96 can be overflowed for multiple well traded tokens:

Eg:

shiba inu :
current price = $0.00003058
value of type(uint96).max tokens ~= 2^96 * 0.00003058 / 10^18 == 2.5 million $

Hence auctions that receive more than type(uint96).max amount of tokens will be downcasted leading to extreme loss for the auctioner

## Impact

The auctioner will suffer extreme loss in situations where the auctions bring in >uint96 amount of tokens

## Code Snippet

downcasting totalAmountIn to uint96
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/auctions/EMPAM.sol#L825

## Tool used

Manual Review

## Recommendation

Use a higher type or warn the user's of the limitations on the auction sizes

# <a id='H-03'></a>H-03 Incorrect prefundingRefund calculation will disallow claiming

## Vulnerability Detail

The `prefundingRefund` variable calculation inside the `claimProceeds` function is incorrect

```solidity
    function claimProceeds(
        uint96 lotId_,
        bytes calldata callbackData_
    ) external override nonReentrant {
        
        ...

        (uint96 purchased_, uint96 sold_, uint96 payoutSent_) =
            _getModuleForId(lotId_).claimProceeds(lotId_);

        ....

        // Refund any unused capacity and curator fees to the address dictated by the callbacks address
        // By this stage, a partial payout (if applicable) and curator fees have been paid, leaving only the payout amount (`totalOut`) remaining.
        uint96 prefundingRefund = routing.funding + payoutSent_ - sold_;
        unchecked {
            routing.funding -= prefundingRefund;
        }
```

Here `sold` is the total base quantity that has been sold to the bidders. Unlike required, the `routing.funding` variable need not be holding `capacity + (0,curator fees)` since it is decremented every time a payout of a bid is claimed

```solidity
    function claimBids(uint96 lotId_, uint64[] calldata bidIds_) external override nonReentrant {
        
        ....

            if (bidClaim.payout > 0) {
 
                ...

                // Reduce funding by the payout amount
                unchecked {
                    routing.funding -= bidClaim.payout;
                }
```

### Example
Capacity = 100 prefunded, hence routing.funding == 100 initially
Sold = 90 and no partial fill/curation
All bidders claim before the claimProceed function is invoked
Hence routing.funding = 100 - 90 == 10
When claimProceeds is invoked, underflow and revert:

uint96 prefundingRefund = routing.funding + payoutSent_ - sold_ == 10 + 0 - 90

## Impact

Claim proceeds function is broken. Sellers won't be able to receive the proceedings

## Code Snippet

wrong calculation
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/AuctionHouse.sol#L604

## Tool used

Manual Review

## Recommendation

Change the calculation to:
```solidity
uint96 prefundingRefund = capacity - sold_ + curatorFeesAdjustment (how much was prefunded initially - how much will be sent out based on capacity - sold)
```

# <a id='H-04'></a>H-04 Overrly restrictive check for claimBid function disallows bidder's from claiming

## Vulnerability Detail

The claimBids function internally calls `_revertIfLotNotSettled` which will revert in case the auction status is not ``
```solidity
    function claimBids(
        uint96 lotId_,
        uint64[] calldata bidIds_
    )
        external
        override
        onlyInternal
        returns (BidClaim[] memory bidClaims, bytes memory auctionOutput)
    {
        // Standard validation
        _revertIfLotInvalid(lotId_);
        _revertIfLotNotSettled(lotId_);


        // Call implementation-specific logic
        return _claimBids(lotId_, bidIds_);
```

```solidity
    function _revertIfLotNotSettled(uint96 lotId_) internal view override {
        // Auction must be settled
        if (auctionData[lotId_].status != Auction.Status.Settled) {
            revert Auction_WrongState(lotId_);
        }
```

This is overrly restrictive as the status can be changed to `Claimed` when the seller claims the proceeds and the bidders should be allowed to claim their payouts even after

```solidity
    function _claimProceeds(uint96 lotId_)
        internal
        override
        returns (uint96 purchased, uint96 sold, uint96 payoutSent)
    {
        // Update the status
        auctionData[lotId_].status = Auction.Status.Claimed;


        // Get the lot data
        Lot memory lot = lotData[lotId_];


        // Return the required data
        return (lot.purchased, lot.sold, lot.partialPayout);
    }
```

## Impact

Bidder's won't be able to claim their payouts

## Code Snippet

claimBids reverts if the status is not Settled
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/auctions/EMPAM.sol#L902-L906

claiming proceeds sets status to claimed
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/auctions/EMPAM.sol#L840-L853

## Tool used

Manual Review

## Recommendation

Allow both `Settled` and `Claimed` statuses

# <a id='H-05'></a>H-05 Gas is not configured to be claimable in Blast

## Vulnerability Detail

To obtain the gas rewards in Blast, the modules inherit the BlastGas contract

Eg:

```solidity
contract BlastEMPAM is EncryptedMarginalPriceAuctionModule, BlastGas {

    constructor(address auctionHouse_)
        EncryptedMarginalPriceAuctionModule(auctionHouse_)
        BlastGas(auctionHouse_)
    {}
}
```

```solidity
abstract contract BlastGas {

    constructor(address parent_) {
        IBlast(0x4300000000000000000000000000000000000002).configureGovernor(parent_);
    }
}
```

The idea is to set the parent_ as the governor and then claim the rewards via the parent_. But inorder to claim the gas rewards, one must [first configure the rewards to be claimable](https://docs.blast.io/building/guides/gas-fees#setting-gas-mode-to-claimable). This is not done and hence the rewards cannot be claimed for the modules

## Impact

Lost gas rewards 

## Code Snippet

https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/blast/modules/BlastGas.sol#L8-L15

## Tool used

Manual Review

## Recommendation

Configure the gas to be claimable by invoking the `configureClaimableGas` for the modules

# <a id='H-06'></a>H-06 Inconsistent timestamp usage across `_revertIfLotActive` and `_revertIfLotConcluded`

## Vulnerability Detail

At timestmap `lotData[lotId_].conclusion`, the auction is considered not active (ie.concluded) by `_revertIfLotActive` function while `_revertIfLotConcluded` considers the auction as not concluded

```solidity
    function _revertIfLotActive(uint96 lotId_) internal view override {
        if (
            auctionData[lotId_].status == Auction.Status.Created
                && lotData[lotId_].start <= block.timestamp
                && lotData[lotId_].conclusion > block.timestamp
        ) revert Auction_WrongState(lotId_);
    }
```

```solidity
    function _revertIfLotConcluded(uint96 lotId_) internal view virtual {
        // Beyond the conclusion time
        if (lotData[lotId_].conclusion < uint48(block.timestamp)) {
            revert Auction_MarketNotActive(lotId_);
        }

        // Capacity is sold-out, or cancelled
        if (lotData[lotId_].capacity == 0) revert Auction_MarketNotActive(lotId_);
    }
```

This allows for a variety of attacks since both these functions are used in conjunction to avoid many invalid flows:

### Example
1. User's can refund a bid after decryption: 
```solidity
    function refundBid(
        uint96 lotId_,
        uint64 bidId_,
        address caller_
    ) external override onlyInternal returns (uint96 refund) {
        // Standard validation
        _revertIfLotInvalid(lotId_);
        _revertIfBeforeLotStart(lotId_);
        _revertIfBidInvalid(lotId_, bidId_);
        _revertIfNotBidOwner(lotId_, bidId_, caller_);
        _revertIfBidClaimed(lotId_, bidId_);
=>      _revertIfLotConcluded(lotId_);


        // Call implementation-specific logic
        return _refundBid(lotId_, bidId_, caller_);
    }
```

```solidity
    function submitPrivateKey(uint96 lotId_, uint256 privateKey_, uint64 num_) external {
        // Validation
        _revertIfLotInvalid(lotId_);
=>      _revertIfLotActive(lotId_);
        _revertIfBeforeLotStart(lotId_);

        ....

        // Decrypt and sort bids
        _decryptAndSortBids(lotId_, num_);
    }
```

Since `_revertIfLotActive` is used, it is possible to submit the private key and decrypt the bids at `lotData[lotId_].conclusion`. Following this a user can refund one of the bids that has already been decrypted. This will cause multiple issues like the funds being taken back by the bidder while the same fund will be allocated to the seller leading to lost asset for the seller, bricking of decryption processes by refunding multiple bids such that the bids.length becomes less than nextDecryptIndex etc.

2. Seller can cancel EMPAM making bidders loose their assets:
EMPAM auctions are not supposed to be cancellable once started. But the timestamp issue above allows to cancel auctions at `lotData[lotId_].conclusion` which will disable bidder's from withdrawing their assets (linked with the status == SETTLED vulnerability) or make them have wasted keeping their assets in bid

## Impact

Possible bricked decryption, lost assets for the seller / bidder if a block is produced at lotConclusion

## Code Snippet

submit private key
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/auctions/EMPAM.sol#L400-L420

refundBid
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/Auction.sol#L501-L516

_revertIfLotConcluded
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/Auction.sol#L733-L741

_revertIfLotActive
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/auctions/EMPAM.sol#L885-L891

## Tool used

Manual Review

## Recommendation

Align conclusion timestamps across these functions

# <a id='H-07'></a>H-07 Bidder's payout claim will fail due to validation checks in LinearVesting

## Vulnerability Detail

Bidder's payout are sent by internally calling the `_sendPayout` function. In case the payout is a derivative which has already expired, this will revert due to the validation check of `block.timestmap < expiry` present in the mint function of LinearVesting derivative

```solidity
    function _sendPayout(
        address recipient_,
        uint256 payoutAmount_,
        Routing memory routingParams_,
        bytes memory
    ) internal {
        
        if (fromVeecode(derivativeReference) == bytes7("")) {
            Transfer.transfer(baseToken, recipient_, payoutAmount_, true);
        }
        else {
            
            DerivativeModule module = DerivativeModule(_getModuleIfInstalled(derivativeReference));

            Transfer.approve(baseToken, address(module), payoutAmount_);

=>          module.mint(
                recipient_,
                address(baseToken),
                routingParams_.derivativeParams,
                payoutAmount_,
                routingParams_.wrapDerivative
            );
```

```solidity
    function mint(
        address to_,
        address underlyingToken_,
        bytes memory params_,
        uint256 amount_,
        bool wrapped_
    )
        external
        virtual
        override
        returns (uint256 tokenId_, address wrappedAddress_, uint256 amountCreated_)
    {
        if (amount_ == 0) revert InvalidParams();

        VestingParams memory params = _decodeVestingParams(params_);

        if (_validate(underlyingToken_, params) == false) {
            revert InvalidParams();
        }
```

```solidity
    function _validate(
        address underlyingToken_,
        VestingParams memory data_
    ) internal view returns (bool) {
        
        ....

=>      if (data_.expiry < block.timestamp) return false;


        // Check that the underlying token is not 0
        if (underlyingToken_ == address(0)) return false;


        return true;
    }
```

Hence the user's won't be able to claim their payouts of an auction once the derivative has expired. For EMPAM auctions, a seller can also wait till this timestmap passes before revealing their private key which will disallow bidders from claiming their rewards.

## Impact

Bidder's won't be able claim payouts from auction after the derivative expiry timestamp

## Code Snippet

_sendPayout invoking mint function on derivative to send payouts 
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/AuctionHouse.sol#L823-L829

linear vesting derivative expiry checks
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/derivatives/LinearVesting.sol#L521-L541

## Tool used

Manual Review

## Recommendation

Allow to mint tokens even after expiry of the vesting token / deploy the derivative token first itself and when making the payout, transfer the base token directly incase the expiry time is passed

# <a id='M-01'></a>M-01 Remaining funds of FMAP auctions cannot be recovered once auction is concluded

## Vulnerability Detail

Auctions cannot be cancelled after its conclusion timestamp has passed

```solidity
    function cancelAuction(uint96 lotId_) external override onlyInternal {
        // Validation
        _revertIfLotInvalid(lotId_);
        _revertIfLotConcluded(lotId_);
```

```solidity
    function _revertIfLotConcluded(uint96 lotId_) internal view virtual {
        // Beyond the conclusion time
        if (lotData[lotId_].conclusion < uint48(block.timestamp)) {
            revert Auction_MarketNotActive(lotId_);
        }
```

This is problematic for prefunded FMAP auctions as the only way to recover tokens for such auctions is by cancelling. Hence there would be no way to recover these funds

## Impact

Lost assets for the seller

## Code Snippet

cancel auction reverts if conclusion timestamp has passed
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/Auction.sol#L351-L354

## Tool used

Manual Review

## Recommendation

Allow FMAP functions to be cancelled even after conclusion / add another method to retrieve remaining funds from FMAP auctions

# <a id='M-02'></a>M-02 Lot id is always set to 0 for new auctions

## Vulnerability Detail

When creating an auction, the lotId used for storing RoutingData is always 0 instead of the next lotId

```solidity
    function auction(
        RoutingParams calldata routing_,
        Auction.AuctionParams calldata params_,
        string calldata infoHash_
    ) external nonReentrant returns (uint96 lotId) {

        if (address(routing_.baseToken) == address(0) || address(routing_.quoteToken) == address(0))
        {
            revert InvalidParams();
        }

        // @audit lotId used is always 0. only updated to latest value later
        Routing storage routing = lotRouting[lotId];

        .....

        // @audit stores routing information in the incorrect
        routing.seller = msg.sender;
        routing.baseToken = routing_.baseToken;
        routing.quoteToken = routing_.quoteToken;
```

Hence all routing information associated with a lot will be lost. This breaks the entire protocol as lots cannot be identified correctly and almost everything requires identifying a lot correctly via the AcutionHouse contract using their lotId

## Impact

All auctions are effectively lost when there are multiple auctions

## Code Snippet

https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/bases/Auctioneer.sol#L174

## Tool used

Manual Review

## Recommendation

Update the lotId before fetching the storage variable

# <a id='M-03'></a>M-03 Inaccurate value is used for partial fill quote amount when calculating fees

## Vulnerability Detail

The fees of an auction is managed as follows:

1. Whenever a bidder claims their payout, calculate the amount of quote tokens that should be collected as fees (instead of giving the entire quote amount to the seller) and add this to the protocol / referrers rewards

```solidity
    function claimBids(uint96 lotId_, uint64[] calldata bidIds_) external override nonReentrant {
        
        ....

        for (uint256 i = 0; i < bidClaimsLen; i++) {
            Auction.BidClaim memory bidClaim = bidClaims[i];

            if (bidClaim.payout > 0) {
               
=>              _allocateQuoteFees(
                    protocolFee,
                    referrerFee,
                    bidClaim.referrer,
                    routing.seller,
                    routing.quoteToken,
=>                  bidClaim.paid
                );
```

Here bidClaim.paid is the amount of quote tokens that was transferred in by the bidder for the purchase

```solidity
    function _allocateQuoteFees(
        uint96 protocolFee_,
        uint96 referrerFee_,
        address referrer_,
        address seller_,
        ERC20 quoteToken_,
        uint96 amount_
    ) internal returns (uint96 totalFees) {
        // Calculate fees for purchase
        (uint96 toReferrer, uint96 toProtocol) = calculateQuoteFees(
            protocolFee_, referrerFee_, referrer_ != address(0) && referrer_ != seller_, amount_
        );

        // Update fee balances if non-zero
        if (toReferrer > 0) rewards[referrer_][quoteToken_] += uint256(toReferrer);
        if (toProtocol > 0) rewards[_protocol][quoteToken_] += uint256(toProtocol);


        return toReferrer + toProtocol;
    }
```

2. Whenever the seller calls claimProceeds to withdraw the amount of quote tokens received from the auction, subtract the quote fees and give out the remaining

```solidity
    function claimProceeds(
        uint96 lotId_,
        bytes calldata callbackData_
    ) external override nonReentrant {
        
        ....
        
        uint96 totalInLessFees;
        {
=>          (, uint96 toProtocol) = calculateQuoteFees(
                lotFees[lotId_].protocolFee, lotFees[lotId_].referrerFee, false, purchased_
            );
            unchecked {
=>              totalInLessFees = purchased_ - toProtocol;
            }
        }
```

Here purchased is the total quote token amount that was collected for this auction.

In case the fees calculated in claimProceeds is less than the sum of fees allocated to the protocol / referrer via claimBids, there will be a mismatch causing the sum of (fees allocated + seller purchased quote tokens) to be greater than the total quote token amount that was transferred in for the auction. This could cause either the protocol/referrer to not obtain their rewards or the seller to not be able to claim the purchased tokens in case there are no excess quote token present in the auction house contract.

In case, totalPurchased is >= sum of all individual bid quote token amounts (as it is supposed to be), the fee allocation would be correct. But due to the inaccurate computation of the input quote token amount associated with a partial fill, it is possible for the above scenario (ie. `fees calculated in claimProceeds is less than the sum of fees allocated to the protocol / referrer via claimBids`) to occur

```solidity
    function settle(uint96 lotId_) external override nonReentrant {
        
        ....

            if (settlement.pfBidder != address(0)) {

                _allocateQuoteFees(
                    feeData.protocolFee,
                    feeData.referrerFee,
                    settlement.pfReferrer,
                    routing.seller,
                    routing.quoteToken,

                    // @audit this method of calculating the input quote token amount associated with a partial fill is not accurate
                    uint96(
=>                      Math.mulDivDown(
                            settlement.pfPayout, settlement.totalIn, settlement.totalOut
                        )
                    )
```

The above method of calculating the input token amount associated with a partial fill can cause this value to be higher than the acutal value and hence the fees allocated will be less than what the fees that will be captured from the seller will be

### POC
Apply the following diff to `test/AuctionHouse/AuctionHouseTest.sol` and run `forge test --mt testHash_SpecificPartialRounding -vv`

It is asserted that the tokens allocated as fees is greater than the tokens that will be captured from a seller for fees

```diff
diff --git a/moonraker/test/AuctionHouse/AuctionHouseTest.sol b/moonraker/test/AuctionHouse/AuctionHouseTest.sol
index 44e717d..9b32834 100644
--- a/moonraker/test/AuctionHouse/AuctionHouseTest.sol
+++ b/moonraker/test/AuctionHouse/AuctionHouseTest.sol
@@ -6,6 +6,8 @@ import {Test} from "forge-std/Test.sol";
 import {ERC20} from "solmate/tokens/ERC20.sol";
 import {Transfer} from "src/lib/Transfer.sol";
 import {FixedPointMathLib} from "solmate/utils/FixedPointMathLib.sol";
+import {SafeCastLib} from "solmate/utils/SafeCastLib.sol";
+
 
 // Mocks
 import {MockAtomicAuctionModule} from "test/modules/Auction/MockAtomicAuctionModule.sol";
@@ -134,6 +136,158 @@ abstract contract AuctionHouseTest is Test, Permit2User {
         _bidder = vm.addr(_bidderKey);
     }
 
+        function testHash_SpecificPartialRounding() public {
+        /*
+            capacity 1056499719758481066
+            previous total amount 1000000000000000000
+            bid amount 2999999999999999999997
+            price 2556460687578254783645
+            fullFill 1173497411705521567
+            excess 117388857750942341
+            pfPayout 1056108553954579226
+            pfRefund 300100000000000000633
+            new totalAmountIn 2700899999999999999364
+            usedContributionForQuoteFees 2699900000000000000698
+            quoteTokens1 1000000
+            quoteTokens2 2699900000
+            quoteTokensAllocated 2700899999
+        */
+
+        uint bidAmount = 2999999999999999999997;
+        uint marginalPrice = 2556460687578254783645;
+        uint capacity = 1056499719758481066;
+        uint previousTotalAmount = 1000000000000000000;
+        uint baseScale = 1e18;
+
+        // hasn't reached the capacity with previousTotalAmount
+        assert(
+            FixedPointMathLib.mulDivDown(previousTotalAmount, baseScale, marginalPrice) <
+                capacity
+        );
+
+        uint capacityExpended = FixedPointMathLib.mulDivDown(
+            previousTotalAmount + bidAmount,
+            baseScale,
+            marginalPrice
+        );
+        assert(capacityExpended > capacity);
+
+        uint totalAmountIn = previousTotalAmount + bidAmount;
+
+        uint256 fullFill = FixedPointMathLib.mulDivDown(
+            uint256(bidAmount),
+            baseScale,
+            marginalPrice
+        );
+
+        uint256 excess = capacityExpended - capacity;
+
+        uint pfPayout = SafeCastLib.safeCastTo96(fullFill - excess);
+        uint pfRefund = SafeCastLib.safeCastTo96(
+            FixedPointMathLib.mulDivDown(uint256(bidAmount), excess, fullFill)
+        );
+
+        totalAmountIn -= pfRefund;
+
+        uint usedContributionForQuoteFees;
+        {
+            uint totalOut = SafeCastLib.safeCastTo96(
+                capacityExpended > capacity ? capacity : capacityExpended
+            );
+
+            usedContributionForQuoteFees = FixedPointMathLib.mulDivDown(
+                pfPayout,
+                totalAmountIn,
+                totalOut
+            );
+        }
+
+        {
+            uint actualContribution = bidAmount - pfRefund;
+
+            // acutal contribution is less than the usedContributionForQuoteFees
+            assert(actualContribution < usedContributionForQuoteFees);
+            console2.log("actual contribution", actualContribution);
+            console2.log(
+                "used contribution for fees",
+                usedContributionForQuoteFees
+            );
+        }
+
+        // calculating quote fees allocation
+        // quote fees captured from the seller
+        {
+            (, uint96 quoteTokensAllocated) = calculateQuoteFees(
+                1e3,
+                0,
+                false,
+                SafeCastLib.safeCastTo96(totalAmountIn)
+            );
+
+            // quote tokens that will be allocated for the earlier bid
+            (, uint96 quoteTokens1) = calculateQuoteFees(
+                1e3,
+                0,
+                false,
+                SafeCastLib.safeCastTo96(previousTotalAmount)
+            );
+
+            // quote tokens that will be allocated for the partial fill
+            (, uint96 quoteTokens2) = calculateQuoteFees(
+                1e3,
+                0,
+                false,
+                SafeCastLib.safeCastTo96(usedContributionForQuoteFees)
+            );
+            
+            console2.log("quoteTokens1", quoteTokens1);
+            console2.log("quoteTokens2", quoteTokens2);
+            console2.log("quoteTokensAllocated", quoteTokensAllocated);
+
+            // quoteToken fees allocated is greater than what will be captured from seller
+            assert(quoteTokens1 + quoteTokens2 > quoteTokensAllocated);
+        }
+    }
+
+        function calculateQuoteFees(
+        uint96 protocolFee_,
+        uint96 referrerFee_,
+        bool hasReferrer_,
+        uint96 amount_
+    ) public pure returns (uint96 toReferrer, uint96 toProtocol) {
+        uint _FEE_DECIMALS = 5;
+        uint96 feeDecimals = uint96(_FEE_DECIMALS);
+
+        if (hasReferrer_) {
+            // In this case we need to:
+            // 1. Calculate referrer fee
+            // 2. Calculate protocol fee as the total expected fee amount minus the referrer fee
+            //    to avoid issues with rounding from separate fee calculations
+            toReferrer = uint96(
+                FixedPointMathLib.mulDivDown(amount_, referrerFee_, feeDecimals)
+            );
+            toProtocol =
+                uint96(
+                    FixedPointMathLib.mulDivDown(
+                        amount_,
+                        protocolFee_ + referrerFee_,
+                        feeDecimals
+                    )
+                ) -
+                toReferrer;
+        } else {
+            // If there is no referrer, the protocol gets the entire fee
+            toProtocol = uint96(
+                FixedPointMathLib.mulDivDown(
+                    amount_,
+                    protocolFee_ + referrerFee_,
+                    feeDecimals
+                )
+            );
+        }
+    }
+
+
     // ===== Helper Functions ===== //
 
     function _mulDivUp(uint96 mul1_, uint96 mul2_, uint96 div_) internal pure returns (uint96) {

```

## Impact

Rewards might not be collectible or seller might not be able to claim the proceeds due to lack of tokens

## Code Snippet

inaccurate computation of the input quote token value for allocating fees
https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/AuctionHouse.sol#L512-L515

## Tool used

Manual Review

## Recommendation

Use `bidAmount - pfRefund` as the quote token input amount value instead of computing the current way

# <a id='M-04'></a>M-04 User's can be griefed by not submitting the private key

## Vulnerability Detail

Bids cannot be refunded once the auction concludes. And bids cannot be claimed until the auction has been settled. Similarly a EMPAM auction cannot be cancelled once started. 

```solidity
    function claimBids(
        uint96 lotId_,
        uint64[] calldata bidIds_
    )
        external
        override
        onlyInternal
        returns (BidClaim[] memory bidClaims, bytes memory auctionOutput)
    {
        // Standard validation
        _revertIfLotInvalid(lotId_);
        _revertIfLotNotSettled(lotId_);
```

```solidity
    function refundBid(
        uint96 lotId_,
        uint64 bidId_,
        address caller_
    ) external override onlyInternal returns (uint96 refund) {
        // Standard validation
        _revertIfLotInvalid(lotId_);
        _revertIfBeforeLotStart(lotId_);
        _revertIfBidInvalid(lotId_, bidId_);
        _revertIfNotBidOwner(lotId_, bidId_, caller_);
        _revertIfBidClaimed(lotId_, bidId_);
        _revertIfLotConcluded(lotId_);
```

```solidity
    function _cancelAuction(uint96 lotId_) internal override {
        // Validation
        // Batch auctions cannot be cancelled once started, otherwise the seller could cancel the auction after bids have been submitted
        _revertIfLotActive(lotId_);
```

```solidity
    function cancelAuction(uint96 lotId_) external override onlyInternal {
        // Validation
        _revertIfLotInvalid(lotId_);
        _revertIfLotConcluded(lotId_);
```

```solidity
    function _settle(uint96 lotId_)
        internal
        override
        returns (Settlement memory settlement_, bytes memory auctionOutput_)
    {
        // Settle the auction
        // Check that auction is in the right state for settlement
        if (auctionData[lotId_].status != Auction.Status.Decrypted) {
            revert Auction_WrongState(lotId_);
        }
```

For EMPAM auctions, the private key associated with the auction has to be submitted before the auction can be settled. In auctions where the private key is held by the seller, they can grief the bidder's or in cases where a key management solution is used, both seller and bidder's can be griefed by not submitting the private key.

## Impact

User's will not be able to claim their assets in case the private key holder doesn't submit the key for decryption 

## Code Snippet

https://github.com/sherlock-audit/2024-03-axis-finance/blob/cadf331f12b485bac184111cdc9ba1344d9fbf01/moonraker/src/modules/auctions/EMPAM.sol#L747-L756

## Tool used

Manual Review

## Recommendation

Acknowledge the risk involved for the seller and bidder
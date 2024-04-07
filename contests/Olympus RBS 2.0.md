# Olympus RBS 2.0 

Type: CONTEST

Dates: 5 Dec 2023 - 26 Dec 2023

Olympus is building OHM, a community-owned, decentralized and censorship-resistant reserve currency that is asset-backed, deeply liquid and used widely across Web3.

Code Under Review: https://github.com/sherlock-audit/2023-11-olympus

More contest details: https://audits.sherlock.xyz/contests/128

Rank: 1

# Findings Summary

- High : 3
- Medium : 5


|ID|Title|Severity|
|-|:-|:-:|
| [H-01](#H-01)| Incorrect ProtocolOwnedLiquidityOhm calculation due to inclusion of other user's reserves | High |
| [H-02](#H-02)| BunniPrice returns totalValue instead of pool token price | High |
| [H-03](#H-03)| Incorrect StablePool BPT price calculation | High |
| [M-01](#M-01)| Possible incorrect price for tokens in Balancer stable pool due to amplification parameter update | Medium |
| [M-02](#M-02)| Incorrect deviation check | Medium |
| [M-03](#M-03)| Pool manipulation check in BunniHelper is flawed as uncollected fees is used | Medium |
| [M-04](#M-04)| Bunni liquidityReserves doesn't count owed fees | Medium |
| [M-05](#M-05)| usage of totalSupply for newer balancer pools should be replaced with getActualSupply | Medium |


# Detailed Findings

# <a id='H-01'></a>H-01 Incorrect ProtocolOwnedLiquidityOhm calculation due to inclusion of other user's reserves

## Vulnerability Detail
The protocol owned liquidity in Bunni is calculated as the sum of reserves of all the BunniTokens
```solidity
    function getProtocolOwnedLiquidityOhm() external view override returns (uint256) {

        uint256 len = bunniTokens.length;
        uint256 total;
        for (uint256 i; i < len; ) {
            TokenData storage tokenData = bunniTokens[i];
            BunniLens lens = tokenData.lens;
            BunniKey memory key = _getBunniKey(tokenData.token);

        .........

            total += _getOhmReserves(key, lens);
            unchecked {
                ++i;
            }
        }


        return total;
    }
```

The deposit function of Bunni allows any user to add liquidity to a token. Hence the returned reserve will contain amounts other than the reserves that actually belong to the protocol
```solidity

    // @audit callable by any user
    function deposit(
        DepositParams calldata params
    )
        external
        payable
        virtual
        override
        checkDeadline(params.deadline)
        returns (uint256 shares, uint128 addedLiquidity, uint256 amount0, uint256 amount1)
    {
    }
```  
## Impact
Incorrect assumption of the protocol owned liquidity. An attacker can inflate the liquidity reserves

## Code Snippet
POL liquidity is calculated as the sum of bunni token reserves
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/SPPLY/submodules/BunniSupply.sol#L171-L191

BunniHub allows any user to deposit
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/external/bunni/BunniHub.sol#L71-L106

## Tool used
Manual Review

## Recommendation
Guard the deposit function in BunniHub or compute the liquidity using shares belonging to the protocol

# <a id='H-02'></a>H-02 BunniPrice returns totalValue instead of pool token price

## Vulnerability Detail
The `getBunniTokenPrice` function returns the totalValue of the Bunni token instead of the price 
```solidity
    function getBunniTokenPrice(
        address bunniToken_,
        uint8 outputDecimals_,
        bytes calldata params_
    ) external view returns (uint256) {
       
        ......

        // Fetch the reserves
        uint256 totalValue = _getTotalValue(token, lens, outputDecimals_);

        return totalValue;
    }
```

## Impact
Incorrect price for BunniToken

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol#L110-L166


## Tool used
Manual Review

## Recommendation
Divide by the totalSupply to obtain the price

# <a id='H-03'></a>H-03 Incorrect StablePool BPT price calculation

## Vulnerability Detail
The price of a stable pool BPT is computed as:

> minimum price among the pool tokens obtained via feeds * return value of `getRate()`
 
This method is used referring to an old documentation of Balancer 

```solidity
    function getStablePoolTokenPrice(
        address,
        uint8 outputDecimals_,
        bytes calldata params_
    ) external view returns (uint256) {
        // Prevent overflow
        if (outputDecimals_ > BASE_10_MAX_EXPONENT)
            revert Balancer_OutputDecimalsOutOfBounds(outputDecimals_, BASE_10_MAX_EXPONENT);


        address[] memory tokens;
        uint256 poolRate; // pool decimals
        uint8 poolDecimals;
        bytes32 poolId;
        {

        ......

            // Get tokens in the pool from vault
            (address[] memory tokens_, , ) = balVault.getPoolTokens(poolId);
            tokens = tokens_;

            // Get rate
            try pool.getRate() returns (uint256 rate_) {
                if (rate_ == 0) {
                    revert Balancer_PoolStableRateInvalid(poolId, 0);
                }


                poolRate = rate_;

        ......

        uint256 minimumPrice; // outputDecimals_
        {
            /**
             * The Balancer docs do not currently state this, but a historical version noted
             * that getRate() should be multiplied by the minimum price of the tokens in the
             * pool in order to get a valuation. This is the same approach as used by Curve stable pools.
             */
            for (uint256 i; i < len; i++) {
                address token = tokens[i];
                if (token == address(0)) revert Balancer_PoolTokenInvalid(poolId, i, token);

                (uint256 price_, ) = _PRICE().getPrice(token, PRICEv2.Variant.CURRENT); // outputDecimals_


                if (minimumPrice == 0) {
                    minimumPrice = price_;
                } else if (price_ < minimumPrice) {
                    minimumPrice = price_;
                }
            }
        }

        uint256 poolValue = poolRate.mulDiv(minimumPrice, 10 ** poolDecimals); // outputDecimals_
```

The `getRate()` function returns the exchange rate of a BPT to the underlying base asset of the pool which can be different from the minimum market priced asset for pools with rateProviders. To consider this, the price obtained from feeds must be divided by the `rate` provided by `rateProviders` before choosing the minimum as mentioned in the previous version of Balancer's documentation.   

https://github.com/balancer/docs/blob/663e2f4f2c3eee6f85805e102434629633af92a2/docs/concepts/advanced/valuing-bpt/bpt-as-collateral.md#metastablepools-eg-wsteth-weth 

```markdown
#### 1. Get market price for each constituent token

Get market price of wstETH and WETH in terms of USD, using chainlink oracles.

#### 2. Get RateProvider price for each constituent token

Since wstETH - WETH pool is a MetaStablePool and not a ComposableStablePool, it does not have `getTokenRate()` function.
Therefore, it`s needed to get the RateProvider price manually for wstETH, using the rate providers of the pool. The rate 
provider will return the wstETH token in terms of stETH.

Note that WETH does not have a rate provider for this pool. In that case, assume a value of `1e18` (it means,
market price of WETH won't be divided by any value, and it's used purely in the minPrice formula).

#### 3. Get minimum price

$$ minPrice = min({P_{M_{wstETH}} \over P_{RP_{wstETH}}}, P_{M_{WETH}}) $$

#### 4. Calculates the BPT price

$$ P_{BPT_{wstETH-WETH}} = minPrice * rate_{pool_{wstETH-WETH}} $$

where `rate_pool_wstETH-WETH` is `pool.getRate()` of wstETH-WETH pool.
```

### Example

The wstEth-cbEth pool is a MetaStablePool having rate providers for both tokens since neither of them is the base token
https://app.balancer.fi/#/ethereum/pool/0x9c6d47ff73e0f5e51be5fd53236e3f595c5793f200020000000000000000042c

At block 18821323:
cbeth : 2317.48812
wstEth : 2526.84
pool total supply : 0.273259897168240633
getRate() : 1.022627523581711856
wstRateprovider rate : 1.150725009180224306
cbEthRateProvider rate : 1.058783029570983377
wstEth balance : 0.133842314907166538
cbeth balance : 0.119822100236557012
tvl : (0.133842314907166538 * 2526.84 + 0.119822100236557012 * 2317.48812) == 615.884408812

according to current implementation:
bpt price = 2317.48812 * 1.022627523581711856 == 2369.927137086
calculated tvl = bpt price * total supply = 647.606045776

correct calculation:
rate_provided_adjusted_cbeth = (2317.48812 / 1.058783029570983377) == 2188.822502132
rate_provided_adjusted_wsteth = (2526.84 / 1.150725009180224306) == 2195.867804942
bpt price = 2188.822502132 * 1.022627523581711856 == 2238.350134915
calculated tvl = bpt price * total supply = (2238.350134915 * 0.273259897168240633) == 611.651327693

## Impact
Incorrect calculation of bpt price. Has possibility to be over and under valued.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/BalancerPoolTokenPrice.sol#L514-L539

## Tool used
Manual Review

## Recommendation
For pools having rate providers, divide prices by rate before choosing the minimum

# <a id='M-01'></a>M-01 Possible incorrect price for tokens in Balancer stable pool due to amplification parameter update

## Vulnerability Detail
The amplification parameter used to calculate the invariant can be in a state of update. In such a case, the current amplification parameter can differ from the amplificaiton parameter at the time of the last invariant calculation.
The current implementaiton of `getTokenPriceFromStablePool` doesn't consider this and always uses the amplification factor obtained by calling `getLastInvariant` 

https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/BalancerPoolTokenPrice.sol#L811-L827
```solidity
            function getTokenPriceFromStablePool(
        address lookupToken_,
        uint8 outputDecimals_,
        bytes calldata params_
    ) external view returns (uint256) {

                .....

                try pool.getLastInvariant() returns (uint256, uint256 ampFactor) {
                   
                   // @audit the amplification factor as of the last invariant calculation is used
                    lookupTokensPerDestinationToken = StableMath._calcOutGivenIn(
                        ampFactor,
                        balances_,
                        destinationTokenIndex,
                        lookupTokenIndex,
                        1e18,
                        StableMath._calculateInvariant(ampFactor, balances_) // Sometimes the fetched invariant value does not work, so calculate it
                    );
```

https://vscode.blockscan.com/ethereum/0x1e19cf2d73a72ef1332c882f20534b6519be0276
StablePool.sol
```solidity

        // @audit the amplification parameter can be updated
        function startAmplificationParameterUpdate(uint256 rawEndValue, uint256 endTime) external authenticate {

        // @audit for calculating the invariant the current amplification factor is obtained by calling _getAmplificationParameter()
        function _onSwapGivenIn(
        SwapRequest memory swapRequest,
        uint256[] memory balances,
        uint256 indexIn,
        uint256 indexOut
    ) internal virtual override whenNotPaused returns (uint256) {
        (uint256 currentAmp, ) = _getAmplificationParameter();
        uint256 amountOut = StableMath._calcOutGivenIn(currentAmp, balances, indexIn, indexOut, swapRequest.amount);
        return amountOut;
    }
```


## Impact
In case the amplification parameter of a pool is being updated by the admin, wrong price will be calculated.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/BalancerPoolTokenPrice.sol#L811-L827

## Tool used
Manual Review

## Recommendation
Use the latest amplification factor by callling the `getAmplificationParameter` function

# <a id='M-02'></a>M-02 Incorrect deviation check

## Vulnerability Detail
When computing the deviations of prices from TWAP in Bunni and UniV3 `Deviation.isDeviatingWithBpsCheck()` always uses the highest of the two values as the denominator
```solidity

    function isDeviating(
        uint256 value0_,
        uint256 value1_,
        uint256 deviationBps_,
        uint256 deviationMax_
    ) internal pure returns (bool) {
        return
            (value0_ < value1_)
                ? _isDeviating(value1_, value0_, deviationBps_, deviationMax_)
                : _isDeviating(value0_, value1_, deviationBps_, deviationMax_);
    }

    function _isDeviating(
        uint256 value0_,
        uint256 value1_,
        uint256 deviationBps_,
        uint256 deviationMax_
    ) internal pure returns (bool) {
        return ((value0_ - value1_) * deviationMax_) / value0_ > deviationBps_;
    }
```
```solidity
    function _validateReserves(
        BunniKey memory key_,
        BunniLens lens_,
        uint16 twapMaxDeviationBps_,
        uint32 twapObservationWindow_
    ) internal view {

        .....

        if (
            // `isDeviatingWithBpsCheck()` will revert if `deviationBps` is invalid.
            Deviation.isDeviatingWithBpsCheck(
                reservesTokenRatio,
                twapTokenRatio,
                twapMaxDeviationBps_,
                TWAP_MAX_DEVIATION_BASE
            )
        ) {
            revert BunniPrice_PriceMismatch(address(key_.pool), twapTokenRatio, reservesTokenRatio);
        }
```
```solidity
    function getTokenPrice(
        address lookupToken_,
        uint8 outputDecimals_,
        bytes calldata params_
    ) external view returns (uint256) {
        
        ......

        if (
            // `isDeviatingWithBpsCheck()` will revert if `deviationBps` is invalid.
            Deviation.isDeviatingWithBpsCheck(
                baseInQuotePrice,
                baseInQuoteTWAP,
                params.maxDeviationBps,
                DEVIATION_BASE
            )
        ) {
            revert UniswapV3_PriceMismatch(address(params.pool), baseInQuoteTWAP, baseInQuotePrice);
        }
```
This allows for underreporting of the deviation causing the pool to be manipulated outside of acceptable limits

### Example
TWAP = 100
Spot Price = 200
maxDeviationBps = 51%

Actual deviation = (200 - 100) / 100 == 100%
Calculated deviation = (200 - 100) / 200 == 50%

## Impact
Pools can be manipulated outside of expected limits

## Code Snippet
deviation lib
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/libraries/Deviation.sol#L69

bunni deviation check
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/BunniPrice.sol#L255-L265

uniV3 deviation check
https://github.com/sherlock-audit/2023-11-olympus/blob/main/bophades/src/modules/PRICE/submodules/feeds/UniswapV3Price.sol#L227-L235

## Tool used

Manual Review

## Recommendation
Keep TWAP in the denominator

# <a id='M-03'></a>M-03 Pool manipulation check in BunniHelper is flawed as uncollected fees is used

## Vulnerability Detail
`_validateReserves` function is used to validate that the current price have not been manipulated before calculating the price/supply of BunniTokens. It does this by checking the deviation of `reservesRatio` from `TWAP`. 
If deposited in the entire range the `liquidity reserves ratio` would track the spot price. But in the calculation of `reservesRatio`, the uncollected fees is also used. This can cause the calculation to deviate from the `TWAP` or approach `TWAP` even if the actual reserve ratio is differing from `TWAP`
```solidity
    function getReservesRatio(BunniKey memory key_, BunniLens lens_) public view returns (uint256) {
        IUniswapV3Pool pool = key_.pool;
        uint8 token0Decimals = ERC20(pool.token0()).decimals();


        (uint112 reserve0, uint112 reserve1) = lens_.getReserves(key_);
        (uint256 fee0, uint256 fee1) = lens_.getUncollectedFees(key_);


        return (reserve1 + fee1).mulDiv(10 ** token0Decimals, reserve0 + fee0);
    }
```

## Impact
1. Allows for manipulation of prices beyond expected deviation: The actual reserve ratio can be increased or decreased by an attacker without deviating from TWAP if the uncollected fee amounts make up the ratio when added. It can happen naturally due to fee accrual or can be caused by an attacker by swapping. Depending on the amount of liquidity present in the pool and the amount of liquidity mintable by an attacker, it might become possible to achieve this at minimal loss since the fees calculated can include fees claimable by the attacker as Bunnihub allows any user to deposit.

2. Price and supply calculation can revert even without any pool manipulation.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/libraries/UniswapV3/BunniHelper.sol#L47-L55


## Tool used
Manual Review

## Recommendation
Don't use uncollected fees in the calculation

# <a id='M-04'></a>M-04 Bunni liquidityReserves doesn't count owed fees

## Vulnerability Detail
The `getProtocolOwnedLiquidityReserves` function returns the (liquidity reserves + uncollected fees)
```solidity
        function getProtocolOwnedLiquidityReserves()
        external
        view
        override
        returns (SPPLYv1.Reserves[] memory)
    {
        .....

        for (uint256 i; i < len; ) {
            TokenData storage tokenData = bunniTokens[i];
            BunniToken token = tokenData.token;
            BunniLens lens = tokenData.lens;
            BunniKey memory key = _getBunniKey(token);
            (
                address token0,
                address token1,
                uint256 reserve0,
                uint256 reserve1
            ) = _getReservesWithFees(key, lens);
```
```solidity
    function _getReservesWithFees(
        BunniKey memory key_,
        BunniLens lens_
    ) internal view returns (address, address, uint256, uint256) {
        (uint112 reserve0, uint112 reserve1) = lens_.getReserves(key_);
        (uint256 fee0, uint256 fee1) = lens_.getUncollectedFees(key_);

        return (key_.pool.token0(), key_.pool.token1(), reserve0 + fee0, reserve1 + fee1);
    }
```

But since Bunnihub has a function to account the owed fees, it is possible for token amounts to be present as positions owed token amounts. This amount will not be reported by the `_getReservesWithFees` function.  
```solidity
    function updateSwapFees(
        BunniKey calldata key
    ) external virtual override returns (uint256 swapFee0, uint256 swapFee1) {
        key.pool.burn(key.tickLower, key.tickUpper, 0);
        (, , , uint128 cachedFeesOwed0, uint128 cachedFeesOwed1) = key.pool.positions(
            keccak256(abi.encodePacked(address(this), key.tickLower, key.tickUpper))
        );

        return (cachedFeesOwed0, cachedFeesOwed1);
    }
```

## Impact
Underreporting of the total token amounts available for a bunni token.

## Code Snippet
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/SPPLY/submodules/BunniSupply.sol#L417-L425


## Tool used
Manual Review

## Recommendation
Include the owed token amounts

# <a id='M-05'></a>M-05 usage of totalSupply for newer balancer pools should be replaced with getActualSupply

## Vulnerability Detail

Balancer pools have different methods to get their total supply of minted LP tokens, which is also specified in the docs here
https://docs.balancer.fi/concepts/advanced/valuing-bpt/valuing-bpt.html#getting-bpt-supply .
The docs specifies that getActualSupply is used by the most recent versions of Weighted and Stable Pools. It accounts for pre-minted BPT as well as due protocol fees. To give you few examples of newer weighted pools that uses getActualSupply
https://etherscan.io/address/0x9f9d900462492d4c21e9523ca95a7cd86142f298
https://etherscan.io/address/0x3ff3a210e57cfe679d9ad1e9ba6453a716c56a2e
https://etherscan.io/address/0xcf7b51ce5755513d4be016b0e28d6edeffa1d52a

In the current implementation, totalSupply is used instead of getActualSupply for the calculation of supply and weighted pool token prices. This can cause incorrect computation for newer pools.

## Impact
Incorrect weighted pool token prices and incorrect supply calculation for newer pools.

## Code Snippet
in price
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/PRICE/submodules/feeds/BalancerPoolTokenPrice.sol#L402

in supply
https://github.com/sherlock-audit/2023-11-olympus/blob/9c8df76dc9820b4c6605d2e1e6d87dcfa9e50070/bophades/src/modules/SPPLY/submodules/AuraBalancerSupply.sol#L345

## Tool used
Manual Review

## Recommendation
If getActualSupply function exists on the contract, invoke it to obtain the supply.
# Olas 

Type: CONTEST

Dates: 22 Dec 2023 - 9 Jan 2024

Olas is a protocol that enables developers to create and run services on the Olas network, which is a platform for coordinating and incentivizing off-chain services in crypto

Code Under Review: https://github.com/code-423n4/2023-12-autonolas

More contest details: https://code4rena.com/audits/2023-12-olas

Rank: 2

# Findings Summary

- High : 3
- Medium : 2


|ID|Title|Severity|
|-|:-|:-:|
| [H-01](#H-01)| Bonds created in year cross epoch's can lead to lost payouts | High |
| [H-02](#H-02)| Attacker can cause deposits to be locked in the Solana lockbox | High |
| [H-03](#H-03)| Wrong invocation of Whirpools's updateFeesAndRewards will cause it to always revert | High |
| [M-01](#M-01)| Possible DOS when withdrawing liquidity from Solana Lockbox | Medium |
| [M-02](#M-02)| User's / protocol would be unable to withdraw intended amounts from Solana lockbox | Medium |


# Detailed Findings

# <a id='H-01'></a>H-01 Bonds created in year cross epoch's can lead to lost payouts

## Impact
Bond depositors and agent/component owner's may never receive the payout Olas. 
Incorrect inflation control

## Proof of Concept
`effectiveBond` is used to account how much of Olas is available for bonding. This includes Olas that are to be minted in the current epoch ie. when epoch 5 starts, effectiveBond would have the Olas partitioned  for bonding in epoch 5. In case of epoch's crossing `YEAR` intervals, a portion of the Olas would actually only be mintable in the next year due to the yearwise inflation control enforced at the mint (after 9 years due to fixed supply till 10 years). Due to silent reverts, this can lead to lost Olas payouts

The inflation for bonds are accounted using the `effectiveBond` variable.
https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/tokenomics/contracts/Tokenomics.sol#L609-L617
```solidity
    function reserveAmountForBondProgram(uint256 amount) external returns (bool success) {
       
       .....

        // Effective bond must be bigger than the requested amount
        uint256 eBond = effectiveBond;
        if (eBond >= amount) {

            eBond -= amount;
            effectiveBond = uint96(eBond);
            success = true;
            emit EffectiveBondUpdated(eBond);
        }
    }
```

This variable is updated with the estimated bond Olas at the beginning of an epoch itself.

https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/tokenomics/contracts/Tokenomics.sol#L1037-L1038

```solidity
    function checkpoint() external returns (bool) {
        
        .....

        // Update effectiveBond with the current or updated maxBond value
        curMaxBond += effectiveBond;
        effectiveBond = uint96(curMaxBond);
```

In case of epochs crossing `YEAR` intervals after 9 years, the new Olas amount will not be fully mintable in the same year due to the inflation control check enforced in the Olas contract.

https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/governance/contracts/OLAS.sol#L75-L84
```solidity
    function mint(address account, uint256 amount) external {

        ....
        
        // Check the inflation schedule and mint
        if (inflationControl(amount)) {
            _mint(account, amount);
        }
```

Whenever a deposit is made on a bond, the required Olas is minted by the treasury and transferred to the Depository contract, from where the depositor claims the payout after the vesting time. `Olas.sol` doesn't revert for inflation check failure but fails silently. This can cause a deposit to succeed but corresponding redeem to fail since payout Olas has not been minted actually. 
It can also happen that agent/component owner's who have not claimed the topup Olas amount will loose their reward due to silent return when minting their reward.  

### Example
- Year 10, 1 month left for Year 11
- All Olas associated with previous epochs have been minted
- New epoch of 2 months is started, 1 month in Year 10 and 1 month in Year 11
- Total Olas for the epoch, t = year 10 1 month inflation + year 11 1 month inflation
 year 10 1 month inflaiton (y10m1) = (1_000_000_000e18 * 2 / 100 / 12) 
 year 11 1 month inflation (y11m1) = (1_020_000_000e18 * 2 / 100 / 12)
 t = y10m1 + y11m1 
- Olas bond percentage = 50%
- Hence effectiveBond = t/2
- But actual mintable remaining in year 0, m = y10m1 < effectiveBond
- A bond is created with supply == effectiveBond
- User's deposit for the entire bond supply but only y10m1 Olas can be minted. Depending on the nature of deposits, the actual amount minted can vary from 0 to y10m1. In case of unminted amounts(as rewards of agent/component owner's etc.) at Year 10, this amount can be minted for bond deposits following which if agent/component owners claim within the year, no Olas will be received by them.
- Users loose their Olas payout

### POC Test
https://gist.github.com/10xhash/2157c1f2cdc9513b3f0a7f359a65015e

## Tools Used
Manual review

## Recommended Mitigation Steps
In case of multi-year epochs, seperate bond amounts of next year

# <a id='H-02'></a>H-02 Attacker can cause deposits to be locked in the Solana lockbox

## Impact
An attacker can cause deposits to be locked in the lockbox 

## Proof of Concept
In `withdraw`, if the position has 0 liquidity the execution is reverted

https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/lockbox-solana/solidity/liquidity_lockbox.sol#L223
```solidity
    function withdraw(uint64 amount) external {
        address positionAddress = positionAccounts[firstAvailablePositionAccountIndex];
        
        .....

        uint64 positionLiquidity = mapPositionAccountLiquidity[positionAddress];
        // Check that the token account exists
        if (positionLiquidity == 0) {
            revert("No liquidity on a provided token account");
        }
```

Since the withdrawal positions are chosen sequentially, a position with liquidity 0 present in the ith index of `positionAccounts` array, will make all positions from `i -> end` locked and unwithdrawable.

Due to lack of checks in the deposit function, an attacker can make a depsoit with 0 liquidity resulting in the above scenario. This will lead to all funds associated with further deposits to be locked in the lockbox. 

https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/lockbox-solana/solidity/liquidity_lockbox.sol#L84

https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/lockbox-solana/solidity/liquidity_lockbox.sol#L140

### POC Test
https://gist.github.com/10xhash/168166ba89447fa6fbc61981866c1b28

## Tools Used
Manual review

## Recommended Mitigation Steps
Require liquidity of depositing positions to be non-zero

# <a id='H-03'></a>H-03 Wrong invocation of Whirpools's updateFeesAndRewards will cause it to always revert

## Impact
Deposits will be unwithdrawable from the lockbox 

## Proof of Concept
If the entire liquidity of a position has been remvoed, the withdraw function calls the `updateFeesAndRewards` function on the Orca pool before attempting to close the position.

https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/lockbox-solana/solidity/liquidity_lockbox.sol#L277-L293
```solidity
    function withdraw(uint64 amount) external {
        address positionAddress = positionAccounts[firstAvailablePositionAccountIndex];
      
        ......

        uint64 positionLiquidity = mapPositionAccountLiquidity[positionAddress];
        
        ......

        uint64 remainder = positionLiquidity - amount;

        ......

        if (remainder == 0) {
            // Update fees for the position
            AccountMeta[4] metasUpdateFees = [
                AccountMeta({pubkey: pool, is_writable: true, is_signer: false}),
                AccountMeta({pubkey: positionAddress, is_writable: true, is_signer: false}),
                AccountMeta({pubkey: tx.accounts.tickArrayLower.key, is_writable: false, is_signer: false}),
                AccountMeta({pubkey: tx.accounts.tickArrayUpper.key, is_writable: false, is_signer: false})
            ];
            whirlpool.updateFeesAndRewards{accounts: metasUpdateFees, seeds: [[pdaProgramSeed, pdaBump]]}();
```

This is faulty as the `updateFeesAndRewards` function will always revert if the position's liquidity is 0.

https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/lockbox-solana/solidity/interfaces/whirlpool.sol#L198

### Whirlpool source code:
[update_fees_and_rewards](https://github.com/orca-so/whirlpools/blob/3206c9cdfbf27c73c30cbcf5b6df2929cbf87618/programs/whirlpool/src/instructions/update_fees_and_rewards.rs#L27) -> [calculate_fee_and_reward_growths](https://github.com/orca-so/whirlpools/blob/3206c9cdfbf27c73c30cbcf5b6df2929cbf87618/programs/whirlpool/src/manager/liquidity_manager.rs#L72) -> [_calculate_modify_liquidity](https://github.com/orca-so/whirlpools/blob/3206c9cdfbf27c73c30cbcf5b6df2929cbf87618/programs/whirlpool/src/manager/liquidity_manager.rs#L97-L99)

https://github.com/orca-so/whirlpools/blob/3206c9cdfbf27c73c30cbcf5b6df2929cbf87618/programs/whirlpool/src/manager/liquidity_manager.rs#L97-L99
```rs
fn _calculate_modify_liquidity(
    whirlpool: &Whirlpool,
    position: &Position,
    tick_lower: &Tick,
    tick_upper: &Tick,
    tick_lower_index: i32,
    tick_upper_index: i32,
    liquidity_delta: i128,
    timestamp: u64,
) -> Result<ModifyLiquidityUpdate> {
    // Disallow only updating position fee and reward growth when position has zero liquidity
    if liquidity_delta == 0 && position.liquidity == 0 {
        return Err(ErrorCode::LiquidityZero.into());
    }
```

Since the withdrawal positions are chosen sequentially, only a maximum of (first position's liquidity - 1) amount of liquidity can be withdrawn. 

### POC Test
https://gist.github.com/10xhash/a687ef66de8210444a41360b86ed4bca

## Tools Used
Manual review

## Recommended Mitigation Steps
Avoid the `update_fees_and_rewards` call completely since fees and rewards would be updated in the `decreaseLiquidity` call.

# <a id='M-01'></a>M-01 Possible DOS when withdrawing liquidity from Solana Lockbox

## Impact
Possible DOS for liquidity withdrawal from the lockbox

## Proof of Concept
When withdrawing it is required to pass all the associated accounts in the [transaction](https://solanacookbook.com/core-concepts/transactions.html#deep-dive). But among these (position,pdaPositionAccount and positionMint) are dependent on time ie. if another withdrawal occurs, the required accounts to be passed to the function call might change resulting in a revert.

https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/lockbox-solana/solidity/liquidity_lockbox.sol#L194-L214
```solidity
    @mutableAccount(pool)
    @account(tokenProgramId)
    @mutableAccount(position)
    @mutableAccount(userBridgedTokenAccount)
    @mutableAccount(pdaBridgedTokenAccount)
    @mutableAccount(userWallet)
    @mutableAccount(bridgedTokenMint)
    @mutableAccount(pdaPositionAccount)
    @mutableAccount(userTokenAccountA)
    @mutableAccount(userTokenAccountB)
    @mutableAccount(tokenVaultA)
    @mutableAccount(tokenVaultB)
    @mutableAccount(tickArrayLower)
    @mutableAccount(tickArrayUpper)
    @mutableAccount(positionMint)
    @signer(sig)
    function withdraw(uint64 amount) external {
        address positionAddress = positionAccounts[firstAvailablePositionAccountIndex];
        if (positionAddress != tx.accounts.position.key) {
            revert("Wrong liquidity token account");
        }
```

The DOS for a withdrawal can be caused by another user withdrawing before the user's transaction. Due to the possibility to steal fees, attackers would be motivated to frequently call the withdraw method making such a scenario likely.   

## Tools Used
Manual review

## Recommended Mitigation Steps
To mitigate this it would require a redesign on how the lockbox accepts liquidity. Instead of adding new positions, the lockbox can keep its liquidity in a single position and continuosly increase it for deposits.

# <a id='M-02'></a>M-02 User's / protocol would be unable to withdraw intended amounts from Solana lockbox

## Impact
User's / protocol unable to withdraw intended amounts from the lockbox

## Proof of Concept
When withdrawing it is required to pass all the associated accounts correctly. Hence the `getLiquidityAmountsAndPositions` function is implemented to obtain the required data for a given withdrawal amount.
But the function is wrongly implemented and gives incorrect withdrawal amount for the last position

https://github.com/code-423n4/2023-12-autonolas/blob/2a095eb1f8359be349d23af67089795fb0be4ed1/lockbox-solana/solidity/liquidity_lockbox.sol#L338-L379
```solidity
    function getLiquidityAmountsAndPositions(uint64 amount)
        external view returns (uint64[] positionAmounts, address[] positionAddresses, address[] positionPdaAtas)
    {
        
        ......

        uint64 liquiditySum = 0;

        for (uint32 i = firstAvailablePositionAccountIndex; i < numPositionAccounts; ++i) {

            .....

            liquiditySum += positionLiquidity;

            .....

            if (liquiditySum >= amount) {
                amountLeft = liquiditySum - amount;
                break;
            }
        }

        for (uint32 i = 0; i < numPositions; ++i) {
            
            ....

            positionAmounts[i] = mapPositionAccountLiquidity[positionAddresses[i]];
            
            ....
        }

        // @audit incorrect
        if (numPositions > 0 && amountLeft > 0) {
            positionAmounts[numPositions - 1] = amountLeft;
        }
    }
```

Instead of subtracting `amountLeft` from the final position, the positionAmount is set as amountLeft. This can cause the overall sum to be greater or less than the actual amount

### Example
1 Position with liquidity 150
User wants to withdraw 100

Function walkthrough:
After first loop block, liquiditySum == 150 hence amountLeft = 50
In the second loop block, positionAmounts[0] would be set to 50
This would make the totalWithdrawal amount to 50 while the actual intended withdrawal was 100

## Tools Used
Manual review

## Recommended Mitigation Steps
Subtract from the final position instead of setting
```solidity
        if (numPositions > 0 && amountLeft > 0) {
            positionAmounts[numPositions - 1] -= amountLeft;
        }
```
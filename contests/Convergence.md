# Convergence 

Type: CONTEST

Dates: 15 Nov 2023 - 29 Nov 2023

Next generation Governance Black Hole. Boosting yields across DeFi, starting with the CurveFinance ecosystem

Code Under Review: https://github.com/sherlock-audit/2023-11-convergence

More contest details: https://audits.sherlock.xyz/contests/126

Rank: 2

# Findings Summary

- High : 2
- Medium : 3


|ID|Title|Severity|
|-|:-|:-:|
| [H-01](#H-01)| Killing a gague can lead to bricking of the protocol | High |
| [H-02](#H-02)| User's can attain unlimited veCvg/mgCvg voting power due to lack of duplication checks | High |
| [M-01](#M-01)| Division difference can result in a revert when claiming treasury yield and excess rewards to some users | Medium |
| [M-02](#M-02)| cvgRewards may be incorrectly calculated due to possible changes in gagueWeights and totalWeight | Medium |
| [M-03](#M-03)| Incorrect slippage protection for cvgSdt/sdt conversion | Medium |


# Detailed Findings

# <a id='H-01'></a>H-01 Killing a gague can lead to bricking of the protocol

## Vulnerability Detail

Points are accounted using `bias`,`slope` and `slope end time`. The bias is gradually decreased by its slope eventually reaching 0 at the `slope end time / lock end time`. In case the bias is suddenly reduced without handling/removing its slope, there will reach a time before the `slope end time` where the bias will drop below 0. 
`points_sum` uses points to calculate the sum of individual gauge weights of a particular type at any time. The above case of bias dropping below 0 is handled as follows in `_get_sum`:
```solidity
            d_bias: uint256 = pt.slope * WEEK
            if pt.bias > d_bias:
                pt.bias -= d_bias
                d_slope: uint256 = self.changes_sum[gauge_type][t]
                pt.slope -= d_slope
            else:
                pt.bias = 0
                pt.slope = 0
```
Whenever bias drops below 0 the slope is also made 0 without removing the `slope end accounting` in `changes_sum`. Afterwards, if a scenario occurs where the pt.bias regains a value greater than the `pt.slope * WEEK`, the true part of the if condition executes which can cause it to revert if at any point `self.changes_sum[gauge_type][t]` is greater than the current `pt.slope`.

The sudden drop in `points_sum` bias can happen due to the following:
1. killing of a gauge
2. admin calls `change_gauge_weight` with a lower value than current

From this moment the accounting of `points_sum` is broken and depending on the current `slope`, `changes_sum`, `bias` and activity following this, the `points_sum.bias` can go to 0.

Once bias goes to 0, it can regain a value greater than the `pt.slope * WEEK` due to the following:

1. another gauge of the same type is added later with a weight
2. a user votes for a gauge of the same type
3. admin calling the `change_gauge_weight` increasing a gauge's weight

Scenario 2 is almost sure to happen following which all calls to `_get_sum` can revert which will cause the protocol to brick since even the cycle update involves a call to `_get_sum`

## POC
```
Gauges A and B are of same type

At t = 0
a_bias = b_bias = 100 , a_slope = b_slope = 1 , slope_end : t = 100 
sum_bias = 200 , sum_slope = 2 , slope_end : t = 100

At t = 50 before kill
a_bias = b_bias = 50 , a_slope = b_slope = 1 
sum_bias = 100 , sum_slope = 2

at t = 50 kill A
a_bias = 0 , a_slope = 1
b_bias = 50 , b_slope = 1
sum_bias = 50 , sum_slope = 2

at t = 75
b_bias = 25 , b_slope = 1
sum_bias = 0 , sum_slope = 0 (pt.slope set to 0 when bias !> d.slope)

at t = 75
another user votes for B with bias = 100 and slope = 1
b_bias = 125 , b_slope = 2
sum_bias = 100 , sum_slope = 1

at t = 100
sum_slope -= 2 ( a_slope_end = b_slope_end : t = 100)

this will cause _get_sum to revert
```

Runnable foundry test gist link:
https://gist.github.com/10xhash/b65867b99841a88e078e34f094fc0554

## Impact

Bricking of the protocol,locked user funds , incorrect `sum` and `total` weight values 

## Code Snippet

`_change_gauge_weight` sets weight directly without handling slope
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/GaugeController.vy#L568-L582

`kill_gauge` calls `_change_gauge_weight`
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/GaugeController.vy#L603C5-L609

## Tool used

Manual Review

## Recommendation

When killing the gauge, decrease its slope from the sum and iterate over all future weeks till the max limit and remove any gauge-associated-slope changes from the `changes_sum`. Handle the user removing a vote from a killed gauge seperately from the general vote function.

# <a id='H-02'></a>H-02 User's can attain unlimited veCvg/mgCvg voting power due to lack of duplication checks

## Vulnerability Detail

The `manageOwnedAndDelegated` function doesn't check for duplication among the added tokens. This allows an user to add for oneself the same token infinite number of times.
The `mgCvgVotingPowerPerAddress` and `veCvgVotingPowerPerAddress` functions which are used to compute the voting power from the `tokenOwnedAndDelegated` which was previously set by `manageOwnedAndDelegated` also doesn't check for duplication. This allows a user to gain infinite governance power.

## Impact

Attackers can gain unlimited voting power on governance disrupting the entire protocol

## Code Snippet

The `manageOwnedAndDelegated` function doesn't check for duplication
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionDelegate.sol#L330-L367

The `mgCvgVotingPowerPerAddress` function doesn't check for duplication
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L773-L821

The `veCvgVotingPowerPerAddress` function doesn't check for duplication
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L727-L767

## Tool used

Manual Review

## Recommendation

Check for duplication in the all arrays.
Since this is being set externally and the newly minted tokens having the latest tokenId, enforcing an increasing tokenId in arrays can solve this issue.

```diff
+++     uint prevTokenId;
        for (uint256 i; i < _ownedAndDelegatedTokens.owneds.length;) {
+++         require(_ownedAndDelegatedTokens.owneds[i] > prevTokenId);
            /** @dev Check if tokenId is owned by the user.*/
            require(
                msg.sender == cvgControlTower.lockingPositionManager().ownerOf(_ownedAndDelegatedTokens.owneds[i]),
                "TOKEN_NOT_OWNED"
            );
            tokenOwnedAndDelegated[msg.sender].owneds.push(_ownedAndDelegatedTokens.owneds[i]);
            unchecked {
                ++i;
            }
        }
```
Similarly for the other 2 arrays

# <a id='M-01'></a>M-01 Division difference can result in a revert when claiming treasury yield and excess rewards to some users

## Vulnerability Detail

`ysTotal` is calculated differently when adding to `totalSuppliesTracking` and when computing `balanceOfYsCvgAt`.
When adding to `totalSuppliesTracking`, the calculation of `ysTotal` is as follows:

```
        uint256 cvgLockAmount = (amount * ysPercentage) / MAX_PERCENTAGE;
        uint256 ysTotal = (lockDuration * cvgLockAmount) / MAX_LOCK;
```

In `balanceOfYsCvgAt`, `ysTotal` is calculated as follows

```
        uint256 ysTotal = (((endCycle - startCycle) * amount * ysPercentage) / MAX_PERCENTAGE) / MAX_LOCK;
```

This difference allows the `balanceOfYsCvgAt` to be greater than what is added to `totalSuppliesTracking`

### POC
```
  startCycle 357
  endCycle 420
  lockDuration 63
  amount 2
  ysPercentage 80
```

Calculation in `totalSuppliesTracking` gives:
```
        uint256 cvgLockAmount = (2 * 80) / 100; == 1
        uint256 ysTotal = (63 * 1) / 96; == 0
```
Calculation in `balanceOfYsCvgAt` gives:
```
        uint256 ysTotal = ((63 * 2 * 80) / 100) / 96; == 10080 / 100 / 96 == 1
```

### Example Scenario
Alice, Bob and Jake locks cvg for 1 TDE and obtains rounded up `balanceOfYsCvgAt`. A user who is aware of this issue can exploit this issue further by using `increaseLockAmount` with small amount values by which the total difference difference b/w the user's calculated `balanceOfYsCvgAt` and the accounted amount in `totalSuppliesTracking` can be increased. Bob and Jake claims the reward at the end of reward cycle. When Alice attempts to claim rewards, it reverts since there is not enough reward to be sent.

## Impact

This breaks the shares accounting of the treasury rewards. Some user's will get more than the actual intended rewards while the last withdrawals will result in a revert

## Code Snippet

`totalSuppliesTracking` calculation

In `mintPosition`
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L261-L263

In `increaseLockAmount`
https://github.com/sherlock-audit/2023-11-convergence/blob/e894be3e36614a385cf409dc7e278d5b8f16d6f2/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L339-L345

In `increaseLockTimeAndAmount`
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L465-L470

`_ysCvgCheckpoint`
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L577-L584

`balanceOfYsCvgAt` calculation
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Locking/LockingPositionService.sol#L673-L675

## Tool used

Manual Review

## Recommendation

Perform the same calculation in both places

```diff
+++                     uint256 _ysTotal = (_extension.endCycle - _extension.cycleId)* ((_extension.cvgLocked * _lockingPosition.ysPercentage) / MAX_PERCENTAGE) / MAX_LOCK;
---     uint256 ysTotal = (((endCycle - startCycle) * amount * ysPercentage) / MAX_PERCENTAGE) / MAX_LOCK;
```

# <a id='M-02'></a>M-02 cvgRewards may be incorrectly calculated due to possible changes in gagueWeights and totalWeight

## Vulnerability Detail

The `totalWeight` is cached before the individual guage weights are captured to give out the rewards. Hence any change in gaugeWeight's between these can cause the rewards to be sent out incorrectly. The voting method to change the weights is prevented but there are other ways that can alter the weight albeit at low likelihood.

1. The `_setTotalWeight` and `_distributeCvgRewards` functions are called in different blocks there is a WEEK occuring b/w these blocks. In this case if the `checkpoint_gauge()` function is called on the gaugeController, the weight sums of the gauges will be lower than the previously computed `totalWeight` leading to less cvg rewards being sent out.
2. The `_setTotalWeight` and `_distributeCvgRewards` functions are called in different transactions there is a gauge is added / killed in b/w these blocks. This will also cause the gauge weight sum to be different from the totalWeight previosuly cached casuing less / more cvg inflation for the cycle. 

## Impact

cvg inlation may be different than expected if timing is not handled correctly

## Code Snippet

cached `totalWeightLocked` is used
https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Rewards/CvgRewards.sol#L289

## Tool used

Manual Review

## Recommendation

Be aware of the timing issues mentioned above and schedule actions accordingly

# <a id='M-03'></a>M-03 Incorrect slippage protection for cvgSdt/sdt conversion

## Vulnerability Detail

The minimum amount required is incorrectly passed as the `current expected swap amount` instead of `input amount` when exchanging from sdt to cvgSdt in the `_withdrawRewards()` function
```solidity
                        ICrvPoolPlain _poolCvgSDT = poolCvgSDT;
                        /// @dev Only swap if the returned amount in CvgSdt is gretear than the amount rewarded in SDT
                        _poolCvgSDT.exchange(0, 1, rewardAmount, _poolCvgSDT.get_dy(0, 1, rewardAmount), receiver);
```

## Impact

User's who opt to convert sdt to cvgSdt when cliaming rewards will be front-runned causing loss of funds

## Code Snippet

https://github.com/sherlock-audit/2023-11-convergence/blob/main/sherlock-cvg/contracts/Staking/StakeDAO/SdtRewardReceiver.sol#L233C1-L235

## Tool used

Manual Review

## Recommendation

Pass rewardAmount instead
```diff
+++ _poolCvgSDT.exchange(0, 1, rewardAmount, rewardAmount, receiver);
--- _poolCvgSDT.exchange(0, 1, rewardAmount, _poolCvgSDT.get_dy(0, 1, rewardAmount), receiver);
```
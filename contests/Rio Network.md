# Rio Network 

Type: CONTEST

Dates: 20 Feb 2024 - 7 March 2024

Rio is a liquid restaking network built on top of Eigenlayer

Code Under Review: https://github.com/sherlock-audit/2024-02-rio-network-core-protocol

More contest details: https://audits.sherlock.xyz/contests/176

Rank: 3

# Findings Summary

- High : 5
- Medium : 6


|ID|Title|Severity|
|-|:-|:-:|
| [H-01](#H-01)| Validator keys are loaded incorrectly from storage to memory | High |
| [H-02](#H-02)| Epoch is not incremented when current epoch settlement is queued | High |
| [H-03](#H-03)| Operators can steal ETH by front running validator registration | High |
| [H-04](#H-04)| Operators undelegating via EigenLayer is not handled | High |
| [H-05](#H-05)| Deactivating operators doesn't clear it's strategy allocations | High |
| [M-01](#M-01)| TranferETH gas limitation of 10k is not enough | Medium |
| [M-02](#M-02)| Min excess scrape amount can cause unused ETH and possbily lost LRT tokens for users | Medium |
| [M-03](#M-03)| Operators can cause verification of other operators to fail by verifying a validator that was added outside Rio | Medium |
| [M-04](#M-04)| Strict check for precalculated shares to equal the actual shares received will revert often due to rounding in eigenlayer | Medium |
| [M-05](#M-05)| Shares associated with operator exits are not marked unwithdrawable | Medium |
| [M-06](#M-06)| Withdrawals close to maximum amount can revert | Medium |


# Detailed Findings

# <a id='H-01'></a>H-01 Validator keys are loaded incorrectly from storage to memory

## Vulnerability Detail
Validator public keys are loaded from storage to memory incorrectly inside the `swapValidatorDetails` function.

The validator keys (48byte) are stored in storage as:
> slotA : first 32bytes, sloat(A+1) : remaining 16bytes right padded

```solidity
    function saveValidatorDetails(
        bytes32 position,
        uint8 operatorId,
        uint256 startIndex,
        uint256 keysCount,
        bytes memory pubkeys,
        bytes memory signatures
    ) internal returns (uint40) {
        
        ....

        for (uint256 i; i < keysCount;) {
            currentOffset = position.computeStorageKeyOffset(operatorId, startIndex);
            assembly {
                let _ofs := add(add(pubkeys, 0x20), mul(i, 48)) // PUBKEY_LENGTH = 48
                let _part1 := mload(_ofs) // bytes 0..31
                let _part2 := mload(add(_ofs, 0x10)) // bytes 16..47
                isEmpty := iszero(or(_part1, _part2))
                mstore(add(tempKey, 0x30), _part2) // Store 2nd part first
                mstore(add(tempKey, 0x20), _part1) // Store 1st part with overwrite bytes 16-31
            }


            if (isEmpty) revert EMPTY_KEY();
            assembly {
                // Store key
                sstore(currentOffset, mload(add(tempKey, 0x20))) // Store bytes 0..31
                sstore(add(currentOffset, 1), shl(128, mload(add(tempKey, 0x30)))) // Store bytes 32..47
```

This key is copied to memory as follows:

```solidity
    function swapValidatorDetails(
        bytes32 position,
        uint8 operatorId,
        uint256 startIndex1,
        uint256 startIndex2,
        uint256 keysCount
    ) internal {
        
        .....

        for (uint256 i; i < keysCount;) {
            keyOffset1 = position.computeStorageKeyOffset(operatorId, startIndex1);
            keyOffset2 = position.computeStorageKeyOffset(operatorId, startIndex2);
            assembly {
                // Load key1 into memory
                let _part1 := sload(keyOffset1) // Load bytes 0..31
                let _part2 := sload(add(keyOffset1, 1)) // Load bytes 32..47
                mstore(add(key1, 0x20), _part1) // Store bytes 0..31
=>              mstore(add(key1, 0x30), shr(128, _part2)) // Store bytes 16..47
```

This is incorrect as the second mstore will overwrite the second half of the first 32bytes with 0.
This incorrect key is then written to the new position and will act as the public key for that validator. 

## Impact
1. In case this validator gets exited out of order, it would not be possible to report it since the public key is incorrect and the check with EigenLayer will revert. 
2. ETH deallocations could fail since the emitted event will contain invalid public key making the operator unable to process

## Code Snippet
saveValidatorDetails function
https://github.com/sherlock-audit/2024-02-rio-network-core-protocol/blob/4f01e065c1ed346875cf5b05d2b43e0bcdb4c849/rio-sherlock-audit/contracts/utils/ValidatorDetails.sol#L63-L95

incorrectly loaded inside swapValidatorDetails
https://github.com/sherlock-audit/2024-02-rio-network-core-protocol/blob/4f01e065c1ed346875cf5b05d2b43e0bcdb4c849/rio-sherlock-audit/contracts/utils/ValidatorDetails.sol#L148-L151

## Tool used

Manual Review

## Recommendation
Change the ordering to:
```solidity
                mstore(add(key1, 0x30), shr(128, _part2))
                mstore(add(key1, 0x20), _part1)
```

# <a id='H-02'></a>H-02 Epoch is not incremented when current epoch settlement is queued

## Vulnerability Detail
The `queueCurrentEpochSettlement` function doesn't update the current epoch. Since the entire withdrawal process is epoch-based this breaks the entire withdrawal process.

For eg:
1. Newer withdrawal requests will be included to the same epoch, which if settled from eigenlayer will cause loss of funds for the user's since the amount of assets received will not cover this newly owed shares. This will also falsely decrease the amount of shares held by protocol.
2. If the epoch is settled from eigenlayer, all future withdrawals will fail since the same epoch cannot be settled twice.
3. If the epoch is settled using deposits, the withdrawal amount from eigenlayer will be lost since the same epoch cannot be settled twice.  

## Impact
Lost funds, bricked withdrawal process

## Code Snippet
queueCurrentEpochSettlement function
https://github.com/sherlock-audit/2024-02-rio-network-core-protocol/blob/4f01e065c1ed346875cf5b05d2b43e0bcdb4c849/rio-sherlock-audit/contracts/restaking/RioLRTWithdrawalQueue.sol#L177-L209

## Tool used

Manual Review

## Recommendation
Increment the epoch. But when doing this, the owed shares of the epoch should be handled as else it will result in user's loosing their funds by queuing more than withdrawable amounts

# <a id='H-03'></a>H-03 Operators can steal ETH by front running validator registration

## Vulnerability Detail
For pre-existing validators, further staking via the beacon deposit contract act as topups and [doesn't verify the withdrawal credentials and signature](https://eth2book.info/capella/part2/deposits-withdrawals/deposit-processing/#validator-top-ups). This allows an operator to steal the about to be staked ETH by adding the validator beforehand with a differnt withdrawal address controlled by the operator. The security review timeframe of 24hrs in which the admin has control to remove the validators can be bypassed by only adding the validator after this confirmation timestamp.

## Impact
Operators can steal the about to be staked ETH

## Code Snippet
stakes the entire 32 eth
https://github.com/sherlock-audit/2024-02-rio-network-core-protocol/blob/4f01e065c1ed346875cf5b05d2b43e0bcdb4c849/rio-sherlock-audit/contracts/restaking/RioLRTOperatorDelegator.sol#L204

## Tool used
Manual Review

## Recommendation
Make an initial deposit, ideally with 1 ETH from operators and verify the withdrawal credentials before staking user's ETH / Acknowledge operator trust and handle this via some other punishment mechanism for the operator

# <a id='H-04'></a>H-04 Operators undelegating via EigenLayer is not handled

## Vulnerability Detail
EigenLayer's DelegationManager has a feature that allows operators to [undelegate](https://github.com/Layr-Labs/eigenlayer-contracts/blob/6de01c6c16d6df44af15f0b06809dc160eac0ebf/src/contracts/core/DelegationManager.sol#L211) a staker

```solidity
    function undelegate(address staker) external onlyWhenNotPaused(PAUSED_ENTER_WITHDRAWAL_QUEUE) returns (bytes32[] memory withdrawalRoots) {
        require(isDelegated(staker), "DelegationManager.undelegate: staker must be delegated to undelegate");
        require(!isOperator(staker), "DelegationManager.undelegate: operators cannot be undelegated");
        require(staker != address(0), "DelegationManager.undelegate: cannot undelegate zero address");
        address operator = delegatedTo[staker];
        require(
            msg.sender == staker ||
                msg.sender == operator ||
                msg.sender == _operatorDetails[operator].delegationApprover,
            "DelegationManager.undelegate: caller cannot undelegate staker"
        );

        ......

        if (strategies.length == 0) {
            withdrawalRoots = new bytes32[](0);
        } else {
            withdrawalRoots = new bytes32[](strategies.length);
            for (uint256 i = 0; i < strategies.length; i++) {
                IStrategy[] memory singleStrategy = new IStrategy[](1);
                uint256[] memory singleShare = new uint256[](1);
                singleStrategy[0] = strategies[i];
                singleShare[0] = shares[i];

                withdrawalRoots[i] = _removeSharesAndQueueWithdrawal({
                    staker: staker,
                    operator: operator,
                    withdrawer: staker,
                    strategies: singleStrategy,
                    shares: singleShare
                });
            }
        }
```

This queues a withdrawalRequest for the entire staked amount with the staker ie. in this case the OperatorDelegator contract, as the withdrawer. Hence the tokens would be going to the OperatorDelegator contract but it has no functionality to access these funds. In case of ETH, the passed amount will be seen as reward and split accordingly.
Also this would break the shares accounting as these shares are no longer availabe for withdrawal.

## Impact
Operators can cause entire staked tokens to be lost

## Code Snippet
eigenLayers undelegate functionality sets the staker as the withdrawer
https://github.com/Layr-Labs/eigenlayer-contracts/blob/6de01c6c16d6df44af15f0b06809dc160eac0ebf/src/contracts/core/DelegationManager.sol#L247-L253

## Tool used

Manual Review

## Recommendation
Add functionality to handle these funds / acknowledge this operator trust

# <a id='H-05'></a>H-05 Deactivating operators doesn't clear it's strategy allocations

## Vulnerability Detail
When operators are deactivated, its strategy allocations are not cleared although its cap and utilization is handled.

Updates the cap by calling setOperatorStrategyCap
```solidity
    function deactivateOperator(
        RioLRTOperatorRegistryStorageV1.StorageV1 storage s,
        IRioLRTAssetRegistry assetRegistry,
        uint8 operatorId
    ) external {
        
        ....

        for (uint256 i = 0; i < strategies.length; ++i) {
=>          s.setOperatorStrategyCap(
                operatorId, IRioLRTOperatorRegistry.StrategyShareCap({strategy: strategies[i], cap: 0})
            );
        }
```

setOperatorStrategyCap doesn't clear allocation
```solidity
    function setOperatorStrategyCap(
        RioLRTOperatorRegistryStorageV1.StorageV1 storage s,
        uint8 operatorId,
        IRioLRTOperatorRegistry.StrategyShareCap memory newShareCap
    ) internal {
        
        ....

        if (currentShareDetails.cap > 0 && newShareCap.cap == 0) {
            // If the operator has allocations, queue them for exit.
=>          if (currentShareDetails.allocation > 0) {
                operatorDetails.queueOperatorStrategyExit(operatorId, newShareCap.strategy);
            }
            // Remove the operator from the utilization heap.
            utilizationHeap.removeByID(operatorId);
        } else if (currentShareDetails.cap == 0 && newShareCap.cap > 0) {
```

If this operator is re-activated, it will incorrectly have the previous uncleared amount of allocation as its current allocation which will cause the strategy shares allocation and deallocation functions to work incorrectly. 
For eg, if the new cap is set to an amount lower than the previous allocation, then it will disallow all deposits to eigen layer for the strategy as this operator could be on the top of the heap (its utilization will be set to 0 when reactivating) and the allocation function will exit without deposting any amount to EigenLayer.   

It would also not be possible to further deactivate this operator since when deactivating, the wrong allocation amount will make it attempt to withdraw that much amount of shares from EigenLayer which will cause it to revert

## Impact
If an operator is deactivated and reactivated again, 
1. Deposits will fail
2. The operator cannot be deactivated again

## Code Snippet
deactivateOperator calls setOperatorStrategyCap to update cap and doesn't clear allocation
https://github.com/sherlock-audit/2024-02-rio-network-core-protocol/blob/4f01e065c1ed346875cf5b05d2b43e0bcdb4c849/rio-sherlock-audit/contracts/utils/OperatorRegistryV1Admin.sol#L112-L137

setOperatorStrategyCap
https://github.com/sherlock-audit/2024-02-rio-network-core-protocol/blob/4f01e065c1ed346875cf5b05d2b43e0bcdb4c849/rio-sherlock-audit/contracts/utils/OperatorRegistryV1Admin.sol#L231-L255

## Tool used

Manual Review

## Recommendation
When deactivating operators, clear its allocation

# <a id='M-01'></a>M-01 TranferETH gas limitation of 10k is not enough

## Vulnerability Detail
The `transferETH` function has gas limit set to 10k. But this is not enough and will revert.

When OperatorDelegator receives ETH (eg:partial ETH withdrawals from eigenPod), it uses transferETH to send the funds to `rewardDistributor`. But reward distributor consumes more than 10k gas in its receive() function and will cause the call to fail

```solidity
    function transferETH(address recipient, uint256 amount) internal {
        (bool success,) = recipient.call{value: amount, gas: 10_000}('');
        if (!success) {
            revert ETH_TRANSFER_FAILED();
        }
    }
```

### POC

Apply the diff and run `forge test --mt testHash_RecieveRevertsLackOfGas -vv`

```diff
diff --git a/rio-sherlock-audit/test/RioLRTOperatorDelegator.t.sol b/rio-sherlock-audit/test/RioLRTOperatorDelegator.t.sol
index 89645c0..3d8d630 100644
--- a/rio-sherlock-audit/test/RioLRTOperatorDelegator.t.sol
+++ b/rio-sherlock-audit/test/RioLRTOperatorDelegator.t.sol
@@ -6,6 +6,7 @@ import {IDelegationManager} from 'contracts/interfaces/eigenlayer/IDelegationMan
 import {RioLRTOperatorDelegator} from 'contracts/restaking/RioLRTOperatorDelegator.sol';
 import {BEACON_CHAIN_STRATEGY, ETH_ADDRESS} from 'contracts/utils/Constants.sol';
 import {Array} from 'contracts/utils/Array.sol';
+import "forge-std/Test.sol";
 
 contract RioLRTOperatorDelegatorTest is RioDeployer {
     using Array for *;
@@ -38,6 +39,20 @@ contract RioLRTOperatorDelegatorTest is RioDeployer {
         assertEq(address(delegatorContract.eigenPod()).balance, 0);
     }
 
+    function testHash_RecieveRevertsLackOfGas() public {
+        uint8 operatorId = addOperatorDelegator(reETH.operatorRegistry, address(reETH.rewardDistributor));
+        RioLRTOperatorDelegator delegatorContract =
+            RioLRTOperatorDelegator(payable(reETH.operatorRegistry.getOperatorDetails(operatorId).delegator));
+
+            // receive of delegator reverts due to only sending 10k gas
+            (bool success,bytes memory errorData) = address(delegatorContract).call{value:1 ether}("");
+            
+            assert(!success);
+
+            // error ETH_TRANSFER_FAILED(), 98ce269a
+            console.logBytes(errorData);
+    }
+
     function test_scrapeExcessFullWithdrawalETHFromEigenPod() public {
         uint8 operatorId = addOperatorDelegator(reETH.operatorRegistry, address(reETH.rewardDistributor));
         address operatorDelegator = reETH.operatorRegistry.getOperatorDetails(operatorId).delegator;

```
## Impact
Lost ETH since it cannot be received by OperatorDelegator

## Code Snippet
10k limit
https://github.com/sherlock-audit/2024-02-rio-network-core-protocol/blob/4f01e065c1ed346875cf5b05d2b43e0bcdb4c849/rio-sherlock-audit/contracts/utils/Asset.sol#L41-L46

usage of transferETH to send ETH in a reverting case
https://github.com/sherlock-audit/2024-02-rio-network-core-protocol/blob/4f01e065c1ed346875cf5b05d2b43e0bcdb4c849/rio-sherlock-audit/contracts/restaking/RioLRTOperatorDelegator.sol#L244-L246

## Tool used

Manual Review

## Recommendation
Increase the gas limit

# <a id='M-02'></a>M-02 Min excess scrape amount can cause unused ETH and possbily lost LRT tokens for users

## Vulnerability Detail
When scraping full withdrawal ETH, it is required for the excess amount to be greater than or equal to 1 ether. 

```solidity
    uint256 internal constant MIN_EXCESS_FULL_WITHDRAWAL_ETH_FOR_SCRAPE = 1 ether;

    ...

    function scrapeExcessFullWithdrawalETHFromEigenPod() external {
        uint256 ethWithdrawable = eigenPod.withdrawableRestakedExecutionLayerGwei().toWei();
        uint256 ethQueuedForWithdrawal = getETHQueuedForWithdrawal();
=>      if (ethWithdrawable <= ethQueuedForWithdrawal + MIN_EXCESS_FULL_WITHDRAWAL_ETH_FOR_SCRAPE) {
            revert INSUFFICIENT_EXCESS_FULL_WITHDRAWAL_ETH();
        }
        _queueWithdrawalForOperatorExitOrScrape(BEACON_CHAIN_STRATEGY, ethWithdrawable - ethQueuedForWithdrawal);
    }
```

Hence if the amount is less 1 ether, it cannot be scraped back to deposit pool. This will make this amount unusable if the operator doesn't have other validators that might eventually make this amount usable (eg: if the operator is deactivated). But since this amount will still be included in the total ETH balance calculation of the protocol, it is possible for users to attempt withdrawals such that this amount is required for fulfilling. In such a case, these user's LRT tokens can be effectively lost. Depending on the number of eigen pods having such amounts the loss can become significant

## Impact
1. Unused ETH
2. Possibly locked LRT tokens of user's

## Code Snippet
Min quantity check when scraping full eth withdrawals
https://github.com/sherlock-audit/2024-02-rio-network-core-protocol/blob/4f01e065c1ed346875cf5b05d2b43e0bcdb4c849/rio-sherlock-audit/contracts/restaking/RioLRTOperatorDelegator.sol#L160-L167

## Tool used
Manual Review

## Recommendation
Avoid the min quantity check

# <a id='M-03'></a>M-03 Operators can cause verification of other operators to fail by verifying a validator that was added outside Rio

## Vulnerability Detail
The `verifyWithdrawalCredentials` function decreases the unverified ETH validator balance once the verification in eigenlayer proves successful  

```solidity
    function verifyWithdrawalCredentials(
        uint8 operatorId,
        uint64 oracleTimestamp,
        IBeaconChainProofs.StateRootProof calldata stateRootProof,
        uint40[] calldata validatorIndices,
        bytes[] calldata validatorFieldsProofs,
        bytes32[][] calldata validatorFields
    ) external onlyOperatorManagerOrProofUploader(operatorId) {

        ....

=>      assetRegistry().decreaseUnverifiedValidatorETHBalance(validatorIndices.length * ETH_DEPOSIT_SIZE);


        emit OperatorWithdrawalCredentialsVerified(operatorId, oracleTimestamp, validatorIndices);
    }
```

This assumes that the corresponding amount of ETH has been added to `unverifiedValidatorETHBalance` beforehand when the deposit from validator was made. But this need not be the case.
An operator can add a validator from outside Rio and have the eigenpod as the withdrawal address. This will make the verification in eigenpod pass but will not have increased the `unverifiedValidatorETHBalance`. 
This will cause the verification of latest operators to fail since `unverifiedValidatorETHBalance` would underflow, effectively loosing these ETH until another deposit increases the `unverifiedValidatorETHBalance` and takes the earlier operators place

But this would require an operator to sacrifice atleast 1 ETH (donating to eigenpod) in exchange for the grief.

## Impact
Operators can grief last verifications. In case the 

## Code Snippet
https://github.com/sherlock-audit/2024-02-rio-network-core-protocol/blob/4f01e065c1ed346875cf5b05d2b43e0bcdb4c849/rio-sherlock-audit/contracts/restaking/RioLRTOperatorRegistry.sol#L236C14-L253

## Tool used

Manual Review

## Recommendation
Acknowledge the operator trust / only allow proof uploader to make the call

# <a id='M-04'></a>M-04 Strict check for precalculated shares to equal the actual shares received will revert often due to rounding in eigenlayer

## Vulnerability Detail
Deposits to EigenLayer will revert in case the percalculated shares is not equal to the actual received shares

```solidity
    function depositTokenToOperators(
        IRioLRTOperatorRegistry operatorRegistry,
        address token,
        address strategy,
        uint256 sharesToAllocate
    ) internal returns (uint256 sharesReceived) {
        (uint256 sharesAllocated, IRioLRTOperatorRegistry.OperatorStrategyAllocation[] memory  allocations) = operatorRegistry.allocateStrategyShares(
            strategy, sharesToAllocate
        );


        for (uint256 i = 0; i < allocations.length; ++i) {
            IRioLRTOperatorRegistry.OperatorStrategyAllocation memory allocation = allocations[i];


            IERC20(token).safeTransfer(allocation.delegator, allocation.tokens);
            sharesReceived += IRioLRTOperatorDelegator(allocation.delegator).stakeERC20(strategy, token, allocation.tokens);
        }
=>      if (sharesReceived != sharesAllocated) revert INCORRECT_NUMBER_OF_SHARES_RECEIVED();
    }
```

The relevant flow is as follows:
1. Given `sharesToAllocate`, find the amount of tokens that is to be staked in order to receive this amount (or max possible amount) of shares. This is done by calling the `sharesToUnderlyingView` function in EigenLayer's strategy.

```solidity
function sharesToUnderlyingView(uint256 amountShares) public view virtual override returns (uint256) {
        
        uint256 virtualTotalShares = totalShares + SHARES_OFFSET;
        uint256 virtualTokenBalance = _tokenBalance() + BALANCE_OFFSET;
        
=>      return (virtualTokenBalance * amountShares) / virtualTotalShares;
    }
```

2. Deposit to EigenLayer with the earlier calculated amount of tokens. Corresponding shares are calculated inside the deposit function as follows:

```solidity
function deposit(
        IERC20 token,
        uint256 amount
    ) external virtual override onlyWhenNotPaused(PAUSED_DEPOSITS) onlyStrategyManager returns (uint256 newShares) {
        
        ....

        uint256 virtualShareAmount = priorTotalShares + SHARES_OFFSET;
        uint256 virtualTokenBalance = _tokenBalance() + BALANCE_OFFSET;
        
        uint256 virtualPriorTokenBalance = virtualTokenBalance - amount;
=>      newShares = (amount * virtualShareAmount) / virtualPriorTokenBalance;

       .....
    }
```

Due to rounding inside both the functions the following is possible often:
1. sharesToAllocate = s
2. Obtained token amount = a
3. sharesReceived for depositing `a` amount of tokens = d
4. d < s

This will cause the call to revert

### POC
Add the following lines of code to `test/RioLRTDepositPool.t.sol` and run `forge test --mt testHash_depositRevertDueToRounding`

```solidity
    error INCORRECT_NUMBER_OF_SHARES_RECEIVED();
    function testHash_depositRevertDueToRounding() public {

        uint256 amount = 1e18;

        addOperatorDelegators(reLST.operatorRegistry, address(reLST.rewardDistributor), 10);
        cbETH.mint(address(reLST.depositPool), amount);

        vm.prank(address(reLST.coordinator));

        (uint256 sharesReceived,) = reLST.depositPool.depositBalanceIntoEigenLayer(CBETH_ADDRESS);
        assertEq(sharesReceived, 1e18);
        assertEq(cbETH.balanceOf(CBETH_STRATEGY), 1e18);

        // now balance = 1e18 and shares = 1e18
        // if 10 cbeth is added to strategy balance, the next deposit will revert due to differences caused by rounding

        cbETH.mint(address(CBETH_STRATEGY), 10);

        cbETH.mint(address(reLST.depositPool), amount);

        vm.prank(address(reLST.coordinator));

        vm.expectRevert(INCORRECT_NUMBER_OF_SHARES_RECEIVED.selector);
        reLST.depositPool.depositBalanceIntoEigenLayer(CBETH_ADDRESS);  
    }
```

## Impact
Deposits will revert often

## Code Snippet
depositTokenToOperators reverts in case the shares due not match exactly
https://github.com/sherlock-audit/2024-02-rio-network-core-protocol/blob/4f01e065c1ed346875cf5b05d2b43e0bcdb4c849/rio-sherlock-audit/contracts/utils/OperatorOperations.sol#L51-L68

allocateStrategyShares uses sharesToUnderlyingView inorder to calculate the amount of tokens 
https://github.com/sherlock-audit/2024-02-rio-network-core-protocol/blob/4f01e065c1ed346875cf5b05d2b43e0bcdb4c849/rio-sherlock-audit/contracts/restaking/RioLRTOperatorRegistry.sol#L342-L363

received shares calculation inside deposit function
https://github.com/Layr-Labs/eigenlayer-contracts/blob/6de01c6c16d6df44af15f0b06809dc160eac0ebf/src/contracts/strategies/StrategyBase.sol#L95-L124

## Tool used

Manual Review

## Recommendation
Currently accurate predictions are necessary due to how the operator utilization is implemented. This would have to be changed to eliminate the accurate precalculation

# <a id='M-05'></a>M-05 Shares associated with operator exits are not marked unwithdrawable

## Vulnerability Detail
When an operator exits, the shares associated with the operator is only cleared once the withdrawal from EigenLayer completes and the assets reach the depositPool

```solidity
    function completeOperatorWithdrawalForAsset(
        address asset,
        uint8 operatorId,
        IDelegationManager.Withdrawal calldata queuedWithdrawal,
        uint256 middlewareTimesIndex
    ) external {
        
        ....
       
        // @audit the associated shares are cleared here

        } else {
            assetRegistry().decreaseSharesHeldForAsset(asset, queuedWithdrawal.shares[0]);
        }

```

During this time period, for user's attempting to withdraw it would seem as if this amount of shares is available to withdraw. This will cause the `rebalance` function to revert until the operator's withdrawal associated amount reaches depositPool. User's withdrawing would expect a total time of (rebalance delay ~1 day + eigen layer delay, max 7 days) to complete the withdrawal. But since the operator's queued withdrawal amount can take ~ 7 days to reach the deposit pool, it is possible for the withdrawal to take ~14 days to complete.  

## Impact
When deposits doesn't cover the entire withdrawal amount, rebalance can revert and cause withdrawals to take twice (ie.~14 days instead of ~7 days) as much time to complete

## Code Snippet
Shares associated with an operator exit is only cleared once the withdrawal is complete
https://github.com/sherlock-audit/2024-02-rio-network-core-protocol/blob/4f01e065c1ed346875cf5b05d2b43e0bcdb4c849/rio-sherlock-audit/contracts/restaking/RioLRTDepositPool.sol#L119-L147

## Tool used

Manual Review

## Recommendation
Account for the currently queued operator withdrawals and decrease it from the withdrawable shares

# <a id='M-06'></a>M-06 Withdrawals close to maximum amount can revert

## Vulnerability Detail
Withdrawals are settled using funds from the depositPool and the earlier acquired shares in EigenLayer. To calculate how much shares is withdrawable, the depositPool token balance is converted to its corresponding share balance using the current exchange rate from EigenLayer. If the shares acquired from previous deposits were `x` and the deposit pool token balance corresponds to `y` shares using the current exchange rate, user's are allowed to request withdrawals for upto `x+y` shares. 
After the withdrawal request from the user, it can take upto rebalance delay ~24hrs amount of time to actually execute the withdrawal. In case the total requested withdrawal amount was close to the maximum amount of withdrawable assets and the exchange rate in EigenLayer changes (one share is worth more now. and hence the `y` amount of shares is now worth more the deposit pool token balance), the rebalance call will revert since the transfer of deposit pool token balance will not cover its associated portion of shares (unless further deposits are made) and there won't be enough shares to withdraw from EigenLayer.    


## Impact
Rebalance and hence withdrawals can revert when close to maximum available amount of shares is withdrawn

## Code Snippet
the transfer from deposit pool is supposed to cover its portion of shares in the total shares owed
https://github.com/sherlock-audit/2024-02-rio-network-core-protocol/blob/4f01e065c1ed346875cf5b05d2b43e0bcdb4c849/rio-sherlock-audit/contracts/restaking/RioLRTCoordinator.sol#L245-L252


## Tool used
Manual Review

## Recommendation
Since this will happen rarely it could be managed by monitoring for this case and making necessary deposits
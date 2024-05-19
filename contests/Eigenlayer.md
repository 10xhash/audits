# Eigenlayer 

Type: CONTEST

Dates: 27 Feb 2024 - 19 Mar 2024

Code Under Review: https://github.com/Layr-Labs/eigenlayer-contracts/tree/6e588701c5f543ae4cd34fe9c6567cc46c7eb722, https://github.com/Layr-Labs/eigenda/tree/91838ba58b8e2525c7fd1e4db5e9903551eed326, https://github.com/Layr-Labs/eigenlayer-middleware/tree/61d554403279826fcbc38d421580811e57d29270

More contest details: https://cantina.xyz/competitions/4b6f08a7-e830-4499-9977-08e2c3b32068

Rank: 1

# Findings Summary

- High : 1
- Medium : 1


|ID|Title|Severity|
|-|:-|:-:|
| [H-01](#H-01)| Beacon Chain withdrawals that occur at lastWithdrawalTimestamp will be lost | High |
| [M-01](#M-01)| Pasued deregistering is bypassed with registering with churn | Medium |


# Detailed Findings

# <a id='H-01'></a>H-01 Beacon Chain withdrawals that occur at lastWithdrawalTimestamp will be lost

## Vulnerability Detail

ETH withdrawn from Beacon Chain is withdrawn from the EigenPod by calling the `withdrawBeforeRestaking` function before restaking is activated. To activate restaking, the `activateRestaking` function is called which internally calls `_processWithdrawalBeforeRestaking` with the idea that all withdrawals till this timestamp will be processed and hence the `verifyAndProcessWithdrawals` function need only be called on timestamps greater than `mostRecentWithdrawalTimestamp` ie. the timestamp in which the last withdrawal via `_processWithdrawalBeforeRestaking` occured.

https://github.com/Layr-Labs/eigenlayer-contracts/blob/6e588701c5f543ae4cd34fe9c6567cc46c7eb722/src/contracts/pods/EigenPod.sol#L566-L583
https://github.com/Layr-Labs/eigenlayer-contracts/blob/6e588701c5f543ae4cd34fe9c6567cc46c7eb722/src/contracts/pods/EigenPod.sol#L119-L126
```solidity
    function _verifyAndProcessWithdrawal(
        bytes32 beaconStateRoot,
        BeaconChainProofs.WithdrawalProof calldata withdrawalProof,
        bytes calldata validatorFieldsProof,
        bytes32[] calldata validatorFields,
        bytes32[] calldata withdrawalFields
    )
        internal
        proofIsForValidTimestamp(withdrawalProof.getWithdrawalTimestamp())
        returns (VerifiedWithdrawal memory)
    {

    modifier proofIsForValidTimestamp(uint64 timestamp) {
        require(
            timestamp > mostRecentWithdrawalTimestamp,
            "EigenPod.proofIsForValidTimestamp: beacon chain proof must be for timestamp after mostRecentWithdrawalTimestamp"
        );
        _;
    }
```

https://github.com/Layr-Labs/eigenlayer-contracts/blob/6e588701c5f543ae4cd34fe9c6567cc46c7eb722/src/contracts/pods/EigenPod.sol#L381-L391
https://github.com/Layr-Labs/eigenlayer-contracts/blob/6e588701c5f543ae4cd34fe9c6567cc46c7eb722/src/contracts/pods/EigenPod.sol#L733-L737
```solidity
    function activateRestaking()
        external
        onlyWhenNotPaused(PAUSED_EIGENPODS_VERIFY_CREDENTIALS)
        onlyEigenPodOwner
        hasNeverRestaked
    {
        hasRestaked = true;
        _processWithdrawalBeforeRestaking(podOwner);


        emit RestakingActivated(podOwner);
    }

    function _processWithdrawalBeforeRestaking(address _podOwner) internal {
        mostRecentWithdrawalTimestamp = uint32(block.timestamp);
        nonBeaconChainETHBalanceWei = 0;
        _sendETH_AsDelayedWithdrawal(_podOwner, address(this).balance);
    }
```

In case there is a withdrawal from beacon chain that occured in `mostRecentWithdrawalTimestamp`, this amount will be lost since the withdrawals from beacon chain are executed after all the user transactions according to [eip-4895](https://eips.ethereum.org/EIPS/eip-4895). Hence when the user executes `activateRestaking` and hence `_processWithdrawalBeforeRestaking`, the pod will not be having the ETH withdrawn from beacon chain in that block. After this block, the user will not be able to withdraw this ETH via `verifyAndProcessWithdrawals` since `verifyAndProcessWithdrawals` can only be called for timestamps greater than `mostRecentWithdrawalTimestamp`

### Example

1. User has a withdrawal of 32 eth from beacon chain at time t
2. User calls activateRestaking in the timestamp t
3. This is supposed to move out all ETH from the contract. But since the withdrawal execution only happens after all the user's transactions, address(this) will not factor this ETH
4. This 32 eth cannot be withdrawn using `verifyAndProcessWithdrawals` since the proof timestamp will be equal to the `mostRecentWithdrawalTimestamp` (timestamp in which acitvateRestaking is called) and the condition for calling `verifyAndProcessWithdrawals` is `timestamp > mostRecentWithdrawalTimestamp`

## Impact

Beacon chain withdrawals will be lost in case any user activates restaking in the block which also contains one of their withdrawal

## Recommendation
Use `>= mostRecentWithdrawalTimestamp` instead

# <a id='M-01'></a>M-01 Pasued deregistering is bypassed with registering with churn

## Vulnerability Detail

Deregistering of operators is supposed to be pausable as indicated by the `onlyWhenNotPaused(PAUSED_REGISTER_OPERATOR)` modifier on the deregister function

```solidity
    function deregisterOperator(
        bytes calldata quorumNumbers
    ) external onlyWhenNotPaused(PAUSED_DEREGISTER_OPERATOR) {
```

But user's can be deregistered even when PAUSED_REGISTER_OPERATOR bit is set by calling the `registerOperatorWithChurn` function 

## Impact

User's can be deregistered even when deregistering is paused 

## Recommendation

In case not intended, check for `onlyWhenNotPaused(PAUSED_REGISTER_OPERATOR)` in the internal _deregister function
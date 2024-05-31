## Low Risk Findings

### [L-01] Nested mappings are not cleared from the storage when `usdUpdateProposal` is deleted

The `usdUpdateProposal` storage variable is of the struct type `USDUpdateProposal`, which contains two storage mappings - [approvals](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/interfaces/ILockManager.sol#L65) and [disapprovals](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/interfaces/ILockManager.sol#L66). When [proposeUSDPrice](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L161), [disapproveUSDPrice](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L238), or [_execUSDPriceUpdate](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L525) are called, `usdUpdateProposal` is deleted. However, nested struct mappings are not deleted.

It is recommended to delete the nested mappings manually before `usdUpdateProposal` deletion.

### [L-02] In the `configureToken` function, there is no check to see if the newly configured `_tokenContract` decimals are not more than 18.

`configuredTokens.decimals` are used in the `getLockedWeightedValue` function to calculate the `lockedWeighted`. The function assumes that all of the `configuredTokens` have decimals fewer than 18. If any tokens exceed 18 decimals, the function will always revert.

```solidity
uint256 deltaDecimal = 10 ** (18 - configuredTokens[configuredTokenContracts[i]].decimals);
```

It should be restricted to configuring `_tokenContract` with decimals > 18.

> According to the README, high decimal ERC20 tokens (decimals > 18) are within the scope, and will be used.
> 

### [L-03] There is no option to remove tokens from the `configuredTokenContracts` array

The [`configureToken`](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L122) function pushes the `_tokenContract` address to the `configuredTokenContracts` array.This array is used in multiple functions ([setLockDuration](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L251), [getLocked](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L433), [getLockedWeightedValue](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L465)) to loop over all configured tokens. However, there is no option to pop out of the array, which could cause problems if the array's length significantly increases.

## **Informational / Non Critical Findings**

### [NC-01] The `snuggeryManager` storage variable is not used. It is recommended to delete it to reduce the contract size.

### [NC-02] When `configureLockdrop` is called the check for the newly configured `lockdrop` period is insufficient

 The check `if (_lockdropData.start >= _lockdropData.end)` is insufficient, because 

### [NC-03] There are repeatable checks in the `LockManager`

To avoid duplicate checks and reduce code complexity, consider extracting modifiers.

The repeated check in [setUSDThresholds](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L133-L134) and [proposeUSDPrice](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L157-L158) can be extracted into a `notActiveProposal` modifier.

```solidity
if (usdUpdateProposal.proposer != address(0))
    revert ProposalInProgressError();
```

The repeated check in [approveUSDPrice](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L191) and [disapproveUSDPrice](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L224) can be extracted into a `activeProposal` modifier.

```solidity
if (usdUpdateProposal.proposer == address(0)) revert NoProposalError();
```
# Quality Assurance for Munchables


## Table of Contents

| Issue ID | Description |
| -------- | ----------- |
| [QA-1](#qa-1-inconsistent-state-in-lockedtokens-mapping-after-unlocking-full-balance) | Inconsistent State in `lockedTokens` Mapping After Unlocking Full Balance |
| [QA-2](#qa-2-unsafe-support-of-non-standard-erc-20-tokens) | Unsafe support of non-standard ERC-20 tokens |
| [QA-3](#qa-3-division-by-0-can-cause-errors-in-lockmanager-contract) | Division by 0 can cause errors in `LockManager` contract |
| [QA-4](#qa-4-incorrect-calculation-of-nft-rewards-in-lock-function) | Incorrect Calculation of NFT Rewards in `_lock` Function |
| [QA-5](#qa-5-inconsistent-behavior-in-getlocked-function-of-lockmanagersol) | Inconsistent Behavior in `getLocked` Function of `LockManager.sol` |
| [QA-6](#qa-6-potential-misleading-data-return-in-getconfiguredtoken-function) | Potential Misleading Data Return in `getConfiguredToken` Function |
| [QA-7](#qa-7-lockdrop-configuration-allows-start-time-in-the-past) | Lockdrop configuration allows start time in the past |
| [QA-8](#qa-8-incompatibility-of-lockmanager-with-blasts-rebasing-eth-and-yield-modes) | Incompatibility of `LockManager` with Blast's Rebasing ETH and Yield Modes |
| [QA-9](#qa-9-gas-issue-in-getlockedweightedvalue-function) | Gas issue in `getLockedWeightedValue` Function |
| [QA-10](#qa-10-lack-of-access-control-on-setusdthresholds-function) | Lack of Access Control on `setUSDThresholds` Function |
| [QA-11](#qa-11-locks-can-be-extended-to-an-old-unlocktime) | Locks can be extended to an old `unlockTime` |


 ## [QA-1]  Inconsistent State in `lockedTokens` Mapping After Unlocking Full Balance

#### Impact
When a user unlocks their entire locked balance for a specific token, the `unlock` function reduces the `lockedToken.quantity` to zero but does not reset the `unlockTime`, `lastLockTime`, and `remainder` fields. This can lead to inconsistencies in the state of the `lockedTokens` mapping, potentially causing confusion or unexpected behavior in other parts of the contract that rely on these values.

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L401-L427

```solidity
function unlock(
    address _tokenContract,
    uint256 _quantity
) external notPaused nonReentrant {
    LockedToken storage lockedToken = lockedTokens[msg.sender][
        _tokenContract
    ];
    // ...

    lockedToken.quantity -= _quantity;

    // ...
}
```

#### Recommended Mitigation Steps:
To maintain a clean state in the `lockedTokens` mapping, add a check after reducing the `lockedToken.quantity` to reset the `unlockTime`, `lastLockTime`, and `remainder` fields when the entire locked balance is unlocked:

```diff
function unlock(
    address _tokenContract,
    uint256 _quantity
) external notPaused nonReentrant {
    // ...

    lockedToken.quantity -= _quantity;

+    if (lockedToken.quantity == 0) {
+        lockedToken.unlockTime = 0;
+        lockedToken.lastLockTime = 0;
+        lockedToken.remainder = 0;
    }

    // ...
}
```
This is to make sure that when a user unlocks their entire locked balance, the corresponding `LockedToken` struct is reset to its default state, maintaining consistency in the `lockedTokens` mapping.



## [QA-2] Unsafe support of non-standard ERC-20 tokens


The `LockManager` contract uses the `transferFrom` function to handle ERC-20 token transfers without checking the return value. While standard ERC-20 tokens revert on a failed transfer, some non-standard tokens (like USDT) may return `false` instead. If such a scenario occurs, the contract assumes the transfer was successful even though it wasn't, potentially leading to incorrect state updates and loss of funds. This issue affects the integrity and reliability of the contract's operations involving token transfers.


For now, USDB and WETH are the ERC2O used by the protocol but for this contest, it states that more tokens will be added or used in future according to the [README](https://github.com/code-423n4/2024-05-munchables#scoping-q--a),

|Question	|Answer|
|-|-|
|ERC20 used by the protocol	| USDB, WETH, (assume we can add more to the future for use in LockManager)|

> A more detailed explanation of this issue and its impacts has been reported under H/M 

#### Proof of Concept

The vulnerable code snippet is located within the `_lock` function of the `LockManager` contract:

```solidity
// Transfer erc tokens
if (_tokenContract!= address(0)) {
    IERC20 token = IERC20(_tokenContract);
    token.transferFrom(_tokenOwner, address(this), _quantity);
}
```

This code attempts to transfer ERC-20 tokens from `_tokenOwner` to the contract itself (`address(this)`). It does not check the return value of `transferFrom`, which means that if the transfer fails silently (returns `false`), the contract will proceed as if the transfer were successful.

#### Recommended Mitigation Steps:
We recommend using `SafeERC20`'s library by OZ.




## [QA-3] Division by 0 can cause errors in LockManager contract

`_lock` function  in the `LockManager` contract performs a division operation using `configuredToken.nftCost` as the denominator without checking if it is zero. If `configuredToken.nftCost` is zero, this will cause the function to revert, potentially freezing the locking mechanism for tokens.

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L363-L364

```solidity
uint256 remainder = quantity % configuredToken.nftCost;
uint256 numberNFTs = (quantity - remainder) / configuredToken.nftCost;
```

If `configuredToken.nftCost` is zero, the division operation will revert, causing the function to fail.

#### Recommended Mitigation Steps:
Add a check to ensure that `configuredToken.nftCost` is not zero before performing the division. 

```diff
+ if (configuredToken.nftCost == 0) revert NFTCostInvalidError(); 
  uint256 remainder = quantity % configuredToken.nftCost;
   uint256 numberNFTs = (quantity - remainder) / configuredToken.nftCost;
```





## [QA-4] Incorrect Calculation of NFT Rewards in `_lock` Function

The current implementation of the `_lock` function in the `LockManager` contract incorrectly calculates the number of NFT rewards (`numberNFTs`) a player should receive based on the locked amount (`quantity`). Specifically, the calculation subtracts the remainder from the total quantity before dividing by the cost per NFT (`configuredToken.nftCost`), which can result in players receiving fewer NFTs than they are entitled to if the quantity is not an exact multiple of the NFT cost. This bug could lead to unfair distribution of rewards and potential loss of funds.

#### Proof of Concept

Take a look at this: https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L362-L369

```solidity
// calculate number of nfts
remainder = quantity % configuredToken.nftCost;
numberNFTs = (quantity - remainder) / configuredToken.nftCost;

if (numberNFTs > type(uint16).max) revert TooManyNFTsError();

// Tell nftOverlord that the player has new unopened Munchables
nftOverlord.addReveal(_lockRecipient, uint16(numberNFTs));
```

As seen above, the subtraction of `remainder` from `quantity` before the division leads to an incorrect calculation of `numberNFTs` when `quantity` is not exactly divisible by `configuredToken.nftCost`.

#### Recommended Mitigation Steps
To resolve this issue, the calculation of `numberNFTs` should be adjusted to ensure that the division occurs before accounting for the remainder. This adjustment will guarantee that players receive the correct number of NFTs based on the locked amount. Something like this:

```diff
- remainder = quantity % configuredToken.nftCost;
- numberNFTs = quantity / configuredToken.nftCost;
+ numberNFTs = quantity / configuredToken.nftCost;
+ remainder = quantity % configuredToken.nftCost;

if (numberNFTs > type(uint16).max) revert TooManyNFTsError();

// Notify the nftOverlord about the newly revealed Munchables for the player
nftOverlord.addReveal(_lockRecipient, uint16(numberNFTs));
```






## [QA-5] Inconsistent Behavior in `getLocked` Function of `LockManager.sol`

The `getLocked` function in the `LockManager.sol` contract incorrectly reports locked token metadata for all configured tokens, regardless of whether the player has actually locked any tokens of those types. This can lead to misleading information being returned to the caller, such as indicating that a player has locked tokens of a certain type when they haven't. It also inefficiently allocates memory for tokens that are not locked by the player.

#### Proof of Concept
Look at this part: https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L438-L456

```solidity
for (uint256 i; i < configuredTokensLength; i++) {
    LockedToken memory tmpLockedToken;
    tmpLockedToken.unlockTime = lockedTokens[_player][configuredTokenContracts[i]].unlockTime;
    tmpLockedToken.quantity = lockedTokens[_player][configuredTokenContracts[i]].quantity;
    tmpLockedToken.lastLockTime = lockedTokens[_player][configuredTokenContracts[i]].lastLockTime;
    tmpLockedToken.remainder = lockedTokens[_player][configuredTokenContracts[i]].remainder;
    tmpLockedTokens[i] = LockedTokenWithMetadata(tmpLockedToken, configuredTokenContracts[i]);
}
```

This loop iterates over all configured token contracts and populates the `tmpLockedTokens` array with metadata for each, without checking if the player has any locked tokens of the current token type.

#### Recommended Mitigation Steps:

1. **Filter Out Unlocked Tokens**: Modify the `getLocked` function to only include token types in the returned array if the player has actually locked tokens of those types.

2. **Dynamic Memory Allocation**: Adjust the memory allocation for the `tmpLockedTokens` array based on the actual number of locked tokens the player has, rather than the total number of configured tokens.

3. **Optimize Performance**: Implement a two-step process where the first step counts the number of locked token types for the player, and the second step allocates memory and populates the array based on this count.






## [QA-6] Potential Misleading Data Return in `getConfiguredToken` Function

The `getConfiguredToken` function returns a `ConfiguredToken` struct for any given `_tokenContract` address, including those that are not actually configured. This could lead to incorrect assumptions about the state of a token, as the function would return a struct with default values (e.g., `false` for `active`, `0` for `nftCost`) for non-configured tokens. This could potentially confuse users or smart contracts interacting with the `LockManager` contract, leading to unexpected behavior or decisions based on inaccurate data.

#### Proof of Concept
https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L490-L494

```solidity
function getConfiguredToken(address _tokenContract) external view returns (ConfiguredToken memory _token) {
    _token = configuredTokens[_tokenContract];
}
```

Given this implementation, calling `getConfiguredToken` with an address that does not correspond to a configured token will still return a `ConfiguredToken` struct, albeit with default values.

#### Recommended Mitigation Steps:

Either revert for Non-Configured Tokens:
Modify the `getConfiguredToken` function to revert if the provided `_tokenContract` is not configured. This ensures that the function only returns valid `ConfiguredToken` structs for configured tokens.

```solidity
function getConfiguredToken(address _tokenContract) external view returns (ConfiguredToken memory _token) {
    _token = configuredTokens[_tokenContract];
    require(_token.nftCost!= 0, "TokenNotConfiguredError");
}
```

OR Return Configuration Status:
Modify the function to return a boolean value indicating whether the token is configured or not, along with the `ConfiguredToken` struct. This allows the caller to verify the validity of the returned struct.

```solidity
function getConfiguredToken(address _tokenContract) external view returns (bool _isConfigured, ConfiguredToken memory _token) {
    _token = configuredTokens[_tokenContract];
    _isConfigured = (_token.nftCost!= 0);
}
```






## [QA-7] Lockdrop configuration allows start time in the past

The `configureLockdrop` function does not validate if the `start` time of the lockdrop is in the past. This could lead to a situation where a lockdrop is configured with a start time that has already passed, which might not be the intended behavior and could cause unexpected issues in the lockdrop process.

#### Proof of Concept

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L98-L112

```solidity
function configureLockdrop(
    Lockdrop calldata _lockdropData
) external onlyAdmin {
    if (_lockdropData.end < block.timestamp)
        revert LockdropEndedError(
            _lockdropData.end,
            uint32(block.timestamp)
        ); // , "LockManager: End date is in the past");
    if (_lockdropData.start >= _lockdropData.end)
        revert LockdropInvalidError();

    lockdrop = _lockdropData;

    emit LockDropConfigured(_lockdropData);
}
```

#### Recommended Mitigation Steps:
Add a check to ensure that the `start` time is not in the past by comparing it to the current block timestamp. Here is the updated function with the additional check:

```diff
function configureLockdrop(
    Lockdrop calldata _lockdropData
) external onlyAdmin {
    if (_lockdropData.end < block.timestamp)
        revert LockdropEndedError(
            _lockdropData.end,
            uint32(block.timestamp)
        ); // , "LockManager: End date is in the past");
    if (_lockdropData.start >= _lockdropData.end)
        revert LockdropInvalidError();
+    if (_lockdropData.start < block.timestamp)
+        revert LockdropStartInPastError();

    lockdrop = _lockdropData;

    emit LockDropConfigured(_lockdropData);
}
```



## [QA-8] Incompatibility of `LockManager` with Blast's Rebasing ETH and Yield Modes

#### Impact
`LockManager` is to be deployed on Blastmainnet according to the README. However, deploying this contract on Blast, which features rebasing ETH and different yield modes (Void, Automatic, Claimable), could lead to unexpected behavior or inefficiencies. `LockManager` does not currently account for the rebasing nature of ETH or the yield modes, which could result in incorrect ETH handling, potential loss of funds, or operational errors.

## Proof of Concept
The `_lock` function in the `LockManager` contract handles ETH directly without considering Blast's rebasing ETH and yield modes:

```solidity
function _lock(
    address _tokenContract,
    uint256 _quantity,
    address _tokenOwner,
    address _lockRecipient
) private {
    if (_tokenContract == address(0)) {
        if (msg.value != _quantity) revert ETHValueIncorrectError();
    } else {
        if (msg.value != 0) revert InvalidMessageValueError();
        // Token transfer logic here
    }

    // Rest of the _lock function logic
}
```

>This logic assumes a static ETH balance, which is not the case on Blast due to rebasing and yield mechanisms.

#### Recommended Mitigation Steps:
1. **Modify the `_lock` function to account for rebasing ETH and yield modes**:
   - Query the current yield mode and adjust the logic accordingly.
   - Incorporate logic to interact with Blast's yield contract to manage ETH balances and yields appropriately.

2. **Ensure that operations dependent on transaction finality are aware of Blast's transaction states**:
   - Add checks or delays to ensure transactions are finalized before proceeding with certain actions.








## [QA-9] Gas issue in `getLockedWeightedValue` Function

#### Impact
The `getLockedWeightedValue` function in the `LockManager` contract accesses storage variables multiple times within a loop, leading to higher gas costs. This can make the function more expensive to execute, especially when there are many configured tokens.

#### Proof of Concept
The following code snippet shows the original implementation of the `getLockedWeightedValue` function, which repeatedly accesses storage variables:

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L461-L487

```solidity
function getLockedWeightedValue(
    address _player
) external view returns (uint256 _lockedWeightedValue) {
    uint256 lockedWeighted = 0;
    uint256 configuredTokensLength = configuredTokenContracts.length;
    for (uint256 i; i < configuredTokensLength; i++) {
        if (
            lockedTokens[_player][configuredTokenContracts[i]].quantity > 0 &&
            configuredTokens[configuredTokenContracts[i]].active
        ) {
            // We are assuming all tokens have a maximum of 18 decimals and that USD Price is denoted in 1e18
            uint256 deltaDecimal = 10 **
                (18 -
                    configuredTokens[configuredTokenContracts[i]].decimals);
            lockedWeighted +=
                (deltaDecimal *
                    lockedTokens[_player][configuredTokenContracts[i]]
                        .quantity *
                    configuredTokens[configuredTokenContracts[i]]
                        .usdPrice) /
                1e18;
        }
    }

    _lockedWeightedValue = lockedWeighted;
}
```

#### Recommended Mitigation Steps:
To improve efficiency, consider caching the storage variables in local variables before using them in the loop. Like this:
```solidity
function getLockedWeightedValue(
    address _player
) external view returns (uint256 _lockedWeightedValue) {
    uint256 lockedWeighted = 0;
    uint256 configuredTokensLength = configuredTokenContracts.length;
    for (uint256 i = 0; i < configuredTokensLength; i++) {
        address tokenContract = configuredTokenContracts[i];
        LockedToken memory lockedToken = lockedTokens[_player][tokenContract];
        ConfiguredToken memory configuredToken = configuredTokens[tokenContract];

        if (lockedToken.quantity > 0 && configuredToken.active) {
            // Ensure token decimals are within a valid range
            require(configuredToken.decimals <= 18, "Token decimals exceed 18");

            // We are assuming all tokens have a maximum of 18 decimals and that USD Price is denoted in 1e18
            uint256 deltaDecimal = 10 ** (18 - configuredToken.decimals);
            lockedWeighted += (deltaDecimal * lockedToken.quantity * configuredToken.usdPrice) / 1e18;
        }
    }

    _lockedWeightedValue = lockedWeighted;
}
```







## [QA-10]  Lack of Access Control on `setUSDThresholds` Function

The `setUSDThresholds` function allows the admin to set the approval and disapproval thresholds arbitrarily without any constraints. This can lead to invalid configurations where the disapprove threshold is higher than or equal to the approve threshold.

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L129-L139

```solidity
function setUSDThresholds(
    uint8 _approve,
    uint8 _disapprove
) external onlyAdmin {
    if (usdUpdateProposal.proposer != address(0))
        revert ProposalInProgressError();
    APPROVE_THRESHOLD = _approve;
    DISAPPROVE_THRESHOLD = _disapprove;

    emit USDThresholdUpdated(_approve, _disapprove);
}
```

#### Recommended Mitigation Steps:
Add constraints to the `setUSDThresholds` function to ensure that the disapprove threshold is not higher than or equal to the approve threshold. This can be done by adding a check before setting the thresholds:

Modified code snippet:
```solidity
function setUSDThresholds(
    uint8 _approve,
    uint8 _disapprove
) external onlyAdmin {
    if (usdUpdateProposal.proposer != address(0))
        revert ProposalInProgressError();
    if (_disapprove >= _approve)
        revert InvalidThresholdsError();
    APPROVE_THRESHOLD = _approve;
    DISAPPROVE_THRESHOLD = _disapprove;

    emit USDThresholdUpdated(_approve, _disapprove);
}

// Define the error
error InvalidThresholdsError();
```





## [QA-11] Locks can be extended to an old `unlockTime`

Allowing the new unlock time to be the same as the old unlock time can lead to several issues:
- **No Effective Extension**: The lock duration is effectively not extended, leading to confusion for users.
- **Bypassing Locking Mechanism**: Users might exploit this to bypass the intended locking mechanism.
- **User Experience**: Confusing and frustrating for users who believe they are extending their lock duration but find that the unlock time remains unchanged.

#### Proof of Concept
The issue is present in the `setLockDuration` function. The current check allows the new unlock time to be the same as the old unlock time:

```solidity
if (
    uint32(block.timestamp) + uint32(_duration) <
    lockedTokens[msg.sender][tokenContract].unlockTime
) {
    revert LockDurationReducedError();
}
```

#### Recommended Mitigation Steps
To ensure the new unlock time is strictly greater than the old unlock time, update the condition in the `setLockDuration` function as follows:

```diff
if (
-     uint32(block.timestamp) + uint32(_duration) <
-    lockedTokens[msg.sender][tokenContract].unlockTime
+    uint32(block.timestamp) + uint32(_duration) <=
+    lockedTokens[msg.sender][tokenContract].unlockTime
) {
    revert LockDurationReducedError();
}
```


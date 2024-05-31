| Severity | Title                                                                                                           |
| -------- | --------------------------------------------------------------------------------------------------------------- |
| L-1      | Underflow when token contracts with more than 18 decimals are used                                              |
| L-2      | Users can force the contract to call the `transfer` method of an arbitrary contract through the `unlock` method |
| L-3      | ETH deposited via smart contracts could become permanently stuck because `unlock` doesn't safely transfer eth   |
| R-1      | Unnecessary getter function for `configuredTokens` mapping can be removed                                       |
| R-2      | Unnecessary token allowance check in `LockManager::_lock` can be removed                                        |
| R-3      | Support emergency withdrawls at the manager or token contract level                                             |

# [L-1] Underflow when token contracts with more than 18 decimals are used

## Impact

As defined in the scoping Q & A, there are plans to support additional ERC20 tokens in the future. However, an assumption is made that token decimals do not exceed 18. This will cause an underflow when trying to calculate the `deltaDecimal` variable.

[LockManager.sol#L472-L475](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L472-L475) (lines combined for readability)

```
// We are assuming all tokens have a maximum of 18 decimals and that USD Price is denoted in 1e18
uint256 deltaDecimal = 10 ** (18 - configuredTokens[configuredTokenContracts[i]].decimals);
```

As long as the token contract is configured to remains active, any locked funds for this token contract will remain stuck as force harvest during an `unlock` transaction will ultimately call the `getLockedWeightedValue` function which would underflow as demonstrated above.

## Recommended Mitigation

It is recommended that the decimal <= 18 assumption is validated when adding token contracts, and that `configureToken` would throw an error when attempting to add a token contract with too many decimals.

```diff
    /// @inheritdoc ILockManager
    function configureToken(
        address _tokenContract,
        ConfiguredToken memory _tokenData
    ) external onlyAdmin {
        if (_tokenData.nftCost == 0) revert NFTCostInvalidError();
+       if (_tokenContract != address(0)) {
+           if (_tokenData.decimals > 18) revert TooManyDecimalsError();
+       }
        if (configuredTokens[_tokenContract].nftCost == 0) {
            // new token
            configuredTokenContracts.push(_tokenContract);
        }
        configuredTokens[_tokenContract] = _tokenData;

        emit TokenConfigured(_tokenContract, _tokenData);
    }
```

Define a new error in ILockManager

```solidity
    /// @notice Error thrown when a token contract with more than 18 decimals is added
    error TooManyDecimalsError();
```

# [L-2] Users can force the contract to call the `transfer` method of an arbitrary contract through the `unlock` method

## Impact

The `unlock` function lacks check on `tokenContract` to confirm it is a valid token in the protocol. This means it's possible to call `unlock(maliciousContractAddress, 0)` and the function will not revert. This means the contract will emit a `Unlocked(msg.sender, maliciousContractAddress, 0)` event which could pollute offchain indexers.

Additionally the contract will attempt to call the `transfer` method on this arbitrary contract which could be used maliciously

## Recommended Mitigation

It is recommended that the `onlyConfiguredToken` modifier is added to the `unlock` function to ensure only the tokens verified for use in the protocol are able to be withdrawn.

# [L-3] ETH deposited via smart contracts could become permanently stuck because `unlock` doesn't safely transfer eth

### Impact

When calling `unlock` to withdraw locked Ether from the contract it is processed in the following line:

```solidity
    payable(msg.sender).transfer(_quantity);
```

As pointed out in the 4naly3er report, using `transfer` causes issues where if the receiver is a smart contract with logic in it's fallback function the transfer attempt can run out of gas and fail. However if this line is changed to `payable(msg.sender).call{value: _quantity}("");` as suggested by the bot report, the call will still fail if the smart contract receiving the funds has no fallback or receive function. On top of this there is no option for the user with the locked Ether to change their recipient address meaning the funds would be stuck in the contract permanently.

### Recommended Mitigation

The most simple way to rectify this would be to add an extra `_recipient` parameter to the `unlock` function, then for if any reason the address which locked the tokens cannot receive Ether they can instead pass a different address which can receive the tokens safely.

```diff
    function unlock(
        address _tokenContract,
        uint256 _quantity,
+       address payable _recipient
    ) external notPaused nonReentrant {
        LockedToken storage lockedToken = lockedTokens[msg.sender][
            _tokenContract
        ];
        if (lockedToken.quantity < _quantity)
            revert InsufficientLockAmountError();
        if (lockedToken.unlockTime > uint32(block.timestamp))
            revert TokenStillLockedError();

        // force harvest to make sure that they get the schnibbles that they are entitled to
        accountManager.forceHarvest(msg.sender);

        lockedToken.quantity -= _quantity;

        // send token
        if (_tokenContract == address(0)) {
-           payable(msg.sender).transfer(_quantity);
+           _recipient.call{value: _quantity}("");
        } else {
            IERC20 token = IERC20(_tokenContract);
            token.transfer(msg.sender, _quantity);
        }

        emit Unlocked(msg.sender, _tokenContract, _quantity);
    }
```

# [R-1] Unnecessary getter function for `configuredTokens` mapping can be removed

The `LockManager` contract has a `getConfiguredToken(address _tokenContract)` getter function which returns `configuredTokens[_tokenContract]`. However the mapping `configuredTokens` is public meaning this getter function will already exist and users can query `configuredTokens(_tokenContract)` to receive the same result. Therefore it's possible to remove this function entirely to improve the readabiltiy of the codebase.

# [R-2] Unnecessary token allowance check in `LockManager::_lock`

The following check in `_lock` is unnecessary as the `transferFrom` function will revert if the caller does not have the sufficient allowance set anyway.

```solidity
    uint256 allowance = token.allowance(_tokenOwner, address(this));
    if (allowance < _quantity) revert InsufficientAllowanceError();
```

Therefore it is possible to remove these lines from the function logic to improve the codes readability and gas usage without introducing any risk to the project.

# [R-3] Support emergency withdrawls at the manager or token contract level

In certain cases, it may be beneficial for the protocol to allow emergency withdraws at a token contract level. A possible way to implement this is to add an `emergencyWithdrawsEnabled` flag in LockManager at the token contract configuration level where locking new funds are disabled and unlocks are processed without lock duration checks and harvesting on an `emergencyWithdraw` function. This would help protect users in the event issues occur within the protocol after launch.

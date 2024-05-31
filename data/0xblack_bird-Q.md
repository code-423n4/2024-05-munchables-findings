## [L-01] modifier `onlyConfiguredToken` not checked for `setLockDuration` potentially causing non-configured tokens in the system.

`onlyConfiguredToken` modifier checks if the particular token contract is configured or not.
```solidity
 modifier onlyActiveToken(address _tokenContract) {
        if (!configuredTokens[_tokenContract].active)
            revert TokenNotConfiguredError();
        _;
    }

```
which actually is done on the admin only accessible function `configureToken` where if token is new sets an nft price
& other values via token data.
The vulnerability lies in the `setLockDuration` function where such a check is not done
```solidity
function setLockDuration(uint256 _duration) external notPaused {
        if (_duration > configStorage.getUint(StorageKey.MaxLockDuration))
            revert MaximumLockDurationError();
        playerSettings[msg.sender].lockDuration = uint32(_duration);
        // update any existing lock
        uint256 configuredTokensLength = configuredTokenContracts.length;
        for (uint256 i; i < configuredTokensLength; i++) {
            address tokenContract = configuredTokenContracts[i];
            if (lockedTokens[msg.sender][tokenContract].quantity > 0) {
                // check they are not setting lock time before current unlocktime
                if (
                    uint32(block.timestamp) + uint32(_duration) <
                    lockedTokens[msg.sender][tokenContract].unlockTime
                ) {
                    revert LockDurationReducedError();
                }

                uint32 lastLockTime = lockedTokens[msg.sender][tokenContract]
                    .lastLockTime;
                lockedTokens[msg.sender][tokenContract].unlockTime =
                    lastLockTime +
                    uint32(_duration);
            }
        }

        emit LockDuration(msg.sender, _duration);
    }
```
consider this edge case scenario 
- Alice locks a particular token 'x' which a configured token at the locking time 
- Alice lock now approaches the unlockTime 
- Alice decides to increase lock duration calling `setLockDuration` but at the current time this 'x' is no longer in the configured  tokens list
- Alice can still have a lock that is not configured in the system.

# Recommendation
- implement a modifier check for the `setLockDuration` maybe have user to input the token contract also,making the modifier
  check possible.


## [L-02] `_lock` doesnt check if the quantity is atleast a minimumAmount.

`_lock` function doesnt check if the quantity which is to be locked is above a minimum amount ,if this check is not followed user can lock very small amount or even zero amounts, which isnt advisable , is a gas waste and cause unnecessary storage variables update.

```solidity
  function _lock(
        address _tokenContract,
        uint256 _quantity,
        address _tokenOwner,
        address _lockRecipient
    ) private {
        (
            address _mainAccount,
            MunchablesCommonLib.Player memory _player
        ) = accountManager.getPlayer(_lockRecipient);
        if (_mainAccount != _lockRecipient) revert SubAccountCannotLockError();
        if (_player.registrationDate == 0) revert AccountNotRegisteredError();
        // check approvals and value of tx matches
        if (_tokenContract == address(0)) {
            if (msg.value != _quantity) revert ETHValueIncorrectError();
        } else {
            if (msg.value != 0) revert InvalidMessageValueError();
            IERC20 token = IERC20(_tokenContract);
            uint256 allowance = token.allowance(_tokenOwner, address(this));
            if (allowance < _quantity) revert InsufficientAllowanceError();
        }

        LockedToken storage lockedToken = lockedTokens[_lockRecipient][
            _tokenContract
        ];
        ConfiguredToken storage configuredToken = configuredTokens[
            _tokenContract
        ];

        // they will receive schnibbles at the new rate since last harvest if not for force harvest
        accountManager.forceHarvest(_lockRecipient);

        // add remainder from any previous lock
        uint256 quantity = _quantity + lockedToken.remainder;
        uint256 remainder;
        uint256 numberNFTs;
        uint32 _lockDuration = playerSettings[_lockRecipient].lockDuration;
        if (_lockDuration == 0) {
            _lockDuration = lockdrop.minLockDuration;
        }
        if (
            lockdrop.start <= uint32(block.timestamp) &&
            lockdrop.end >= uint32(block.timestamp)
        ) {
            if (
                _lockDuration < lockdrop.minLockDuration ||
                _lockDuration >
                uint32(configStorage.getUint(StorageKey.MaxLockDuration))
            ) revert InvalidLockDurationError();
            if (msg.sender != address(migrationManager)) {
                // calculate number of nfts
                remainder = quantity % configuredToken.nftCost;
                numberNFTs = (quantity - remainder) / configuredToken.nftCost;

                if (numberNFTs > type(uint16).max) revert TooManyNFTsError();

                // Tell nftOverlord that the player has new unopened Munchables
                nftOverlord.addReveal(_lockRecipient, uint16(numberNFTs));
            }
        }

        // Transfer erc tokens
        if (_tokenContract != address(0)) {
            IERC20 token = IERC20(_tokenContract);
            token.transferFrom(_tokenOwner, address(this), _quantity);
        }

        lockedToken.remainder = remainder;
        lockedToken.quantity += _quantity;
        lockedToken.lastLockTime = uint32(block.timestamp);
        lockedToken.unlockTime =
            uint32(block.timestamp) +
            uint32(_lockDuration);

        // set their lock duration in playerSettings
        playerSettings[_lockRecipient].lockDuration = _lockDuration;

        emit Locked(
            _lockRecipient,
            _tokenOwner,
            _tokenContract,
            _quantity,
            remainder,
            numberNFTs,
            _lockDuration
        );
    }

```
## [L-03] `_lock` allows for _lockDuration to be less than minLockDuration when not in lockdrop period.

Take Look at this snippet from `_lock`

```solidity
if (_lockDuration == 0) {
            _lockDuration = lockdrop.minLockDuration;
        }
        if (
            lockdrop.start <= uint32(block.timestamp) &&
            lockdrop.end >= uint32(block.timestamp)
        )

```
Here we can see that that _lockDuration is checked to be zero and if zero updates to minimum lock duration,however 
in the edge case where a lock duration is higher than zero and less than minLockDuration i not addressed here,As
a result a user can assign a lock duration lesser than minimum.

## [L-04] Remainder amount in the `_lock` cannot be withdrawn and might remain in the system.

In the `_lock` function a remainder is calculated,which is the remaining amount that does not reach an nft cost.This
remainder amount is added to the quantity nexttime lock is called.in a condition where say ,If a user doesnt want to lock again ,this amount|
might remain in the system with no way of withdrawing.

```solidity
  uint256 quantity = _quantity + lockedToken.remainder;
  remainder = quantity % configuredToken.nftCost;
  numberNFTs = (quantity - remainder) / configuredToken.nftCost;
  
```

## [l-05] `unlock` allows for withdrawing of zero amounts.

`unlock` function main functionality is to check if a lock duration is over and return the tokens to the owner.However the function doesnt check if withdrawing quantity is zero or a minimum amount.Adding this check improves codes robustness and makes it future proof 

```solidity
 function unlock(
        address _tokenContract,
        uint256 _quantity
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
            payable(msg.sender).transfer(_quantity);
        } else {
            IERC20 token = IERC20(_tokenContract);
            token.transfer(msg.sender, _quantity);
        }

        emit Unlocked(msg.sender, _tokenContract, _quantity);
    }


```

## [l-06] modifier `onlyActiveToken` not checked for `setLockDuration` causing inactive tokens in the system. 

`onlyActiveToken` modifier checks if the particular token contract is configured or not.
```solidity
  modifier onlyActiveToken(address _tokenContract) {
        if (!configuredTokens[_tokenContract].active)
            revert TokenNotConfiguredError();
        _;
    }

```
Which sets the particular token contract active or not in the system

The vulnerability lies in the `setLockDuration` function where such a check is not done
```solidity
function setLockDuration(uint256 _duration) external notPaused {
        if (_duration > configStorage.getUint(StorageKey.MaxLockDuration))
            revert MaximumLockDurationError();
        playerSettings[msg.sender].lockDuration = uint32(_duration);
        // update any existing lock
        uint256 configuredTokensLength = configuredTokenContracts.length;
        for (uint256 i; i < configuredTokensLength; i++) {
            address tokenContract = configuredTokenContracts[i];
            if (lockedTokens[msg.sender][tokenContract].quantity > 0) {
                // check they are not setting lock time before current unlocktime
                if (
                    uint32(block.timestamp) + uint32(_duration) <
                    lockedTokens[msg.sender][tokenContract].unlockTime
                ) {
                    revert LockDurationReducedError();
                }

                uint32 lastLockTime = lockedTokens[msg.sender][tokenContract]
                    .lastLockTime;
                lockedTokens[msg.sender][tokenContract].unlockTime =
                    lastLockTime +
                    uint32(_duration);
            }
        }

        emit LockDuration(msg.sender, _duration);
    }
```
consider this edge case scenario 
- Bob locks a particular token 'x' which is a active token at the locking time.
- Bob lock now approaches the unlockTime 
- Bob decides to increase lock duration calling `setLockDuration` but at the current time this 'x' is no longer an active token.
- Bob can have a lock with locked token which is not active in the system.

# Recommendation
- implement a modifier check for the `setLockDuration` maybe have user to input the token contract also,making the modifier
  check possible.
## [01] Token decimals are not checked in `configureToken` but assumed that token has `<=18` decimals

In `getLockedWeightedValue` function we have calculation that assumed that token has 18 decimals.
Same says the comment in the code.

```solidity
    // We are assuming all tokens have a maximum of 18 decimals and that USD Price is denoted in 1e18
    uint256 deltaDecimal = 10 **
        (18 -
@>          configuredTokens[configuredTokenContracts[i]].decimals);
    lockedWeighted +=
        (deltaDecimal *
            lockedTokens[_player][configuredTokenContracts[i]]
                .quantity *
            configuredTokens[configuredTokenContracts[i]]
                .usdPrice) /
        1e18;
```

In that case we should check in `configureToken()` that configured token has <=18 decimals.

```diff
    function configureToken(
        address _tokenContract,
        ConfiguredToken memory _tokenData
    ) external onlyAdmin {
+       if (_tokenData.decimals > 18) revert DecimalsInvalidError();
        if (_tokenData.nftCost == 0) revert NFTCostInvalidError();
        if (configuredTokens[_tokenContract].nftCost == 0) {
            // new token
            configuredTokenContracts.push(_tokenContract);
        }
        configuredTokens[_tokenContract] = _tokenData;

        emit TokenConfigured(_tokenContract, _tokenData);
    }
```

### Recommendation

Add following check in `configureToken` function as mentioned above.

## [02] `setLockDuration()` may revert and not update all player's tokens

According to comments/documentation function `setLockDuration()` should set duration for all player's tokens.
`@notice Sets the lock duration for a player's tokens`
But if for any token this condition will be `true`

```solidity
uint32(block.timestamp) + uint32(_duration) < lockedTokens[msg.sender][tokenContract].unlockTime
```

then function will `revert` and all tokens after this token will not be updated.

```solidity
File: src/managers/LockManager.sol

                if (
                    uint32(block.timestamp) + uint32(_duration) <
                    lockedTokens[msg.sender][tokenContract].unlockTime
                ) {
                    revert LockDurationReducedError();
                }
```

### Recommendations

Consider use `skip` model instead of `revert`.

## [03] `lock()`/`unlock()` will revert if amount larger than `uint96` some `ERC20` tokens

`ERC20` tokens like `UNI`, `COMP` will revert if `transfer`red value is larger than `uint96`.
This tokens cant be `locked/unlocked` more than `uint96` in one tx.

- [COMP (Compound Protocol)](https://github.com/compound-finance/compound-protocol/blob/a3214f67b73310d547e00fc578e8355911c9d376/contracts/Governance/Comp.sol#L115-L142)
- [UNI (Uniswap)](https://github.com/Uniswap/governance/blob/eabd8c71ad01f61fb54ed6945162021ee419998e/contracts/Uni.sol#L209-L236)

According to `ERC20 token behaviors in scope` this non standard ERC20 in scope.

```solidity
File: src/managers/LockManager.sol

376: token.transferFrom(_tokenOwner, address(this), _quantity)
423: token.transfer(msg.sender, _quantity)
```

## [04] Use `call()` instead of `transfer()` since `transfer()` can revert

the `transfer()/send()` call only provides 2300 gas for the contract to complete its operations.
This means the following cases can cause the transfer to fail:

- Contract does not have a payable callback
- Contract's payable callback spends more than 2300 gas (which is only enough to emit something)
- The destination is a smart contract but that smart contract is called via an intermediate proxy contract increasing the case requirements to more than 2300 gas units. A further example of unknown destination complexity is that of a multisig wallet that as part of its operation uses more than 2300 gas units.
- Future changes or forks in Ethereum result in higher gas fees than transfer provides. The .`transfer()/send()` creates a hard dependency on 2300 gas units being appropriate now and into the future.

```solidity
File: src/managers/LockManager.sol

420: payable(msg.sender).transfer(_quantity)
```

### Recommendations

Use low level `call()` to transfer ETH.

## [05] Unsafe use of `transfer()/transferFrom()` with arbitrary `token`

Some tokens returns bool, USDT token does not return anything from `transfer()` and `transferFrom()` functions.
`LockManager` assumes that if tx not revert then token was transferred successfully, but it may not be true.

```solidity
File: src/managers/LockManager.sol

376: token.transferFrom(_tokenOwner, address(this), _quantity);
423: token.transfer(msg.sender, _quantity);
```

### Recommendations

Use OpenZeppelin's `SafeERC20` library to handle `ERC20` token transfers.

## [06] `APPROVE_THRESHOLD` and `DISAPPROVE_THRESHOLD` are not limited to a reasonable range

`APPROVE_THRESHOLD` and `DISAPPROVE_THRESHOLD` are not limited to a reasonable range, they can be set to higher or lower values than expected by human error.

```solidity
File: src/managers/LockManager.sol

135: APPROVE_THRESHOLD = _approve;
136: DISAPPROVE_THRESHOLD = _disapprove;
```

### Recommendations

Add simple range checks for `APPROVE_THRESHOLD` and `DISAPPROVE_THRESHOLD`.

## [07] Code does not follow the best practice of check-effects-interaction

Code should follow the best-practice of [CEI](https://blockchain-academy.hs-mittweida.de/courses/solidity-coding-beginners-to-intermediate/lessons/solidity-11-coding-patterns/topic/checks-effects-interactions/), where state variables are updated before any external calls are made. Doing so prevents a large class of reentrancy bugs.

```solidity
File: src/managers/LockManager.sol

/// `token.transferFrom(_tokenOwner, address(this), _quantity)`
387: playerSettings[_lockRecipient].lockDuration = _lockDuration
```

### Recommendations

Change state variables before making external calls.

## [08] Unchecked Return Values of `transfer()/transferFrom()`

`LockManager` use `transfer()` and `transferFrom()` without checking their return values.
Some IERC20 implementations use these boolean return values to signal transaction failures instead of using `revert()`.
It is recommended to always check the return values of these methods.

```solidity
File: src/managers/LockManager.sol

376: token.transferFrom(_tokenOwner, address(this), _quantity);
423: token.transfer(msg.sender, _quantity);
```

### Recommendations

Use OpenZeppelin's `SafeERC20` library to handle `ERC20` token transfers.

## [09] Events may be emitted out of order due to reentrancy

It's essential to ensure that events follow the best practice of check-effects-interaction and are emitted before any external calls to prevent out-of-order events due to reentrancy.
Emitting events post external interactions may cause them to be out of order due to reentrancy, which can be misleading or erroneous for event listeners.
[Refer to the Solidity Documentation for best practices.](https://solidity.readthedocs.io/en/latest/security-considerations.html#reentrancy)

```solidity
File: src/managers/LockManager.sol

/// `transferFrom()` before `Locked` event emit
388: emit Locked(
            _lockRecipient,
            _tokenOwner,
            _tokenContract,
            _quantity,
            remainder,
            numberNFTs,
            _lockDuration
        );
/// `transfer()` before `Unlocked` event emit
425: emit Unlocked(msg.sender, _tokenContract, _quantity);
```

### Recommendations

Change the order of events to emit before external calls.

## [10] `setLockDuration()` iterates over `configuredTokenContracts` if length is large, it may lead to high gas costs

`setLockDuration()` iterates over `configuredTokenContracts` which may be expensive if the array is large.
Consider using index access instead of iterating over the array if possible in contract logic.

```solidity
File: src/managers/LockManager.sol

252: for (uint256 i; i < configuredTokensLength; i++) {
```

### Recommendations

Use index access instead of iterating over the array if it's possible in contract logic.

## [11] `setUSDThresholds()` should be a two step procedure

Critical `setUSDThresholds()` function should be a use two-step procedure to avoid human error.

```solidity
File: src/managers/LockManager.sol

128: function setUSDThresholds(uint8 _approve, uint8 _disapprove) external onlyAdmin {
```

### Recommendations

Add a two-step process for `setUSDThresholds()` function.

## [12] Loss of precision on division

Solidity doesn't support fractions, so divisions by large numbers could result in the quotient being zero.

```solidity
File: src/managers/LockManager.sol

364: numberNFTs = (quantity - remainder) / configuredToken.nftCost;
481: .usdPrice) / 1e18;
```

## [13] Unsafe downcast from larger to smaller integer types

When a type is downcast to a smaller type, the higher order bits are discarded, resulting in the application of a modulo operation to the original value.

```solidity
File: src/managers/LockManager.sol

/// casted from `uint256` to `uint32`
249: playerSettings[msg.sender].lockDuration = uint32(_duration);
/// casted from `uint256` to `uint32`
257: uint32(block.timestamp) + uint32(_duration) <
/// casted from `uint256` to `uint32`
267: uint32(_duration);
/// casted from `uint256` to `uint16`
369: nftOverlord.addReveal(_lockRecipient, uint16(numberNFTs));
```

### Recommendations

Use OpenZeppelin's SafeCast library to safely cast uints.

## [14] Missing checks for `address(0)` in constructor

Check for zero-address to avoid the risk of setting `address(0)` for state variables when deploying.

```solidity
File: src/managers/LockManager.sol

/// `_configStorage` has lack of `address(0)` check before use
59: constructor(address _configStorage) {
```

[59](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L59)

## [15] Casting `block.timestamp` to `uint32` Limit Contract Lifespan

Casting `block.timestamp` to a smaller integer type like uint32 can potentially reduce the lifespan of a contract.
While the current Unix timestamp fits within the uint32 range as of now, it will eventually exceed this range by the year 2106.

```solidity
File: src/managers/LockManager.sol

104: uint32(block.timestamp)
            ); // , "LockManager: End date is in the past");
166: usdUpdateProposal.proposedDate = uint32(block.timestamp);
257: uint32(block.timestamp) + uint32(_duration) <
                    lockedTokens[msg.sender][tokenContract].unlockTime
                ) {
                    revert LockDurationReducedError();
353: lockdrop.start <= uint32(block.timestamp) &&
            lockdrop.end >= uint32(block.timestamp)
        ) {
            if (
                _lockDuration < lockdrop.minLockDuration ||
                _lockDuration >
                uint32(configStorage.getUint(StorageKey.MaxLockDuration))
            ) revert InvalidLockDurationError();
381: lockedToken.lastLockTime = uint32(block.timestamp);
383: uint32(block.timestamp) +
            uint32(_lockDuration);
410: if (lockedToken.unlockTime > uint32(block.timestamp))
            revert TokenStillLockedError();
```

### Recommendations

Use `uint48` / `uint64` for time related variables to future-proof the contract.

## [16] Centralization issue caused by admin privileges

Since in readme trusted roles not specified `All trusted roles in the protocol N/A`.
There are priviliged roles that are a single point of failure, who can use critical functions, posing a centralization issue.

```solidity
File: src/managers/LockManager.sol

85: function configUpdated() external override onlyConfigStorage
98: function configureLockdrop(
        Lockdrop calldata _lockdropData
    ) external onlyAdmin
115: function configureToken(
        address _tokenContract,
        ConfiguredToken memory _tokenData
    ) external onlyAdmin
129: function setUSDThresholds(
        uint8 _approve,
        uint8 _disapprove
    ) external onlyAdmin
142: function proposeUSDPrice(
        uint256 _price,
        address[] calldata _contracts
    )
        external
        onlyOneOfRoles(
            [
                Role.PriceFeed_1,
                Role.PriceFeed_2,
                Role.PriceFeed_3,
                Role.PriceFeed_4,
                Role.PriceFeed_5
            ]
        )
177: function approveUSDPrice(
        uint256 _price
    )
        external
        onlyOneOfRoles(
            [
                Role.PriceFeed_1,
                Role.PriceFeed_2,
                Role.PriceFeed_3,
                Role.PriceFeed_4,
                Role.PriceFeed_5
            ]
        )
210: function disapproveUSDPrice(
        uint256 _price
    )
        external
        onlyOneOfRoles(
            [
                Role.PriceFeed_1,
                Role.PriceFeed_2,
                Role.PriceFeed_3,
                Role.PriceFeed_4,
                Role.PriceFeed_5
            ]
        )
275: function lockOnBehalf(
        address _tokenContract,
        uint256 _quantity,
        address _onBehalfOf
    )
        external
        payable
        notPaused
        onlyActiveToken(_tokenContract)
        onlyConfiguredToken(_tokenContract)
        nonReentrant
297: function lock(
        address _tokenContract,
        uint256 _quantity
    )
        external
        payable
        notPaused
        onlyActiveToken(_tokenContract)
        onlyConfiguredToken(_tokenContract)
        nonReentrant
```

## [17] Lack of index element validation in external/public function

There's no validation to check whether the index element provided as an argument actually exists in the call.
The function should validate that the provided index element exists in the call before proceeding.

```solidity
File: src/managers/LockManager.sol

120: configuredTokens[_tokenContract]
124: configuredTokens[_tokenContract]
405: lockedTokens[msg.sender][
            _tokenContract
        ]
440: lockedTokens[_player]
443: lockedTokens[_player]
446: lockedTokens[_player]
449: lockedTokens[_player]
468: lockedTokens[_player]
478: lockedTokens[_player]
493: configuredTokens[_tokenContract]
499: playerSettings[_player]
```

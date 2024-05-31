### [Low-01] `_execUSDPriceUpdate()` only able to set one `configuredToken`s `USD` price in single type cause `USDUpdateProposal.contracts` is an `Array` where as `USDUpdateProposal.proposedPrice` is a `uint`

`_execUSDPriceUpdate()` loops over `usdUpdateProposal.contracts.length` and set `configuredTokens[tokenContract].usdPrice` to `usdUpdateProposal.proposedPrice`

https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L512-L516

But problem is `usdUpdateProposal.contracts` is an `ARRAY` and `usdUpdateProposal.proposedPrice` is a `uint256`
https://github.com/code-423n4/2024-05-munchables/blob/main/src/interfaces/ILockManager.sol#L63-L64

So its setting all contracts present in `usdUpdateProposal.contracts` to a single `USD` value, which is not practically possible. As different token have different prices.

Although this function intended to set multiple contracts USD price at a time, but a/c current implementation it only able to set `USD price of a TokenContract` each single time.

### mitigation
As `usdUpdateProposal.contracts` is an `ARRAY`
`usdUpdateProposal.proposedPrice` should be a `ARRAY` corresponding to each conracts


### [Low-02] In case of instant price fluctuation, A player can front-run priceFeeders and get benefited.

USD price for an `configureToken` is set by following steps
- One out of Five `PriceFeeder` purpose a USD price for Token
- Then this proposal under goes `approval` & `disapproval`
- Once any of 2 thresold hit first, 
    - if approval thresold hit then `_execute()` preformed and price set to tokenContract
    - if disapproval thresold hit then `puposed price` data stucture deleted.

In sudden market fluctuation this become irrevelent, and a User can get benefited from it by front running `approveUSDPrice()` in suitable situation

https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L129-L242

### Mitigation
Protocol should use `Oracles` like other defi project used.  

### [Low-03] Role.PriceFeed can `approve` & `disapprove` same time as there is no check for `disapproval` in `approveUSDPrice()`

A `PriceFeeder` at same time can `disapprove` and `approve` same purposed price cause `absence` of `disapproval` check in `approveUSDPrice()`

`approveUSDPrice()` has following checks
```solidity
    function approveUSDPrice(
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
    {
        if (usdUpdateProposal.proposer == address(0)) revert NoProposalError();
        if (usdUpdateProposal.proposer == msg.sender)
            revert ProposerCannotApproveError();
        if (usdUpdateProposal.approvals[msg.sender] == _usdProposalId) 
            revert ProposalAlreadyApprovedError();
        if (usdUpdateProposal.proposedPrice != _price)
            revert ProposalPriceNotMatchedError();
```
https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L191-L197
It never checks msg.sender already disapproved current purposal or not,
So as a result `priceFeeder` can first call `disapproveUSDPrice()` and then call `approveUSDPrice()` which is not a intended behaviour.

### Mitigation
Implement similar check present in `disapproveUSDPrice()`
```diff
    function approveUSDPrice(
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
    {
        if (usdUpdateProposal.proposer == address(0)) revert NoProposalError();
        if (usdUpdateProposal.proposer == msg.sender)
            revert ProposerCannotApproveError();
        if (usdUpdateProposal.approvals[msg.sender] == _usdProposalId) 
            revert ProposalAlreadyApprovedError();
        if (usdUpdateProposal.proposedPrice != _price)
            revert ProposalPriceNotMatchedError();
+       if (usdUpdateProposal.disapprovals[msg.sender] == _usdProposalId)
+           revert ProposalAlreadyDisapprovedError();
```

### [Low-04] `getLocked()` should checked that a Player have locked amount for a corresponding `configuredToken`, otherwise returns data will full of empty data structures

`getLocked()` basically loop through all `configuredToken` present in `configuredTokenContracts[]` and fetch `Player's LockedToken data`
https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L430-L458

But point it is not necessary that `A Player` locked all type of `configuredToken`

For Ex:- there are 10 type of `configuredToken`, Player only locked 1 type, so `getLocked()`'s return data feild will have 9 empty and 1 fill

This will problamatic when this function used by other contract. in that case that function will have to filter this return data to fecth Player's data.

### Mitigation
Before setting Player state to local variable it should check `Player's lockedToken` quantity should > 0.

```diff
    function getLocked(
        address _player
    ) external view returns (LockedTokenWithMetadata[] memory _lockedTokens) {
        uint256 configuredTokensLength = configuredTokenContracts.length; // @audit if array increased significantly then could be DOSed
        LockedTokenWithMetadata[]
            memory tmpLockedTokens = new LockedTokenWithMetadata[](
                configuredTokensLength
            );
        for (uint256 i; i < configuredTokensLength; i++) {
+         if(lockedTokens[_player][configuredTokenContracts[i]].quantity > 0){
            LockedToken memory tmpLockedToken;
            tmpLockedToken.unlockTime = lockedTokens[_player][
                configuredTokenContracts[i]
            ].unlockTime; 
            tmpLockedToken.quantity = lockedTokens[_player][
                configuredTokenContracts[i]
            ].quantity;
            tmpLockedToken.lastLockTime = lockedTokens[_player][
                configuredTokenContracts[i]
            ].lastLockTime;
            tmpLockedToken.remainder = lockedTokens[_player][
                configuredTokenContracts[i]
            ].remainder;
            tmpLockedTokens[i] = LockedTokenWithMetadata(
                tmpLockedToken,
                configuredTokenContracts[i]
            );
+         }
        }
```

### [Low-05] If no. of configuredTokens increase significantly then some functions like `getLocked()` will result DoS, due to exceeding block gas limit.

`getLocked()` & `getLockedWeightedValue()` loop through all `configuredToken` present inside `configuredTokenContracts[]` and fetch player data
https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L430-L487

if `configuredTokenContracts[]` increase indefinityly then above both function lead to a DoS situation.

### Mitigation
There should be a length limit check in loop

### [Low-06] There should be a minimum `lockDuration` check while confiuring / reconfiguring `lockDrop`

While setting `lockdrop'sparameters` there should be a upper limit and lower limit check for `Lockdrop.minLockDuration`
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
https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L98-L112

### [Low-07] Centralization
A Admin can 
- `Adds or updates a token configuration for locking purposes` 
- `Sets the thresholds for approving or disapproving USD price updates`
- `Configures the start and end times for a lockdrop event`

These lead protocol to Centralization side

https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L115

https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L98

https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L129
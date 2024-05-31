
## Configuring too many tokens can make functions to run OOG
### Description
`configureToken` function allows unlimited token configurations, potentially causing out-of-gas errors in functions like `getLocked` and `getLockedWeightedValue`.

### Example
```solidity
function configureToken(
    address _tokenContract,
    ConfiguredToken memory _tokenData
) external onlyAdmin {
    if (_tokenData.nftCost == 0) revert NFTCostInvalidError();
    if (configuredTokens[_tokenContract].nftCost == 0) {
        configuredTokenContracts.push(_tokenContract);
    }
    configuredTokens[_tokenContract] = _tokenData;
    emit TokenConfigured(_tokenContract, _tokenData);
}
```

### Context
https://github.com/code-423n4/2024-05-munchables/blob/d3db85c1c584fd226315ebfb6c4a085e44a8806a/src/managers/LockManager.sol#L122

### Recommendation
Consider setting a maximum limit for the number of tokens that can be configured to prevent out-of-gas issues. Also can be helpful a remove function.

## Approve and disapprove thresholds can be set to zero
### Description
`setUSDThresholds` function allows thresholds to be set to zero, making any price update automatically be able to be approved or disapproved.

### Example
```solidity
function setUSDThresholds(
    uint8 _approve,
    uint8 _disapprove
) external onlyAdmin {
    if (usdUpdateProposal.proposer != address(0))
        revert ProposalInProgressError();
    APPROVE_THRESHOLD = _approve;
    DISAPPROVE_THRESHOLD = _disapprove;
}
```

### Context
https://github.com/code-423n4/2024-05-munchables/blob/d3db85c1c584fd226315ebfb6c4a085e44a8806a/src/managers/LockManager.sol#L129

### Recommendation
Ensure thresholds cannot be set to zero or one to avoid automatic approvals or disapprovals. A minimum value for both should be helpful, indeed, a minimum of the initial value of `3` makes sense.

## Any `onlyOneOfRoles` can DoS the others by frontrunning

### Description
Members with roles from `Role.PriceFeed_1` to `Role.PriceFeed_5` can propose prices in order to block legitimate price updates indefinitely by creating continuous dummy proposals, leading to a typical DoS scenario.

Even not being malicious or compromissed, an operation can be set randomly in a order than makes it fail.

### Context
https://github.com/code-423n4/2024-05-munchables/blob/d3db85c1c584fd226315ebfb6c4a085e44a8806a/src/managers/LockManager.sol#L142

### Example
```solidity
if (usdUpdateProposal.proposer != address(0)) //@audit-issue compromissed one of roles can DoS other proposals with dummy ones
    revert ProposalInProgressError();
if (_contracts.length == 0) revert ProposalInvalidContractsError();
```
In the code snippet above, if `usdUpdateProposal.proposer` is not empty, a `ProposalInProgressError` exception is thrown

### Recommendation
Consider documenting this or doing something regarding ids or a an array of orders


## `lockOnBehalf` can be use to reduce other users rewards

### Description
This vulnerability relative to `lockOnBehalf` function happens where one user can lock rewards on behalf of another user. This could be potentially malicious if used improperly as it can affect the user's fees when rounding issues come into play. The caller of the function, by specifying the `_onBehalfOf` argument, could force harvest. 

### Example
Here is the affected code snippet:
```solidity
if (_onBehalfOf != address(0)) {
    lockRecipient = _onBehalfOf;
}
_lock(_tokenContract, _quantity, tokenOwner, lockRecipient);
```
In this part of code, if `_onBehalfOf` is not equal to zero address, the `lockRecipient` will be `_onBehalfOf`, allowing the caller to lock tokens on behalf of someone else.

### Context
https://github.com/code-423n4/2024-05-munchables/blob/d3db85c1c584fd226315ebfb6c4a085e44a8806a/src/managers/LockManager.sol#L293

### Recommendation
The ideal way to resolve this issue would be to validate the user permission before locking the tokens. This way, users will not be able to lock tokens on behalf of others without given consent. For instance, the lock recipient may have to sign a message with their private key, proving they have authorized the transaction. 

Alternatively, the contract might include a mechanism where users can “approve” others to lock on their behalf, similar to the ERC20 approve/allowance model.

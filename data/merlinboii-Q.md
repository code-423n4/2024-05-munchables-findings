# QA Report

## **[L-01] Donation to extend the unlocking time of other players**

The [lockOnBehalf()](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L275-L294) allows any user to donate 1 wei to extend the unlocking time of any player's locks.

Location: [LockManager::lockOnBehalf()](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L275-L294)
```solidity
        function lockOnBehalf(
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
    {
        address tokenOwner = msg.sender;
        address lockRecipient = msg.sender;
        if (_onBehalfOf != address(0)) {
            lockRecipient = _onBehalfOf;
        }

        _lock(_tokenContract, _quantity, tokenOwner, lockRecipient);
    }
```
Location: [LockManager::_lock()](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L311-L398)

```solidity
    function _lock(
        address _tokenContract,
        uint256 _quantity,
        address _tokenOwner,
        address _lockRecipient
    ) private {

        // SNIPPED 
        
        lockedToken.lastLockTime = uint32(block.timestamp);
        lockedToken.unlockTime =
            uint32(block.timestamp) +
            uint32(_lockDuration);
    }
```
---
## **[L-02] Inconsistent conditions for approving and disapproving the proposal price**

The approval and disapproval proposer price mechanism does not allow the `PriceFeed_X` that was approved first to become disapproved. 

However, this logic does not implement the vice versa scenario, as the mechanism allows `PriceFeed_X` that was disapproved first to become approved afterward. 

This results in an inconsistent mechanism and incorrectly tracks `usdUpdateProposal.approvalsCount` and `usdUpdateProposal.disapprovalsCount`.

Location: [LockManager::approveUSDPrice()](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L177-L207)
```solidity
File: ..
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

        usdUpdateProposal.approvals[msg.sender] = _usdProposalId;
        usdUpdateProposal.approvalsCount++;

        if (usdUpdateProposal.approvalsCount >= APPROVE_THRESHOLD) {
            _execUSDPriceUpdate();
        }

        emit ApprovedUSDPrice(msg.sender);
    }
```
---
## **[L-03] Players can set their `lockDuration` less than the minimum lock duration**

The [setLockDuration()](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L245-L272) allows player to set their `lockDuration` less than the `lockdrop.minLockDuration`

Location: [LockManager::setLockDuration()](https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L245-L272)

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
---
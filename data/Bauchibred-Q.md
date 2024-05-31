# QA Report

## Table of Contents

| Issue ID | Description |
| -------- | ----------- |
| [QA-01](#qa-01-protocol-claims-to-support->18-decimal-tokens-but-the-code-wouldnt-work-with-them) | Protocol claims to support` >18 decimal` tokens but the code wouldn't work with them |
| [QA-02](#qa-02-users-are-unfairly-blocked-from-withdrawals-when-protocol-is-paused) | User's are unfairly blocked from withdrawals when protocol is paused |
| [QA-03](#qa-03-current-voting-logic-is-flawed-as-it-allows-for-double-votes-for-the-same-proposal) | Current voting logic is flawed as it allows for double votes for the same proposal |
| [QA-04](#qa-04-protocols-logic-for-nft-costs-may-cause-over/undervaluation-of-locked-positions-over-time) | Protocol's Logic for NFT Costs May Cause Over/Undervaluation of Locked Positions Over Time |
| [QA-05](#qa-05-make-lockmanager#_execusdpriceupdate()-more-effective) | Make `LockManager#_execUSDPriceUpdate()` more effective |
| [QA-06](#qa-06-protocol-limits-itself-with-the-amount-of-voters-it-could-have) | Protocol limits itself with the amount of voters it could have |
| [QA-07](#qa-07-wrong-duration-could-be-set-due-to-unsafe-operations) | Wrong duration could be set due to unsafe operations |
| [QA-08](#qa-08-setters-dont-have-equality-checkers) | Setters don't have equality checkers |
| [QA-09](#qa-09-make-lockmanager#lockonbehalf()-more-effective) | Make `LockManager#lockOnBehalf()` more effective |

## QA-01 Protocol claims to support` >18 decimal` tokens but the code wouldn't work with them

### Proof of Concept

First take a look at https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/README.md#L161

| Question                                                                                      | Answer |
| --------------------------------------------------------------------------------------------- | ------ |
| [High decimals ( > 18)](https://github.com/d-xo/weird-erc20?tab=readme-ov-file#high-decimals) | Yes    |

Evidently, protocol plans to support >18 decimals token, issue however is that the `LockManager.sol` contract does not support these tokens and would revert in instances where these tokens are to be integrated. For example, take a look at https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L461-L487

```solidity
    function getLockedWeightedValue(
        address _player
    ) external view returns (uint256 _lockedWeightedValue) {
        uint256 lockedWeighted = 0;
        uint256 configuredTokensLength = configuredTokenContracts.length;
        for (uint256 i; i < configuredTokensLength; i++) {
            if (
                lockedTokens[_player][configuredTokenContracts[i]].quantity >
                0 &&
                configuredTokens[configuredTokenContracts[i]].active
            ) {
                // @audit
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

Evidently, the attempt at `18 - configuredTokens[configuredTokenContracts[i]].decimals` would always revert due to an underflow in the case where the decimals of the underlying token is more than 18.

### Impact

Protocol is incompatible with tokens it should be compatible with.

### Recommended Mitigation Steps

Make code more in alignment with docs, either integrate a mechanism to ensure these > 18 decimal tokens can be integrated or clearly document that they are not supported.

## QA-02 User's are unfairly blocked from withdrawals when protocol is paused

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L401-L427

```solidity
    function unlock(
        address _tokenContract,
        uint256 _quantity
    ) external notPaused nonReentrant {//|< notPaused?
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

Evidently the `notPaused` modifier has been applied to the attempt of unlocking which then makes users to be blocked off from withdrawing their tokens even if their lock duration has passed.

### Impact

Users are unfairly blocked off from accessing their tokens when protocol is paused, since the attempt to unlock wouldn't even reach the attempt to [transfer](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L422-L423) the tokens and revert.

> NB: This is borderline low/medium, submitting as QA due to Code4rena's new rule around admin actions, however it's popular web3 logic not to block user's off their withdrawals which could make this suffice as medium, leaving at the discretion of the judge.

### Recommended Mitigation Steps

Consider allowing users to access their tokens in as much as their unlock duration has passed, i.e apply these changes:
https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L401-L427

```diff
    function unlock(
        address _tokenContract,
        uint256 _quantity
-    ) external notPaused nonReentrant {
+    ) external  nonReentrant {
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


## QA-03 Current voting logic is flawed as it allows for double votes for the same proposal

### Proof of Concept

Take a look at [LockManager.\_execUSDPriceUpdate()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L506-L527)

```solidity
    function _execUSDPriceUpdate() internal {
        if (
            usdUpdateProposal.approvalsCount >= APPROVE_THRESHOLD &&
            usdUpdateProposal.disapprovalsCount < DISAPPROVE_THRESHOLD
        ) {
            uint256 updateTokensLength = usdUpdateProposal.contracts.length;
            for (uint256 i; i < updateTokensLength; i++) {
                address tokenContract = usdUpdateProposal.contracts[i];
                if (configuredTokens[tokenContract].nftCost != 0) {
                    configuredTokens[tokenContract].usdPrice = usdUpdateProposal
                        .proposedPrice;

                    emit USDPriceUpdated(
                        tokenContract,
                        usdUpdateProposal.proposedPrice
                    );
                }
            }

            delete usdUpdateProposal;
        }
    }
```

This function is called when updating the price of a token if the proposal meets the `APPROVE_THRESHOLD` and has fewer than `DISAPPROVE_THRESHOLD` disapprovals. However, the logic for counting votes has a flaw that allows a voter to vote both for and against the same proposal. This occurs because while there is a check to prevent disapproving a proposal that has already been approved, there is no similar check to prevent approving a proposal that has already been disapproved.

### Approve Function

In `approveUSDPrice()`, the checks are as follows:
[approveUSDPrice()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L191-L198)

```solidity
        if (usdUpdateProposal.proposer == address(0)) revert NoProposalError();
        if (usdUpdateProposal.proposer == msg.sender)
            revert ProposerCannotApproveError();
        if (usdUpdateProposal.approvals[msg.sender] == _usdProposalId)
            revert ProposalAlreadyApprovedError();
        if (usdUpdateProposal.proposedPrice != _price)
            revert ProposalPriceNotMatchedError();
```

### Disapprove Function

In `disapproveUSDPrice()`, the checks are as follows:
[disapproveUSDPrice()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L224-L231)

```solidity
        if (usdUpdateProposal.proposer == address(0)) revert NoProposalError();
        if (usdUpdateProposal.approvals[msg.sender] == _usdProposalId)
            revert ProposalAlreadyApprovedError();
        if (usdUpdateProposal.disapprovals[msg.sender] == _usdProposalId)
            revert ProposalAlreadyDisapprovedError();
        if (usdUpdateProposal.proposedPrice != _price)
            revert ProposalPriceNotMatchedError();
```

A price feed updater who has previously disapproved an update can later approve the same update. This discrepancy occurs because only the disapproval state is being set in `disapproveUSDPrice()`, and the check in `approveUSDPrice()` would still pass.

### Impact

Allowing a voter to vote both for and against the same proposal results in an inflated vote count, causing erroneous approvals or disapprovals. This can lead to incorrect execution of price updates since the thresholds (`>= DISAPPROVE_THRESHOLD` & `>= APPROVE_THRESHOLD`) are based on inaccurate vote counts. Consequently, proposals may be approved or disapproved inappropriately.

### Recommended Mitigation Steps

Ensure that each price feeder can vote only once per proposal. A voter changing their decision should have their previous vote nullified rather than counting as an additional vote. Implement checks to prevent double voting and ensure that each voter's final decision is accurately reflected in the vote count.

## QA-04 Protocol's Logic for NFT Costs May Cause Over/Undervaluation of Locked Positions Over Time

### Proof of Concept

Consider the following code snippet: [ILockManager.sol](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/interfaces/ILockManager.sol#L22-L27)

```solidity
    struct ConfiguredToken {
        uint256 usdPrice;
        uint256 nftCost;
        uint8 decimals;
        bool active;
    }
```

Each configured token uses this struct where `nftCost` represents the cost of the NFT associated with locking this token and `usdPrice` represents the USD price per token. However, the implementation [here](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L506-L527) shows that only the price of the underlying token can be updated. There is no functionality to update the `nftCost` associated with locking the token. This omission is problematic because the value of an NFT is not static and can depreciate or appreciate over time.

### Impact

Over time, the disparity between the initial and current value of the NFT will cause the `nftCost` to become outdated, leading to the overvaluation or undervaluation of the NFT. This discrepancy affects the calculation of the number of NFTs as shown in the code snippet below, which relies on potentially stale data:
[LockManager.sol](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L361-L370)

```solidity
            if (msg.sender != address(migrationManager)) {
                // calculate number of nfts
                remainder = quantity % configuredToken.nftCost;
                numberNFTs = (quantity - remainder) / configuredToken.nftCost;

                if (numberNFTs > type(uint16).max) revert TooManyNFTsError();

                // Tell nftOverlord that the player has new unopened Munchables
                nftOverlord.addReveal(_lockRecipient, uint16(numberNFTs));
            }
```

This leads to incorrect data being sent to the [`NFTOverlord#addReveal()`](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/overlords/NFTOverlord.sol#L65-L71) function, inflating or deflating the amount of unrevealed Munchable NFTs the user has.

### Recommended Mitigation Steps

Introduce functionality to update the `nftCost` to reflect the current value of the NFTs accurately.

## QA-05 Make `LockManager#_execUSDPriceUpdate()` more effective

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L506-L527

```solidity
    function _execUSDPriceUpdate() internal {
        if (
            usdUpdateProposal.approvalsCount >= APPROVE_THRESHOLD &&
            usdUpdateProposal.disapprovalsCount < DISAPPROVE_THRESHOLD
        ) {
            uint256 updateTokensLength = usdUpdateProposal.contracts.length;
            for (uint256 i; i < updateTokensLength; i++) {
                address tokenContract = usdUpdateProposal.contracts[i];
                if (configuredTokens[tokenContract].nftCost != 0) {
                    configuredTokens[tokenContract].usdPrice = usdUpdateProposal
                        .proposedPrice;

                    emit USDPriceUpdated(
                        tokenContract,
                        usdUpdateProposal.proposedPrice
                    );
                }
            }

            delete usdUpdateProposal;
        }
    }
```

The `usdUpdateProposal.approvalsCount >= APPROVE_THRESHOLD` check is redundant as the only instance this function gets called is from `approveUSDPrice()` when the `usdUpdateProposal.approvalsCount >= APPROVE_THRESHOLD)`, see `approveUSDPrice()`: https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L177-L207

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

        usdUpdateProposal.approvals[msg.sender] = _usdProposalId;
        usdUpdateProposal.approvalsCount++;

        if (usdUpdateProposal.approvalsCount >= APPROVE_THRESHOLD) {
            _execUSDPriceUpdate();
        }

        emit ApprovedUSDPrice(msg.sender);
    }
```

### Impact

Unnecessary code execution, redundant checks hinting flaws in implementation.

### Recommended Mitigation Steps

Apply these changes: https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L506-L527

```diff
    function _execUSDPriceUpdate() internal {
        if (
-            usdUpdateProposal.approvalsCount >= APPROVE_THRESHOLD &&
            usdUpdateProposal.disapprovalsCount < DISAPPROVE_THRESHOLD
  ) {
            uint256 updateTokensLength = usdUpdateProposal.contracts.length;
            for (uint256 i; i < updateTokensLength; i++) {
                address tokenContract = usdUpdateProposal.contracts[i];
                if (configuredTokens[tokenContract].nftCost != 0) {
                    configuredTokens[tokenContract].usdPrice = usdUpdateProposal
                        .proposedPrice;

                    emit USDPriceUpdated(
                        tokenContract,
                        usdUpdateProposal.proposedPrice
                    );
                }
            }

            delete usdUpdateProposal;
        }
    }
```

## QA-06 Protocol limits itself with the amount of voters it could have

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L129-L140

```solidity
    function setUSDThresholds(
        uint8 _approve,//@audit uint8 is too small
        uint8 _disapprove
    ) external onlyAdmin {
        if (usdUpdateProposal.proposer != address(0))
            revert ProposalInProgressError();
        APPROVE_THRESHOLD = _approve;
        DISAPPROVE_THRESHOLD = _disapprove;

        emit USDThresholdUpdated(_approve, _disapprove);
    }

```

Evidently, there is a small 255 limitation on the amount of threshold since the `uint8` value is being used and the max is 255

### Impact

Would affect future implementation of the protocol if voters/threshold are to increase.

> Likelihood is low, so only QA is justified.

### Recommended Mitigation Steps

Consider using a higher `uint` type.

## QA-07 Wrong duration could be set due to unsafe operations

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L245-L269

```solidity
    function setLockDuration(uint256 _duration) external notPaused {
        if (_duration > configStorage.getUint(StorageKey.MaxLockDuration))
            revert MaximumLockDurationError();

        playerSettings[msg.sender].lockDuration = uint32(_duration);//@audit duration is unsafely casted down to uint32
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
```

Evidently we can see that the duration is unsafely casted to `uint32` whereas it's accepted as `uint256`.

### Impact

Unsafe casting would then make the wrong duration to be set.

### Recommended Mitigation Steps

Apply the changes below to ensure the right duration is always set:

```diff
-    function setLockDuration(uint256 _duration) external notPaused {
+    function setLockDuration(uint32 _duration) external notPaused {
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
```

## QA-08 Setters don't have equality checkers

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L245-L269

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
                // check they are not setting lock time before current unlocktime @audit the check below should be `<=`
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
```

Evidently we can see that there is a check when setting the duration, that the new duration is not less than what's been previously set, problem however is that this implementation does not check if the new duration is `==` the current one which would then make the attempt to still set the value even if the new duration is exactly as previously set.

### Impact

Unnecessary operations.

### Recommended Mitigation Steps

Apply the changes below

```diff
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
-                    uint32(block.timestamp) + uint32(_duration) <
+                    uint32(block.timestamp) + uint32(_duration) <=
                    lockedTokens[msg.sender][tokenContract].unlockTime
                ) {
-                    revert LockDurationReducedError();
+                    revert LockDurationReducedOrNotChangedError();
                }

                uint32 lastLockTime = lockedTokens[msg.sender][tokenContract]
                    .lastLockTime;
                lockedTokens[msg.sender][tokenContract].unlockTime =
                    lastLockTime +
                    uint32(_duration);
            }
        }
```

## QA-09 Make `LockManager#lockOnBehalf()` more effective

### Proof of Concept

Take a look at https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L275-L295

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
  //@audit
        _lock(_tokenContract, _quantity, tokenOwner, lockRecipient);/
    }

```

This function is used whenever there is a need to lock on the behalf of another user, issue however is that the `tokenOwner` is always going to be the `msg.sender` so we should make the function more effective by just sending the msg.sender directly via `_lock()` instead of caching tokenOwner.

### Impact

Unnecessary caching, leading to more executional costs.

### Recommended Mitigation Steps

Apply these changes https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L275-L295

```diff
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
-        address tokenOwner = msg.sender;
        address lockRecipient = msg.sender;
        if (_onBehalfOf != address(0)) {
            lockRecipient = _onBehalfOf;
        }
  //@audit
-        _lock(_tokenContract, _quantity, tokenOwner, lockRecipient);
+        _lock(_tokenContract, _quantity, msg.sender, lockRecipient);
    }

```
---
>Edit: I assume QA-03 & QA-04 might be more impactful than QA to protocol so also submitted reports on similar bug ideas as med severity incase this QA gets dropped during validation.

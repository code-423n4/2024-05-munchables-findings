# L1. Using uint32 for block.timestamp is generally not sufficient for representing block timestamps

## Impact
This range covers timestamps up to around the year 2106.

## Proof of Concept
```solidity
File: src/interfaces/ILockManager.sol#L61
    struct USDUpdateProposal {
        uint32 proposedDate; // @audit around year 2106?
        address proposer;
        address[] contracts;
        uint256 proposedPrice;
        mapping(address => uint32) approvals;
        mapping(address => uint32) disapprovals;
        uint8 approvalsCount;
        uint8 disapprovalsCount;
    }
File: src/managers/LockManager.sol#L104
    uint32(block.timestamp)
File: src/managers/LockManager.sol#L166
    usdUpdateProposal.proposedDate = uint32(block.timestamp);
File: src/managers/LockManager.sol#L249
    playerSettings[msg.sender].lockDuration = uint32(_duration);
File: src/managers/LockManager.sol#L257
    uint32(block.timestamp) + uint32(_duration) <
File: src/managers/LockManager.sol#L267
    uint32(_duration);
File: src/managers/LockManager.sol#L353
    lockdrop.start <= uint32(block.timestamp) &&
File: src/managers/LockManager.sol#L354
    lockdrop.end >= uint32(block.timestamp)
File: src/managers/LockManager.sol#L359
    uint32(configStorage.getUint(StorageKey.MaxLockDuration))
File: src/managers/LockManager.sol#L381
    lockedToken.lastLockTime = uint32(block.timestamp);
File: src/managers/LockManager.sol#L383
    uint32(block.timestamp) +
File: src/managers/LockManager.sol#L384
    uint32(_lockDuration);
File: src/managers/LockManager.sol#L410
    if (lockedToken.unlockTime > uint32(block.timestamp))
```

## Tools Used
Manual review

## Recommended Mitigation Steps
It's recommended to use uint48 for timestamps.




# L2. If APPROVE_THRESHOLD is 1, the _execUSDPriceUpdate() function should be invoked in the proposeUSDPrice function.

## Impact
APPROVE_THRESHOLD can be set as 1, so it's needed to check if approvalsCount is greater than APPROVE_THRESHOLD in the proposedUSDPrice function.

## Proof of Concept

https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L171
```solidity
File: src/managers/LockManager.sol#L171
    function proposeUSDPrice(
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
    {
        ...
        usdUpdateProposal.approvalsCount++;

        emit ProposedUSDPrice(msg.sender, _price);
    }
```

## Tools Used
Manual review

## Recommended Mitigation Steps
It's recommended to add the validation for thresholds.
```diff
File: src/managers/LockManager.sol#L171
    function proposeUSDPrice(
        uint256 _price,
        address[] calldata _contracts
    ) {
        ...
        usdUpdateProposal.approvalsCount++;
+        if (usdUpdateProposal.approvalsCount >= APPROVE_THRESHOLD) {
+            _execUSDPriceUpdate();
+        }

        emit ProposedUSDPrice(msg.sender, _price);
    }
```


# L3. There are missing checks for the _approve and _disapprove thresholds.

## Impact
The thresholds for _approve and _disapprove should be validated.

## Proof of Concept

https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L135
```solidity
File: src/managers/LockManager.sol#L135
    function setUSDThresholds(
        uint8 _approve,
        uint8 _disapprove
    ) external onlyAdmin {
        if (usdUpdateProposal.proposer != address(0))
            revert ProposalInProgressError();
        APPROVE_THRESHOLD = _approve; //@audit should be >1
        DISAPPROVE_THRESHOLD = _disapprove; // should be >=1?

        emit USDThresholdUpdated(_approve, _disapprove);
    }
```

## Tools Used
Manual review

## Recommended Mitigation Steps
It's recommended to add the validation for thresholds.
```diff
File: src/managers/LockManager.sol#L135
    function setUSDThresholds(
        uint8 _approve,
        uint8 _disapprove
    ) external onlyAdmin {
        if (usdUpdateProposal.proposer != address(0))
            revert ProposalInProgressError();
+        require(_approve > 1);
+        require(_disapprove >= 1);
        APPROVE_THRESHOLD = _approve;
        DISAPPROVE_THRESHOLD = _disapprove;

        emit USDThresholdUpdated(_approve, _disapprove);
    }
```

# L4. A feeder who has already disapproved the USD price can approve it in the same proposal.

## Impact
There are no checks to ensure that msg.sender has not already disapproved the price in the approveUSDPrice function. This means a feeder who has previously disapproved the same price can also approve it.

## Proof of Concept

https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L177
```solidity
File: src/managers/LockManager.sol#L177
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
        if (usdUpdateProposal.proposer == msg.sender) // @audit missed disapprovals checks
            revert ProposerCannotApproveError();
        if (usdUpdateProposal.approvals[msg.sender] == _usdProposalId)
            revert ProposalAlreadyApprovedError();
        if (usdUpdateProposal.proposedPrice != _price)
            revert ProposalPriceNotMatchedError();
        ...
    }
```

## Tools Used
Manual review

## Recommended Mitigation Steps
It is advised to include validation to check if it has already been disapproved.

```diff
File: src/managers/LockManager.sol#L177
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
+        if (usdUpdateProposal.disapprovals[msg.sender] == _usdProposalId)
+            revert ProposalAlreadyDisapprovedError();
        if (usdUpdateProposal.proposedPrice != _price)
            revert ProposalPriceNotMatchedError();
        ...
    }
```

# L5. Proposer can disapprove the usdprice in the same proposal.

## Impact
There are no checks to ensure that msg.sender is not the proposer in the disapproveUSDPrice function. This means the proposer can also disapprove the proposal.

## Proof of Concept

https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L225
```solidity
File: src/managers/LockManager.sol#L210
    function disapproveUSDPrice(
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
        if (usdUpdateProposal.approvals[msg.sender] == _usdProposalId) // @audit missed proposer is msg.sender?
            revert ProposalAlreadyApprovedError();
        if (usdUpdateProposal.disapprovals[msg.sender] == _usdProposalId)
            revert ProposalAlreadyDisapprovedError();
        if (usdUpdateProposal.proposedPrice != _price)
            revert ProposalPriceNotMatchedError();
        ...
    }
```

## Tools Used
Manual review

## Recommended Mitigation Steps
It is recommended to add validation to ensure that the caller is the proposer.

```diff
File: src/managers/LockManager.sol#L177
    function disapproveUSDPrice(
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
+        if (usdUpdateProposal.proposer == msg.sender)
+            revert ProposerCannotDisapproveError();
        if (usdUpdateProposal.approvals[msg.sender] == _usdProposalId)
            revert ProposalAlreadyApprovedError();
        if (usdUpdateProposal.disapprovals[msg.sender] == _usdProposalId)
            revert ProposalAlreadyDisapprovedError();
        if (usdUpdateProposal.proposedPrice != _price)
            revert ProposalPriceNotMatchedError();
        ...
    }
```

# L6. Missing the _price validation in the proposedUSDPrice
```solidity
File: src/managers/LockManager.sol#L171
159:     if (_contracts.length == 0) revert ProposalInvalidContractsError();
+        if (_price == 0) revert USDPriceInvalidError();
```

# NC1. Event missing indexed field

Index event fields make the field more quickly accessible [to off-chain tools](https://ethereum.stackexchange.com/questions/40396/can-somebody-please-explain-the-concept-of-event-indexing) that parse events. This is especially useful when it comes to filtering based on an address. However, note that each index field costs extra gas during emission, so it's not necessarily best to index the maximum allowed per event (three fields). Where applicable, each `event` should use three `indexed` fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three applicable fields, all of the applicable fields should be indexed.

```solidity
File: src/interfaces/ILockManager.sol
159:     event TokenConfigured(address _tokenContract, ConfiguredToken _tokenData); 
168:     event ProposedUSDPrice(address _proposer, uint256 _price);
172:     event ApprovedUSDPrice(address _approver);
176:     event DisapprovedUSDPrice(address _disapprover);
225:     event USDPriceUpdated(address _tokenContract, uint256 _newPrice);
```

# NC2. Some errors are not used

Remove the unused error types.

```solidity
File: src/interfaces/ILockManager.sol
228:     error OnlyAccountManagerError();
245:     error USDPriceInvalidError();
293:     error InvalidTokenContractError();

```

# NC3. Some events are not used
Remove the unused event types.

```solidity
File: src/interfaces/ILockManager.sol
220:     event DiscountFactorUpdated(uint256 discountFactor);
```

# NC4. If LockManager does not handle receiving the native token, there is no need to declare the receive() function.
```solidity
File: src/interfaces/ILockManager.sol
93:     receive() external payable {
94:        revert LockManagerRefuseETHError();
95:    }
```


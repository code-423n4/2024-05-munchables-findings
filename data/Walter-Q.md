## 1
## Missing Validation Checks for decimals and usdPrice in configureToken Function
A potential issue has been identified within the ``configureToken`` function related to the validation of token data. Specifically, there is no check to ensure that the decimals value in the ``_tokenData`` parameter is different from 0, and that the usdPrice is equal to 0.

This is important because if decimals is 0 and usdPrice is different from 0, the price will later be modified by the price feed, which may lead to unexpected behavior or incorrect pricing.

### affected code:
```
function configureToken(
    address _tokenContract,
    ConfiguredToken memory _tokenData
) external onlyAdmin {
    if (_tokenData.nftCost == 0) revert NFTCostInvalidError();
    // @audit - add here all the validation checks
    if (configuredTokens[_tokenContract].nftCost == 0) {
        configuredTokenContracts.push(_tokenContract);
    }
    configuredTokens[_tokenContract] = _tokenData;

    emit TokenConfigured(_tokenContract, _tokenData);
}
```
[LockManager#115-127]("https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L115-L127")

### Recommended fix
Add checks to ensure that the decimals field in ``_tokenData`` is not 0 and that the usdPrice is 0 before assigning ``_tokenData`` to ``configuredTokens[_tokenContract]``.

## 2
## Missing Validation Checks for setUSDThresholds Function
The ``setUSDThresholds`` function is intended to set the thresholds for approvals and disapprovals of a price by the price feed contracts. However, the function lacks validation checks for the ``_approve`` and ``_disapprove`` parameters. Specifically, these values must be between 2 and 5 (inclusive). If the thresholds are not properly validated, it can lead to invalid configurations that disrupt the voting process.
### Code Snippet
```
function setUSDThresholds(
    uint8 _approve,
    uint8 _disapprove
) external onlyAdmin {
    if (usdUpdateProposal.proposer != address(0))
        revert ProposalInProgressError();
    // @audit approve and disapprove need to be >= 2 and <= 5
    APPROVE_THRESHOLD = _approve;
    DISAPPROVE_THRESHOLD = _disapprove;

    emit USDThresholdUpdated(_approve, _disapprove);
}
```
[LockManager#129-139]("https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L129-L139")

### Potential Impact
Without proper validation, thresholds could be set to values that are either too low or too high. If the threshold is set to 1, it undermines the voting process, as the proposer is always counted as one vote. And for the fact that in the propose function there's no if statement that checks approvals,this will skip the first vote,making the first effective valide threshold at 2.
## Recommended Fix
Add checks to ensure that the ``_approve`` and ``_disapprove`` values are between 2 and 5 inclusive. This ensures that the voting process is not compromised by invalid threshold values.

## 3
## Unnecessary Deletion of usdUpdateProposal in proposeUSDPrice Function
The ``proposeUSDPrice`` function is responsible for proposing a new price for an asset by the price feed contracts. Within this function, the ``usdUpdateProposal`` struct is instantiated to temporarily store the proposed values. This struct is used throughout the proposal process and is either approved or disapproved through separate functions (``approveUSDPrice`` and ``disapproveUSDPrice``). When a proposal reaches a threshold for approval or disapproval, the proposal is either executed or discarded, and the struct is reset to make way for new proposals.

The issue is that the line ``delete usdUpdateProposal;`` is unnecessarily present at the beginning of the ``proposeUSDPrice`` function. This deletion is redundant because the ``usdUpdateProposal`` struct will be reset regardless of whether the proposal is approved or disapproved. Therefore, this line of code does not contribute to the function's purpose and can be removed to clean up the code.

### Code Snippet
```
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
    if (usdUpdateProposal.proposer != address(0))
        revert ProposalInProgressError();
    if (_contracts.length == 0) revert ProposalInvalidContractsError();

    delete usdUpdateProposal; // @audit QA no need to delete 

    // Approvals will use this because when the struct is deleted the approvals remain
    ++_usdProposalId;

    usdUpdateProposal.proposedDate = uint32(block.timestamp);
    usdUpdateProposal.proposer = msg.sender;
    usdUpdateProposal.proposedPrice = _price;
    usdUpdateProposal.contracts = _contracts;
    usdUpdateProposal.approvals[msg.sender] = _usdProposalId;
    usdUpdateProposal.approvalsCount++;

    emit ProposedUSDPrice(msg.sender, _price);
}
```
[LockManager#142-174]("https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L142-L174")

### Potential Impact
While this issue does not affect the functionality or security of the contract, it represents unnecessary code that could be removed to improve readability and maintainability.

### Recommended Fix
Remove the line ``delete usdUpdateProposal;`` from the ``proposeUSDPrice`` function to eliminate redundant code.

## 4
## Missing Check for Prior Disapproval in approveUSDPrice Function
The ``approveUSDPrice`` function allows price feed contracts to vote in favor of a proposed price. However, unlike the ``disapproveUSDPrice`` function, ``approveUSDPrice`` does not verify whether the voter has already voted against the current proposal. A price feed contract should only be allowed to vote on one side (approve or disapprove) of the proposal. This missing check can lead to inconsistent voting behavior and potential conflicts in the proposal process.

### Affected code
```
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
    // @audit - LOW should check even disapprovals
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
[LockManager#177-207]("https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L177-L207")

### Potential Impact
Without a check to prevent a voter who has disapproved the proposal from approving it, the voting process can become inconsistent, leading to potential manipulation or invalid results. This could undermine the integrity of the price update mechanism.

### Recommended Fix
Add a check in the ``approveUSDPrice`` function to ensure that a voter who has already disapproved the proposal cannot approve it.

## 5
## Users can bypass minLockDuration check
The ``setLockDuration`` function allows setting the lock time for a specific asset locked in the contract. While it correctly checks that the provided duration is less than the ``MaxLockDuration``, it does not ensure that the duration is greater than the ``MinLockDuration``. This omission can lead to lock durations that are too short, bypassing the intended minimum lock period.
```
function setLockDuration(uint256 _duration) external notPaused {
    if (_duration > configStorage.getUint(StorageKey.MaxLockDuration))
        revert MaximumLockDurationError();
    // @audit - LOW - need to add minLockDuration this could bypass minLockDuration check (present in lock,but not here)

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
[LockManager#245-272]("https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L245-L272")

### Potential Impact
Failing to enforce a minimum lock duration can allow users to set lock times that are too short, which might not comply with the intended locking policies. This can lead to issues where assets are unlocked too soon, defeating the purpose of having a minimum lock duration.

### Recommended Fix
Add a check to ensure that the ``_duration`` is also greater than or equal to the ``MinLockDuration``.

## 6
## Missing Check for active Status in _lock Function
The ``_lock`` function allows locking tokens within the contract, but it lacks a check for the ``active`` status of the ``ConfiguredToken``. According to the provided documentation in the codebase:
```
    /// @notice Struct holding details about tokens that can be locked
    /// @param usdPrice USD price per token
    /// @param nftCost Cost of the NFT associated with locking this token
    /// @param decimals Number of decimals for the token
    /// @param active Boolean indicating if the token is currently active for locking
    struct ConfiguredToken {
        uint256 usdPrice;
        uint256 nftCost;
        uint8 decimals;
        bool active;
    }
```
[ILockManager#17-27]("https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/interfaces/ILockManager.sol#L17-L27")

The ``active`` status should be verified before allowing tokens to be locked. However, this check is missing from the ``_lock`` function, which may allow users to lock tokens that are not intended to be locked according to the contract's configuration.

### Potential Impact
Without verifying the ``active`` status of the token, the ``_lock`` function may allow tokens to be locked even if they are not currently intended to be locked according to the contract's configuration. This could lead to unexpected behavior and potential misuse of the locking mechanism.

### Recommended Fix
Add a check to ensure that the ``active`` status of the ``ConfiguredToken`` is true before allowing tokens to be locked in the ``_lock`` function.
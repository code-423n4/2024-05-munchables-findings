### Title
PriceFeed can vote both against and in favor of confirming the new price at the same time 

### Severity
Low

### Target
src/managers/LockManager.sol

## Description
One of the roles can first call `disapproveUSDPrice()` and then call `approveUSDPrice()`, thus increasing the count of those who agree and disagree for the new token price, since the approveUSDPrice function does not check if one of the roles has voted against it. 

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

```solidity 
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
        if (usdUpdateProposal.approvals[msg.sender] == _usdProposalId) 
            revert ProposalAlreadyApprovedError();
        if (usdUpdateProposal.disapprovals[msg.sender] == _usdProposalId) 
            revert ProposalAlreadyDisapprovedError();
        if (usdUpdateProposal.proposedPrice != _price)
            revert ProposalPriceNotMatchedError();

        usdUpdateProposal.disapprovalsCount++; 
        usdUpdateProposal.disapprovals[msg.sender] = _usdProposalId; 

        emit DisapprovedUSDPrice(msg.sender);

        if (usdUpdateProposal.disapprovalsCount >= DISAPPROVE_THRESHOLD) {  
            delete usdUpdateProposal; 
            emit RemovedUSDProposal();
        }
    }
```


## Recommendations
it is worth adding to the `approveUSDPrice()` function to check `usdUpdateProposal.disapprovals[msg.sender] == _usdProposalId`
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
->      if (usdUpdateProposal.disapprovals[msg.sender] == _usdProposalId){
            revert ProposalAlreadyDisApprovedError();
}
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



Permalink:
https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L177-L242

---

### Title
Add a check for minimum lock duration  

### Severity
Low

### Target
src/managers/LockManager.sol

## Description
In the `configureLockdrop()` function, you should add a check that `_lockdropData.minLockDuration>=30 days` as specified in the [documentation](https://munchables.gitbook.io/munchables/explorers-guide/glossary/lockdrop#dictionary-definition)
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



## Recomendations 
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
->        if (_locdropData.minLockDuration < 30 days ) {
->            revert Error();
            }
        lockdrop = _lockdropData;

        emit LockDropConfigured(_lockdropData);
    }
 ```   

Permalink:
https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L98C5-L112C6

---

### Title
STATE VARIABLE VISIBILITY NOT SET

### Severity
Informational

### Target
src/managers/LockManager.sol

## Description
It is best practice to set the visibility of state variables explicitly.
```solidity
uint8 APPROVE_THRESHOLD = 3;
uint8 DISAPPROVE_THRESHOLD = 3;
...
mapping(address => PlayerSettings) playerSettings;
...
USDUpdateProposal usdUpdateProposal;
```

## Recommendations
It is recommended to explicitly specify the visibility of all state variables.


Permalink:
https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L16
https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L18
https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L26
https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L30

---

### Title
Unnecessary code repetition 

### Severity
Informational

### Target
src/managers/LockManager.sol

## Description
in the `LockManager::approveUSDPrice` function there is a condition 
```solidity
if (usdUpdateProposal.approvalsCount >= APPROVE_THRESHOLD) { 
            _execUSDPriceUpdate();
        }
```
if it is **true**, the `internal` function `_execUSDPriceUpdate()` is called, and this function will be executed only if this condition is **true**. 
```solidity 
 if (
            usdUpdateProposal.approvalsCount >= APPROVE_THRESHOLD && 
            usdUpdateProposal.disapprovalsCount < DISAPPROVE_THRESHOLD 
        ) {...}
```

## Recommendations
Consider removing unnecessary repetitive code 

Permalink:
https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L202-L204
https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L507-L510

---

### Title
Add the getter function

### Severity
Informational
### Target
src/managers/LockManager.sol

## Description
In the `approveUSDPrice(uint256 _price)` and `disapproveUSDPrice(uint256 _price)` functions, one of the roles passes to these functions the price they want to approve or reject. These functions have a check that the input must equal the suggested price. If it is not, it revert the error `ProposalPriceNotMatchedError`.

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
->      if (usdUpdateProposal.proposedPrice != _price)
            revert ProposalPriceNotMatchedError();
...}
```

## Recommendations
Consider adding a `getter function` for the convenience of all `PriceFeeds` that will return a `proposed price` 

```solidity 
function getProposedPrice() external returns (uint256){
    return usdUpdateProposal.proposedPrice;
}
```
Permalink:
https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L177C5-L207
https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L177C5-L207

---


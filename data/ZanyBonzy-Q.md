
## 1. Consider introducing an admin brute force function to execute or reject a price update. 

Links to affected code *

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L192

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L210

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L238

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L157


### Impact

Only the currently available five price feeds can propose usd price. If at one of them goes down, there'll be four available feeds. This can lead to a potential deadlock situation if 2 of the price feeds approve the price, and 2 disapprove. When this happens, the price cannot be updated and a new price cannot be proposed dossing the contract's operation. Thus a function to forcefully execute or reject a price update is needed to handle cases like this.


***

## 2. Getting the lockweighted value of certain tokens will be impossible due to the abscene of the `decimals` function.

Links to affected code *

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L461

### Impact

From the [ERC20](https://eips.ethereum.org/EIPS/eip-20#decimals) standard, the `decimals` function is optional.
> decimals
> Returns the number of decimals the token uses - e.g. 8, means to divide the token amount by 100000000 to get its user representation.

> OPTIONAL - This method can be used to improve usability, but interfaces and other contracts MUST NOT expect these values to be present.

COnsidering that the protocol aims to work with [different token types](https://github.com/code-423n4/2024-05-munchables/blob/main/README.md#general-questions), using a standard ERC20 token which lacks the decimals parameter will cause issues with the `getLockedWeightedValue`

> Question	Answer
> ERC20 used by the protocol	USDB, WETH, (assume we can add more to the future for use in LockManager)

The function is used outside of the in-scope contract, notably in the MigrationManager contract during migration, and having such a token that lacks the decimals function will cause migration to be impossible. Consider creating a special function to query decimals and wrapping the `.decimals` function in a trycatch block, returning 18 if the function fails.

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
                // We are assuming all tokens have a maximum of 18 decimals and that USD Price is denoted in 1e18
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


***


## 3. Protocol is not compatible with rebasing tokens 

Links to affected code *

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L423

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L376

### Impact

The protocol aims to work with [different token types](https://github.com/code-423n4/2024-05-munchables/blob/main/README.md#general-questions), using a standard ERC20 token which lacks the decimals parameter will cause issues with the `getLockedWeightedValue`

> Question	Answer
> ERC20 used by the protocol	USDB, WETH, (assume we can add more to the future for use in LockManager)


Protocol functions when locking and unlocking perform operations using inputed `quantity`. The functions however don't query the existing balance of tokens before or after receiving/sending in order to properly account for tokens that shift balance over time (rebasing). Depositing rebasing tokens into a system that does not support them will lead to several incorrect balance tracking due to rebases, leading to incorrect display or accounting of the user's holdings. The value of the user's deposit could be misrepresented, as the system might not update the user's balance in response to the token's supply changes. If users attempts to withdraw all of their tokens after a potential negative rebase, they'd be unable to do so since their quantity will be potentially more than the actual tokens available.

```solidity

        if (_tokenContract != address(0)) {
            IERC20 token = IERC20(_tokenContract);
            token.transferFrom(_tokenOwner, address(this), _quantity);
        }

```
```solidity
        } else {
            IERC20 token = IERC20(_tokenContract);
            token.transfer(msg.sender, _quantity);
        }
```


***

## 4. No need to check for contract allowance as the check is performed when calling `transferFrom`

Links to affected code *

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L328C1-L331C10

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L374C1-L377C10

### Impact

The _lock function checks for the contract's allowance from the token owner and further down performs the `transferFrom`. The `transferFrom` function also checks for the spenders allowance, making the contract's own allowance check not needed.
```solidity
    function _lock(
        address _tokenContract,
        uint256 _quantity,
        address _tokenOwner,
        address _lockRecipient
    ) private {
        ...
        else {
            if (msg.value != 0) revert InvalidMessageValueError();
            IERC20 token = IERC20(_tokenContract);
            uint256 allowance = token.allowance(_tokenOwner, address(this));
            if (allowance < _quantity) revert InsufficientAllowanceError();
        }
        ...
                // Transfer erc tokens
        if (_tokenContract != address(0)) {
            IERC20 token = IERC20(_tokenContract);
            token.transferFrom(_tokenOwner, address(this), _quantity);
        }
```



***

## 5. Pricefeeds that had disapproved a price proposal can also approve it

Links to affected code *

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L192-L197

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L224-L230

### Impact

The `approveUSDPrice` function doesn't check that the caller had not previously disapproved the price. This can lead to prices being updated without the actual needed consensus as it seems like the same price feed is double voting. If a contentious price is set and two price feed disapprove it, the same two can later on approve it, causing the price to reach the desired threshold, and for the update to be executed. Important to note that this also goes against the protcol's intended design as if a pricefeed had previously approved a price, they're [unable](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L225) to disapprove it.
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
      ...
    }
```

***
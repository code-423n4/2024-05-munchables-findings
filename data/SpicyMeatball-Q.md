### Wallet that disapproved token price can still approve it
The `approveUSDPrice` function doesn't check whether the user disapproved the price or not. 
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
    }
```

Consider validating `usdUpdateProposal.disapprovals[msg.sender]` to prevent users who have disapproved the price from approving it.
 
### No threshold validation
In the current `LockManager` implementation only 5 wallets can approve or disapprove prices, setting `APPROVE_THRESHOLD` or `DISAPPROVE_THRESHOLD` to a value greater than 5 will result in the inability to set a new price for the token.

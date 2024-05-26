# 1, Lack of time limit in propose can make price update slower

## Lines of code
https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L142-#L174

## Vulnerability details
When propose price, function `proposeUSDPrice()` will check if previous propose is done or not:

        if (usdUpdateProposal.proposer != address(0))
            revert ProposalInProgressError();
But in current mechanism, there is no limitation about the time propose can last long. If one or more proposer do not approve or disapprove, new price cant be proposed.

## Impact
Unable to propose new price if one or more proposer do not approve or disapprove price.

## Mitigration
Add time limitation that propose can last long. If the time is passed and there is no enough propose, that propose will be rejected
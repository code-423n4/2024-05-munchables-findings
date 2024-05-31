# **INFORMATIONAL**

# Info 01 Configured Check Misssing

## Summary
In the `LockManager::getConfiguredToken` and `LockManager::getPlayerSettings` functions, the contract fails to verify the existence of the records being queried. This oversight can lead to unintended behavior, potential errors, and security risks. To mitigate these issues, it is recommended to add checks ensuring that the records exist before accessing them. This will enhance the contract's reliability and security.

## Vulnerability Detail
The functions `LockManager::getConfiguredToken` and `LockManager::getPlayerSettings` do not check if the queried records exist. As a result, they might attempt to access data for non-existent or unconfigured tokens and players.

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L490-L494

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L496-L500

## Impact
- **Unintended Behavior:** The functions might return incorrect or default values if the records do not exist.
- **Potential Errors:** Attempting to access non-existent records can lead to errors or undefined behavior.
- **Security Risks:** The lack of validation can be exploited by attackers to manipulate the contract's behavior or extract unintended information

## Code Snippet
```solidity
function getConfiguredToken(address _tokenContract) external view returns (ConfiguredToken memory _token) {

    _token = configuredTokens[_tokenContract]; 

}

function getPlayerSettings(address _player) external view returns (PlayerSettings memory _settings) {

    _settings = playerSettings[_player];

}
```

## Tool used
Manual Review

## Recommendation
Implement checks in both of these functions.

# Info 02 Remove Fallback And Receive

## Summary
The implementation of both `fallback()` and `receive()` functions in the `LockManager` contract appears redundant, as they both revert transactions with custom errors. This redundancy potentially complicates contract logic and could lead to confusion for developers interacting with the contract.

## Vulnerability Detail
The `fallback()` and `receive()` functions in the `LockManager` contract are both designed to handle incoming Ether transfers. However, instead of performing any meaningful operations or processing these transfers, both functions simply revert the transactions with custom errors:

```solidity
fallback() external payable {
    revert LockManagerInvalidCallError();
}

receive() external payable {
    revert LockManagerRefuseETHError();
}
```

This creates unnecessary complexity and confusion in the contract codebase without adding any functionality.

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L89-L95

## Impact
The impact of this redundancy is primarily on code clarity and developer experience. Redundant functions can make the contract code harder to understand and maintain, increasing the risk of introducing errors during updates or modifications. Additionally, the presence of these functions may mislead developers into thinking that the contract supports certain operations related to Ether transfers, when in fact, it does not.

## Code Snippet
None

## Tool used
Manual Review

## Recommendation
To improve code clarity and simplify contract logic, consider removing the redundant `fallback()` and `receive()` functions if they serve no purpose other than reverting transactions. If the intention is to reject Ether transfers, this can be achieved directly in the contract's constructor or by adding a modifier to relevant functions. 

# Info 03 getLockedWeight Calculates Only Active Tokens

## Summary:
The `getLockedWeightedValue` function in the LockManager contract calculates the `_lockedWeightedValue` considering only active tokens. The variable `lockedWeighted` is initialized but not used, indicating redundant code that can be simplified for better efficiency and clarity.

## Vulnerability Detail:
In the `getLockedWeightedValue` function, the calculation of `_lockedWeightedValue` considers only active tokens, meaning the `lockedWeighted` variable initialized to 0 is redundant:
```solidity
uint256 lockedWeighted = 0;
```
This suggests redundant code that adds unnecessary complexity without contributing to the logic of the function. The calculation can directly assign the value to `_lockedWeightedValue` without the need for this intermediate variable.

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L461-L487

## Impact:
The impact of this redundancy is primarily on code clarity and potential gas inefficiency. Unnecessary variables increase the cognitive load for developers reading and maintaining the code. Additionally, the gas cost of deploying and executing the contract may be slightly higher due to the presence of redundant variables.

## Code Snippet
None

## Tool used
Manual Review

## Recommendation:
To improve code clarity and efficiency, consider removing the `lockedWeighted` variable and directly calculate the `_lockedWeightedValue` in the `getLockedWeightedValue` function based on active tokens. This simplifies the code and makes it easier to understand for developers. Always strive to eliminate redundant code and variables to improve readability and optimize gas usage in smart contracts. Additionally, thorough testing should be conducted after making these changes to ensure that the behavior of the contract remains unchanged.

# **LOW**

# Low 01 Only Configured Can Be Removed

## Summary
In the `LockManager::lock` and `LockManager::lockOnBehalf` functions, there are currently two modifiers in use: `LockManager::onlyConfiguredToken` and `LockManager::onlyActiveToken`. The `LockManager::onlyConfiguredToken` modifier can be removed because its functionality is already encompassed by the `LockManager::onlyActiveToken` modifier.

## Vulnerability Detail
The `LockManager` contract currently uses two modifiers to enforce constraints on token operations:

1. **onlyConfiguredToken**: Ensures that the supplied token is configured.
```solidity
    @notice Token supplied must be configured but can be inactive
    modifier onlyConfiguredToken(address _tokenContract) {
        if (configuredTokens[_tokenContract].nftCost == 0)
            revert TokenNotConfiguredError();
        _;
    }
```

2. **onlyActiveToken**: Ensures that the supplied token is both configured and active.
```solidity
    /// @notice Token supplied must be configured and active
    modifier onlyActiveToken(address _tokenContract) {
        if (!configuredTokens[_tokenContract].active)
            revert TokenNotConfiguredError();
        _;
    }
```

**Redundancy Issue:**
- The `onlyConfiguredToken` modifier checks if the `nftCost` of the token is zero, which is used to determine if a token is configured.
- The `onlyActiveToken` modifier checks if the token is active.

A token being active implies that it is also configured. Therefore, the `onlyActiveToken` modifier inherently ensures that the token is configured. This makes the `onlyConfiguredToken` check redundant when `onlyActiveToken` is used.

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L275-L294

## Impact
- **Redundant Checks:** Using both modifiers introduces redundant checks, leading to unnecessary code complexity.
- **Maintenance Challenges:** Redundant checks increase the maintenance burden and the risk of introducing inconsistencies during future updates.
- **Performance Overhead:** Although minimal, performing unnecessary checks can lead to slight performance inefficiencies, especially if these functions are called frequently.

## Code Snippet
None

## Tool used
Manual Review

## Recommendation
To simplify the code and eliminate redundancy, it is recommended to remove the `onlyConfiguredToken` modifier from functions where `onlyActiveToken` is used, as `onlyActiveToken` already ensures the token is configured.

# Low 02 User Can Approve And Dissaprove

## Summary
There are two functions responsible for approving and disapproving proposals. Users are expected to either approve or disapprove a proposal, but not both. Currently approve function doesn't check if the user was already disapproved.

## Vulnerability Detail
In the `LockManager` contract, there are two functions for managing proposals: `approveUSDPrice` and `disapproveUSDPrice`. Users are expected to either approve or disapprove a proposal, but not both.

**Disapprove Function:**

The `disapproveUSDPrice` function correctly checks if the user has already either approved or disapproved the proposal. If either condition is true, the function reverts to prevent the user from performing conflicting actions.

**Approve Function:**

The current implementation of the `approveUSDPrice` function only checks if the user has already approved the proposal. It does not verify if the user has previously disapproved the proposal, creating an inconsistency.

This omission allows a user who has disapproved a proposal to later approve it, which contradicts the intended logic and can lead to inconsistent proposal states.

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L177-L207

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L210-L242

## Impact
The lack of a disapproval check in the `approveUSDPrice` function can result in a proposal being both approved and disapproved by the same user, leading to inconsistencies.

## Code Snippet
None

## Tool used
Manual Review

## Recommendation
Add this line of code tot `LockManager:approveUSDPrice` function:
```solidity
if (usdUpdateProposal.disapprovals[msg.sender] == _usdProposalId) 
    revert ProposalAlreadyDisapprovedError();
```

# Low 03 USD Price Is Not Denotated

## Summary
In the `LockManager` contract, the price of the USD is expected to be denoted as 1e18, but this standard is not enforced during the proposal process. Without proper validation, proposals may be submitted with USD prices that do not conform to this requirement, potentially leading to incorrect calculations in the `LockManager::getLockedWeightedValue`.

## Vulnerability Detail
In the `LockManager` contract, the price of the USD is expected to be denoted in wei, which is 1e18. However, the contract does not currently check if the price of the USD is correctly denoted in the `LockManager::proposeUSDPrice`.

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L461-L487

## Impact
This could lead to incorrect calculation in the function `LockManager::getLockedWeightedValue`.

## Code Snippet
None

## Tool used
Manual Review

## Recommendation
Implement a validation check to ensure that the price of the USD is correctly denoted(1e18) during the proposal process. This will ensure consistency and accuracy in the representation of the USD price within the contract.

# Low 04 Use Block Number

## Summary
In the `LockManager` contract, the usage of `block.timestamp` instead of `block.number` for time-related functionalities introduces vulnerabilities due to its susceptibility to manipulation and external factors. This can lead to time-based attacks, front-running vulnerabilities, and unpredictable behavior in the contract.

## Vulnerability Detail
The contract's reliance on `block.timestamp` makes it vulnerable to time manipulation attacks by miners or attackers. Additionally, using `block.timestamp` can expose the contract to front-running attacks and unpredictable behavior due to its dependence on external factors.

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L101

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L104

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L166

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L257

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L353-L354

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L381

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L383

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L410

## Impact
- **Time Manipulation:** Attackers can manipulate `block.timestamp` to exploit time-sensitive functionalities in the contract.
- **Front-Running Attacks:** Dependence on `block.timestamp` can make the contract susceptible to front-running attacks, where attackers exploit timing differences for their advantage.
- **Unpredictable Behavior:** External factors affecting `block.timestamp`, such as network conditions, can lead to unpredictable behavior in the contract.

## Code Snippet
None

## Tool used
Manual Review

## Recommendation
Replace the usage of `block.timestamp` with `block.number` for time-related functionalities in the `LockManager` contract. `block.number` provides a more reliable measure of time, reducing the vulnerability to time-based attacks and ensuring more predictable contract behavior.

# Low 05 Add Cancel For Proposals

## Summary
In the current `LockManager` contract implementation, only one proposal can exist at a time, making the ability to cancel proposals essential for user control and system flexibility. To address this limitation and enhance user experience, consider implementing a cancellable mechanism for proposals using timestamps.

## Vulnerability Detail
The current `LockManager` contract's restriction of allowing only one proposal at a time creates a situation where a proposal can potentially remain active for an extended period. This can lead to inefficiencies and frustrations for users, especially if they need to wait for a long time before submitting a new proposal.

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L14

## Impact
The absence of a cancellable mechanism limits the system's ability to adapt to changing circumstances, potentially causing inefficiencies and delays.

## Code Snippet
None

## Tool used
Manual Review

## Recommendation
Implementing a cancellable mechanism for proposals using timestamps will address this vulnerability, providing users with greater control over their proposals and mitigating the risk of inefficiencies and frustrations.

# Low 06 User Can Lock Outside Drop Period

## Summary
The vulnerability in the `LockManager::_lock` function permits users to lock funds into the contract regardless of whether it's within the designated airdrop period. Consequently, users may lock their funds without receiving the associated NFT, as the NFT distribution mechanism relies on the airdrop period. Enhancements are necessary to ensure that users receive NFTs when locking funds into the contract, irrespective of the timing relative to the airdrop period.

## Vulnerability Detail
The current implementation of the `LockManager::_lock` function allows users to lock funds into the contract even outside of the designated airdrop period. This vulnerability enables users to lock their funds without receiving an associated NFT, as the mechanism to distribute NFTs is tied to the airdrop period. Without proper validation, users may inadvertently lock their funds without receiving the expected NFT, leading to confusion, frustration, and potential loss of funds.

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L311-L398

## Impact
- **Loss of Funds:** Users may lock funds into the contract without receiving the corresponding NFT, resulting in a loss of value or assets.
- **User Frustration:** Lack of clarity and assurance regarding NFT distribution can lead to frustration and dissatisfaction among users.

## Code Snippet
```solidity
if (lockdrop.start <= uint32(block.timestamp) && lockdrop.end >= uint32(block.timestamp)) {
    if (_lockDuration < lockdrop.minLockDuration || _lockDuration >
             uint32(configStorage.getUint(StorageKey.MaxLockDuration))) 
        revert InvalidLockDurationError();

    if (msg.sender != address(migrationManager)) {
    }
}
```

## Tool used
Manual Review

## Recommendation
To mitigate this vulnerability, it's advisable to revise the `LockManager::_lock` function to revert transactions when funds are locked outside of the designated airdrop period. By implementing this validation check, users will be prevented from locking funds into the contract without the assurance of receiving the associated NFT. Additionally, clarity should be provided to users regarding the airdrop period and the requirement to lock funds only within this period to ensure NFT distribution. This approach will enhance transparency and user confidence in the contract's functionality.

# Low 07 DoS In setLockDuration

## Summary
The `LockManager::setLockDuration` function will become unusable after Sunday, 7 February 2106, UTC, due to a vulnerability in the way it handles time calculations. Specifically, the function converts timestamps to `uint32`, leading to a potential Denial of Service (DoS) vulnerability as of this date.

## Vulnerability Detail
The vulnerability arises from the following code snippet within the `LockManager::setLockDuration` function:

```solidity
if (
    uint32(block.timestamp) + uint32(_duration) <
    lockedTokens[msg.sender][tokenContract].unlockTime
)
```

Here, `block.timestamp` and `_duration` are being cast to `uint32`. When the sum of `block.timestamp` and `_duration` exceeds the maximum value representable by `uint32`, it will wrap around to zero, leading to incorrect calculations. This can allow an attacker to manipulate the unlock time, potentially causing a DoS by locking tokens indefinitely.

https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L257

## Impact
The impact of this vulnerability is significant. It allows an attacker to exploit the function to lock tokens indefinitely, effectively denying access to legitimate users. This could disrupt token functionality, cause financial losses, and damage the reputation of the affected system.

## Code Snippet
None

## Tool used
Manual Review

## Recommendation
To address this vulnerability, the code should be revised to use a safer method for time calculations that does not rely on `uint32` conversions. 
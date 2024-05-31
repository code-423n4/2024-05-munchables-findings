## Low Risk Findings

| Number | issue                                                                 | Instances |
| ------ | --------------------------------------------------------------------- | --------: |
| [L-01] | Missing Input validation Min lock               |         1 |
| [L-02] |`getLockedWeightedValue` could be out-dated                      |         1 |
| [L-03] | NO way to recover mistakenly sent tokens |         1 |
| [L-04] | Inability to Unlock Tokens Locked on Behalf of Another User   |         1 |
| [L-05] | Enforce a minimum unlock amount   |         1 |

                                   
## [L-01]  Missing Input validation Min lock 

While performing the lock operations there is no way of ensuring that users  cannot lock beyond a certain minimum which could make users lock dust amounts that could 
harm the protocol

consider adding a minimum quantity to lock that will be required, this will avoid locking very small and negligable amounts

***Instances***
https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L311

## [L-02] `getLockedWeightedValue` could be out-dated 

The function `getLockedWeightedValue` in the current implementation may produce outdated or inaccurate results because it does not invoke the `forceHarvest` function. This can lead to a misrepresentation of a user's locked value at the time, which could affect decision-making processes and the overall user experience.

Proof of Concept
The `getLockedWeightedValue` function calculates the total weighted value of a user's locked tokens. Here is the relevant code snippet:

```javascript
function getLockedWeightedValue(
    address _player
) external view returns (uint256 _lockedWeightedValue) {
    uint256 lockedWeighted = 0;
    uint256 configuredTokensLength = configuredTokenContracts.length;
    for (uint256 i; i < configuredTokensLength; i++) {
        if (
            lockedTokens[_player][configuredTokenContracts[i]].quantity > 0 &&
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

The function does not call accountManager.forceHarvest(_player), which means any pending rewards that should be harvested and added to the user's account are not accounted for. This can result in an inaccurate calculation of the user's total locked value.

Without the forceHarvest call, the internal state of rewards and locked amounts might not be up-to-date. This can cause the function to return a lower weighted value than what the user actually holds if rewards have accrued but not yet been harvested.

## Recommendation
Integrate the `forceHarvest` function within it. This will ensure that all pending rewards are accounted for before calculating the total locked weighted value.

***Instances***
https://github.com/code-423n4/2024-05-munchables/tree/main/src/managers#L461


## [L-03] : No way to recover mistakenly sent tokens

Based on the README 

```
ERC20 used by the protocol	USDB, WETH, (assume we can add more to the future for use in LockManager)
```

we are asked to assume more tokens can be added to the future for use in the LockManager, at the moment
while unlocking the tokens the following code is run

```javascript
// send token
        if (_tokenContract == address(0)) {
            payable(msg.sender).transfer(_quantity);
        } else {
            IERC20 token = IERC20(_tokenContract);
            token.transfer(msg.sender, _quantity); 
        }
```
while sending the token they transfer quantity to `msg.sender`, However, based on the README when in future they were to add USDC, there could be issues when the msg.sender is blacklisted, considering the contract is not upgradeable it will be great to allow users to be able to provide an address they would like the tokens to be sent to instead of sending directly to the `msg.sender` blocklists are out of scope but given the scenario the issue could arise

***Instances***
https://github.com/code-423n4/2024-05-munchables/tree/main/src/managers#L423

##  [L-04] : Inability to Unlock Tokens Locked on Behalf of Another User 

Impact
Users have the ability of locking tokens on behalf of other users, however, the current implementation of the unlock function prevents a user from unlocking tokens on behalf of another user. This restriction could limit the flexibility and functionality of the locking mechanism, especially in scenarios where one user locks tokens on behalf of another and needs to facilitate the unlocking process as well. Specifically, the function checks `msg.sender` against the locked tokens, which means only the user who locked the tokens can unlock them.

Proof of Concept (PoC)
Consider the following scenario:

Bob locks tokens on behalf of Alicew using the `lockOnBehalf` function.
Alice still needs Bob to do everything elese including  unlocking these tokens  after a given duration.
When Bob attempts to unlock the tokens, the function checks the locked tokens associated with `msg.sender` Bob, not Alice.
consider this part in the `unlock` function:

```javascript
LockedToken storage lockedToken = lockedTokens[msg.sender][
        _tokenContract
    ];
```
In this implementation:

The function retrieves the locked tokens using lockedTokens[msg.sender][_tokenContract].
msg.sender is the caller of the unlock function, not the recipient of the locked tokens.
As a result, Bob cannot unlock the tokens for Alice, even though Bob initially locked them on Alice behalf.

Recommendation
Modify the `unlock` function to allow the unlocking of tokens on behalf of another user. This can be done by accepting an additional parameter for the recipient address and verifying that the caller has the right to unlock the tokens for the specified recipient.

```javascript
LockedToken storage lockedToken = lockedTokens[_lockRecipient][
        _tokenContract
    ];
```

***Instaces***
https://github.com/code-423n4/2024-05-munchables/tree/main/src/managers#L401

## [L-05] : Enforce a minimum unlock amount

The current implementation of the `unlock` function in the contract exposes it to a potential denial-of-service (DoS) attack.An attacker could exploit this vulnerability by pushing numerous unlocks with very small quantity, overwhelming the system and causing it to become unusable. This attack is possible because of gas costs.

Proof of Concept:

Attack Setup:
The attacker, Alice, spends a significant amount on gas to push many unlocks with very small quantity.
Gas Consumption:
Each unlock, incurs gas costs for processing.

Recommended Mitigation Steps:
To mitigate this vulnerability, a minimum unlockquantity amount should be enforced in the `unlock` function. By setting a minimum unlock quantity, the contract can prevent potential DoS attacks.

***Instaces***
https://github.com/code-423n4/2024-05-munchables/tree/main/src/managers#L401



## Non-Critical Issues

| Number  |                      issue                       | Instances |
| ------- | :----------------------------------------------: | --------: |
| [NC-01] |              Activate the Optimizer              |         1 |
| [NC-02] | Navigating new frontiers in transaction fairness |         1 |
| [NC-03] |        unlock Process Not Pull-Based         |         1 |
| [NC-04] |       Missing checks for duplicate tokens        |         1 |
| [NC-05] |        block.timestamp can be manipulated        |         1 |

## [NC-01] : Activate the Optimizer

Before deploying your contract, activate the optimizer when compiling using “solc —optimize —bin sourceFile.sol”. By default, the optimizer will optimize the contract assuming it is called 200 times across its lifetime. If you want the initial contract deployment to be cheaper and the later function executions to be more expensive, set it to “ —optimize-runs=1”. Conversely, if you expect many transactions and do not care for higher deployment cost and output size, set “—optimize-runs” to a high number.

module.exports = {
solidity: {
version: "0.8.24",
settings: {
optimizer: {
enabled: true,
runs: 1000,
},
},
},
};
Please visit this site for further information:

https://docs.soliditylang.org/en/v0.5.4/using-the-compiler.html#using-the-commandline-compiler

Here’s one example of instance on opcode comparison that delineates the gas saving mechanism:

for !=0 before optimization
PUSH1 0x00
DUP2
EQ
ISZERO
PUSH1 [cont offset]
JUMPI
after optimization
DUP1
PUSH1 [revert offset]
JUMPI
Disclaimer: There have been several bugs with security implications related to optimizations. For this reason, Solidity compiler optimizations are disabled by default, and it is unclear how many contracts in the wild actually use them. Therefore, it is unclear how well they are being tested and exercised. High-severity security issues due to optimization bugs have occurred in the past.

## [NC-02] : Navigating new frontiers in transaction fairness

Navigating new frontiers in transaction fairness
As Layer 2, like Arbitrum or Base, considers moving towards a more decentralized sequencer model, the platform faces the challenge of maintaining its current mitigation of frontrunning risks inherent in a “first come, first served” system.

The transition could reintroduce vulnerabilities to transaction ordering manipulation, demanding innovative solutions to uphold transaction fairness. Strategies such as commit-reveal schemes, submarine sends, Fair Sequencing Services (FSS), decentralized MEV mitigation techniques, and the incorporation of time-locks and randomness could play pivotal roles. These measures aim to preserve the integrity of transaction sequencing, ensuring that the L2’s evolution towards decentralization enhances its ecosystem without compromising the security and fairness that are crucial for user trust and platform reliability.


## [NC-04]  : unlock Process Not Pull-Based 

The current unlock contains a  withdrawal process in the contract is not following the pull-based approach.
Contract proactively sends funds to users instead of allowing users to pull funds themselves.
Proof of Concept:

 the contract initiates the transfer of funds to users.
Funds are sent to users using 

```javascript
// send token
        if (_tokenContract == address(0)) {
            payable(msg.sender).transfer(_quantity);
        } else {
            IERC20 token = IERC20(_tokenContract);
            token.transfer(msg.sender, _quantity); 
        }

```

Recommendation:

Implement a pull-based approach where users initiate the withdrawal process and pull funds from the contract themselves.
Modify the withdrawal function to deduct the requested amount from the user's balance and hold it until the user requests to withdraw it.

## [NC-05] :  Missing checks for duplicate tokens


Implement a check  to ensure that the token addresses provided in the  array are unique. This can be done by using a mapping to track unique tokens and reverting if a token address is encountered more than once.

Updated `configureToken` function with Unique Token Check:

```javascript
function configureToken(
        address _tokenContract,
        ConfiguredToken memory _tokenData
    ) external onlyAdmin {
        if (_tokenData.nftCost == 0) revert NFTCostInvalidError();
        if (configuredTokens[_tokenContract].nftCost == 0) {
            // new token
            configuredTokenContracts.push(_tokenContract);
        }
        configuredTokens[_tokenContract] = _tokenData;

        emit TokenConfigured(_tokenContract, _tokenData);
    }
```

## [NC-06] : block.timestamp can be manipulated 


`block.timestamp` can be manipulated by miners to a small extent, so relying on it for precise timing might be risky.
Remediation:
Use `block.timestamp` only where a slight inaccuracy is acceptable, such as for longer intervals.


# Quality Assurance Report

The report consists of low-severity risk issues, governance risk issues arising from centralization vectors and a few non-critical issues in the end. The governance risks section describes scenarios in which the actors could misuse their powers.

 - [Low-severity issues](#low-severity-issues)
 - [Governance risks](#governance-risks)
   - [Roles/Actors in the system](#rolesactors-in-the-system)
   - [Admin powers](#admin-powers)
   - [Config Storage contract powers](#config-storage-contract-powers)
   - [Price feed role powers](#price-feed-role-powers)
 - [Non-Critical issues](#non-critical-issues)

## Low-severity issues

| ID     | Low-severity issues                                                                                                                       |
|--------|-------------------------------------------------------------------------------------------------------------------------------------------|
| [L-01] | Missing setter function to change config storage contract in LockManager                                                                  |
| [L-02] | Missing minLockDuration validations in function configureLockdrop()                                                                       |
| [L-03] | Configured token decimals are not validated against token's decimals() function                                                           |
| [L-04] | Missing notPaused() modifier on functions proposeUSDPrice(), approveUSDPrice(), disapproveUSDPrice()                                      |
| [L-05] | Incorrect use of only one price for all token contracts                                                                                   |
| [L-06] | Prices can be proposed for inactive tokens                                                                                                |
| [L-07] | Function approveUSDPrice() does not validate if price feed approves prices for the same token contract                                    |
| [L-08] | Incorrect validation in function approveUSDPrice() allows price feed roles to both approve and disapprove for a proposal Link to instance |
| [L-09] | Missing minLockDuration check in function setLockDuration()                                                                               |
| [L-10] | Function getLockedWeightedValue() does not account for duration in schibbles calculation                                                  |
| [L-11] | Negative rebase can tap into other user's tokens when a user unlocks                                                                      |
| [L-12] | Users can force receive schnibbles by making 0 value lock() or lockOnBehalfOf() calls                                                                         |
| [L-13] | Disallow users from extending their lock durations for inactive tokens                                                                    |

### [L-01] Missing setter function to change config storage contract in LockManager

[Link to instance](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L61)

Currently it is possible to reconfigure the BaseBlastManager state (inherited by LockManager) by using the [configUpdated()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L85) function. But it is not possible to change the config storage contract using the __BaseConfigStorage_setConfigStorage() internal function existing in the BaseBlastManager.

We can see that on Line 62 of the constructor, the config storage contract is set by calling the __BaseConfigStorage_setConfigStorage() internal function. But following that if the config storage contract needs to be changed, it is not possible to do so since the internal function has not been exposed in the LockManager contract.

It's unlikely the team might want to change the config contract itself but it would be good to leave the option open in case it ever needs to be changed.
```solidity
File: LockManager.sol
60:     constructor(address _configStorage) {
61:        
62:         __BaseConfigStorage_setConfigStorage(_configStorage);
63:         _reconfigure();
64:     }

File: LockManager.sol
87:     function configUpdated() external override onlyConfigStorage {
88:         _reconfigure();
89:     }
```

### [L-02] Missing minLockDuration validations in function configureLockdrop()

[Link to instance](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L98)

First validation:
The function should ensure `_lockdropData.end - _lockdropData.start >= _lockdropData.minLockDuration`. This ensures the lockdrop event is longer than the minimum lock times user need to lock their tokens for.

Second validation:
The function should ensure `_lockdropData.minLockDuration < configStorage.getUint(StorageKey.MaxLockDuration)`. This ensure function setLockDuration() does not revert [due to the check here](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L246C25-L246C74) and lock() as well due to the check [here](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L358).
```solidity
File: LockManager.sol
099:     function configureLockdrop(
100:         Lockdrop calldata _lockdropData
101:     ) external onlyAdmin {
102:         if (_lockdropData.end < block.timestamp)
103:             revert LockdropEndedError(
104:                 _lockdropData.end,
105:                 uint32(block.timestamp)
106:             ); // , "LockManager: End date is in the past");
107:         if (_lockdropData.start >= _lockdropData.end)
108:             revert LockdropInvalidError();
109:         
111:         lockdrop = _lockdropData;
112: 
113:         emit LockDropConfigured(_lockdropData);
114:     }
```

### [L-03] Configured token decimals are not validated against token's decimals() function

[Link to instance](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L115)

The function does not validate `_tokenData.decimals` for the `_tokenContract` against the ERC20 decimals() function of the token. This validation should be put in place to not only avoid human errors but also to avoid manipulation of schnibbles calculation (see second point in [Admin powers](#admin-powers) section for specific scenario on this).
```solidity
File: LockManager.sol
119:     function configureToken(
120:         address _tokenContract,
121:         ConfiguredToken memory _tokenData
122:     ) external onlyAdmin {
123:         if (_tokenData.nftCost == 0) revert NFTCostInvalidError();
124:         if (configuredTokens[_tokenContract].nftCost == 0) {
125:             // new token
126:             
127:             configuredTokenContracts.push(_tokenContract);
128:         }
129:         
131:         configuredTokens[_tokenContract] = _tokenData;
132: 
133:         emit TokenConfigured(_tokenContract, _tokenData);
134:     }
```

### [L-04] Missing notPaused() modifier on functions proposeUSDPrice(), approveUSDPrice(), disapproveUSDPrice()

Functions [proposeUSDPrice()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L142), [approveUSDPrice()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L177), [disapproveUSDPrice()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L210) are missing the notPaused() modifier on them. This allows the price feed role to update the prices of tokens even when locks and unlocks are paused. There does not seem to be a risk associated with this but it would be better to avoid changes in token prices while the protocol is paused. This does not cause any harm as well since once the protocol wants to unpause, the team can make a pause + update prices batched call to ensure users do not profit or are affected from the previous stale price. 

### [L-05] Incorrect use of only one price for all token contracts

[Link to instance](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L142)

The function proposeUSDPrice() incorrectly uses only one price for all token contracts. This is incorrect since token contracts e.g. WETH and USDB have different USD prices. This can cause prices to be set incorrectly for token contracts and thus weighted value of the locked tokens during schnibble calculation ends up being incorrect. 

The issue is being marked as low since although the functionality of the protocol is impacted, the root cause arises from the assumption that price feed roles are not aware that all token contracts share the same price and pass in incorrect parameters. 

**Solution: Either take in a single token contract as the input parameter or multiple prices i.e. an array as done with the _contracts parameter currently.**
```solidity
File: LockManager.sol
146:     function proposeUSDPrice(
147:         uint256 _price,
148:         address[] calldata _contracts
149:     )
```

### [L-06] Prices can be proposed for inactive tokens

[Link to instance](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L142)

The function proposeUSDPrice() does not check if a token contract is inactive. This allows the price feed role to update the prices for tokens that have been marked as inactive by the admin.

Solution: In [_execUSDPriceUpdate()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L506), consider checking inactive tokens and skip their price updates.  
```solidity
File: LockManager.sol
144:     function proposeUSDPrice(
145:         uint256 _price,
146:         address[] calldata _contracts
147:     )
148:         external
149:         onlyOneOfRoles(
150:             [
151:                 Role.PriceFeed_1,
152:                 Role.PriceFeed_2,
153:                 Role.PriceFeed_3,
154:                 Role.PriceFeed_4,
155:                 Role.PriceFeed_5
156:             ]
157:         )
158:     {   
159:        
160:         if (usdUpdateProposal.proposer != address(0))
161:             revert ProposalInProgressError();
162:         if (_contracts.length == 0) revert ProposalInvalidContractsError();
163: 
164:         delete usdUpdateProposal;
165: 
166:         // Approvals will use this because when the struct is deleted the approvals remain
167:         ++_usdProposalId;
168: 
169:         usdUpdateProposal.proposedDate = uint32(block.timestamp);
170:         usdUpdateProposal.proposer = msg.sender;
171:         usdUpdateProposal.proposedPrice = _price;
172:         usdUpdateProposal.contracts = _contracts;
173:         usdUpdateProposal.approvals[msg.sender] = _usdProposalId; 
174:         usdUpdateProposal.approvalsCount++;
175: 
176:         emit ProposedUSDPrice(msg.sender, _price);
177:     }
```

### [L-07] Function approveUSDPrice() does not validate if price feed approves prices for the same token contract

[Link to instance](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L177)

The function approveUSDPrice() does not check if price feed roles are approving the price of the correct token contract (i.e. usdUpdateProposal.contracts). Due to this, if two token contracts have the same price (e.g. ETH and WETH), it does not check if the price being approved is actually for ETH or WETH i.e. the respective token contract price was proposed for. The same issue exists in disapproveUSDPrice().
```solidity
File: LockManager.sol
181:     function approveUSDPrice(
182:         uint256 _price
183:     )
184:         external
185:         onlyOneOfRoles(
186:             [
187:                 Role.PriceFeed_1,
188:                 Role.PriceFeed_2,
189:                 Role.PriceFeed_3,
190:                 Role.PriceFeed_4,
191:                 Role.PriceFeed_5
192:             ]
193:         )
194:     {
195:         if (usdUpdateProposal.proposer == address(0)) revert NoProposalError();
196:         
197:         if (usdUpdateProposal.proposer == msg.sender)
198:             revert ProposerCannotApproveError();
199:         if (usdUpdateProposal.approvals[msg.sender] == _usdProposalId)
200:             revert ProposalAlreadyApprovedError();
201:         if (usdUpdateProposal.proposedPrice != _price)
202:             revert ProposalPriceNotMatchedError();
203: 
204:         usdUpdateProposal.approvals[msg.sender] = _usdProposalId;
205:         usdUpdateProposal.approvalsCount++;
206: 
207:         if (usdUpdateProposal.approvalsCount >= APPROVE_THRESHOLD) {
208:             _execUSDPriceUpdate();
209:         }
210: 
211:         emit ApprovedUSDPrice(msg.sender);
212:     }
```

### [L-08] Incorrect validation in function approveUSDPrice() allows price feed roles to both approve and disapprove for a proposal

[Link to instance](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L225C1-L228C54)

Unlike the [disapproveUSDPrice()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L210) function which validates both the disapproval and approval mapping with [these checks](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L225C1-L228C54), the [approveUSDPrice()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L177) function does not check the disapproval mapping. 

This allows price feed roles to first call [disapproveUSDPrice()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L210), which will pass since there has neither been an approval or disapproval from their side on the proposal yet. The function would set the disapproval mapping from price feed to the current `_usdProposalId`. The price feed role can then call [approveUSDPrice()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L177) and pass [this check](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L194) since it only checks for approval mapping and not the disapproval mapping as well.

Due to this, price feed roles can both approve and disapprove for a proposal, thereby double counting their votes.

Solution: Add the below check to the approveUSDPrice() function.

```solidity
File: LockManager.sol
233:         if (usdUpdateProposal.disapprovals[msg.sender] == _usdProposalId)
234:             revert ProposalAlreadyDisapprovedError();
```

### [L-09] Missing minLockDuration check in function setLockDuration()

[Link to instance](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L245)

As we can see below, function setLockDuration() only checks for MaxLockDuration and not minLockDuration. This is problematic since users can set _duration to be lower than the minimum. This has no problem outside the lockdrop event since the minimum lock duration is only specific to lockdrop events. In the lockdrop though, the calls would revert for users having lock times less than the minimum due to this [check here](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L357). This is not a big issue since users can just extend their lock durations to the minLockDuration. Enforcing the minimum check would be helpful though.

```solidity
File: LockManager.sol
249:     function setLockDuration(uint256 _duration) external notPaused {
250:         if (_duration > configStorage.getUint(StorageKey.MaxLockDuration))
251:             revert MaximumLockDurationError();
252: 
```

**Solution: Check if the _duration parameter passed by user is greater than equal to minLockDuration, only if lockdrop event has started i.e. block.timestamp is greater than start and less than end time of the lockdrop event. This would ensure users are not forced to extend their lock times to minLockDuration when there is no lockdrop occurring and users are locking tokens just for schnibbles earning.** 

### [L-10] Function getLockedWeightedValue() does not account for duration in schibbles calculation

[Link to instance](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L461)

This function is used by the AccountManager contract to calculate the schnibbles earned by the user for the tokens they locked. The issue is that the `lockedWeight` does not seem to consider the time duration the user has locked their tokens for. This means that the schnibbles the user currently receives is just based on the price and quantity they deposited. 

I'm not sure if this is the intended behaviour or not since the AccountManager contract [here](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/AccountManager.sol#L363C1-L370C6) calculates a bonus through the BonusManager which uses user's lock time - see [here](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/BonusManager.sol#L150). 

Raising this issue to bring the sponsor's attention i.e. whether weighted value should be time weighted or not since a user locking tokens for longer has no benefit for the schnibble calculation. 
```solidity
File: LockManager.sol
477:     function getLockedWeightedValue(
478:         address _player
479:     ) external view returns (uint256 _lockedWeightedValue) {
480:         uint256 lockedWeighted = 0;
481:         uint256 configuredTokensLength = configuredTokenContracts.length;
482:         for (uint256 i; i < configuredTokensLength; i++) {
483:             if (
484:                 lockedTokens[_player][configuredTokenContracts[i]].quantity >
485:                 0 &&
486:                 configuredTokens[configuredTokenContracts[i]].active
487:             ) {
488:                 // We are assuming all tokens have a maximum of 18 decimals and that USD Price is denoted in 1e18
489:                 
490:                 uint256 deltaDecimal = 10 **
491:                     (18 -
492:                         configuredTokens[configuredTokenContracts[i]].decimals);
493:                 
494:                 lockedWeighted +=
495:                     (deltaDecimal *
496:                         lockedTokens[_player][configuredTokenContracts[i]]
497:                             .quantity *
498:                         configuredTokens[configuredTokenContracts[i]]
499:                             .usdPrice) /
500:                     1e18;
501:             }
502:         }
503: 
504:         _lockedWeightedValue = lockedWeighted;
505:     }
```

### [L-11] Negative rebase can tap into other user's tokens when a user unlocks

[Link to instance](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L416)

If the configured token contract being used is a rebasing token that negatively rebases, the protocol can cause loss of funds to users. This is because when the overall balance of the contract decreases, the quantity being used for withdrawals on Line 424 is based on the time the the tokens were locked. Due to this, the user unlocking can receive more tokens than needed, causing loss to other users.

The blast docs [here](https://docs.blast.io/building/guides/eth-yield) mention that the rebasing for ETH, WETH and USDB is increasing and never decreases. But if future tokens that rebase negatively exist, this issue exists.
```solidity
File: LockManager.sol
409:     function unlock(
410:         address _tokenContract,
411:         uint256 _quantity
412:     ) external notPaused nonReentrant {
413:         LockedToken storage lockedToken = lockedTokens[msg.sender][
414:             _tokenContract
415:         ];
416:         if (lockedToken.quantity < _quantity)
417:             revert InsufficientLockAmountError();
418:         if (lockedToken.unlockTime > uint32(block.timestamp))
419:             revert TokenStillLockedError();
420: 
421:         // force harvest to make sure that they get the schnibbles that they are entitled to
422:         accountManager.forceHarvest(msg.sender);
423: 
424:         lockedToken.quantity -= _quantity;
425: 
426:         // send token
427:         if (_tokenContract == address(0)) {
428:             payable(msg.sender).transfer(_quantity);
429:         } else {
430:             IERC20 token = IERC20(_tokenContract);
431:             token.transfer(msg.sender, _quantity);
432:         }
433: 
434:         emit Unlocked(msg.sender, _tokenContract, _quantity);
435:     }
```

### [L-12] Users can force receive schnibbles by making 0 value lock() or lockOnBehalfOf() calls

[Link to instance](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L311)

As discussed in my only High-severity issue, it is possible for anyone to make 0 quantity lock() or lockOnBehalfOf() calls. This allows users or an attacker to force users to receive their schnibbles at any time without requiring to lock tokens or unlock them.

### [L-13] Disallow users from extending their lock durations for inactive tokens

[Link to instance](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L245)

Users should not be allowed to extend durations for their existing locks for tokens that have been deactivated by the admin. The users should instead be forced to leave the system with the token for safer side since tokens can be deactivated for any reason and should not exist in the LockManager for long once deactivated.

## Governance risks

### Roles/Actors in the system
 - Admin
 - Config storage contract (indirectly controlled by onlyOwner modifier)
 - 5 Price feed roles

### Admin powers
 - [configureLockdrop()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L98) - Allows admin to change the start, end and minimum duration for a lockdrop event. The minimum duration can be changed during an ongoing lockdrop event as well. The risk associated with this is that if the minLockDuration is increased, existing locks using the lock durations less than the increased minLockDuration will not be able to lock their tokens due to this [check here](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L357). This can force them to extend their lock times through [setLockDuration()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L245).
 - [configureToken()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L115) - Allows admin to activate/deactivate a token contract and set the nftCost in terms of the token for the lockdrop and the token decimals. The first risk associated is that the admin can set the nftCost to a lower or higher value to favour some users or themselves over the others. The second risk associated is that token decimals can be decreased or increased to provide higher or lower schnibbles to specific users on lock() and unlock() operations that forceHarvest() these schnibbles and use [getLockedWeightedValue()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L461) for the calculation. For example, admin can change decimals to 1e18 for a token with 6 decimals. Assuming the user deposited 1 token and usdPrice = 1e18, the weighted value = 1 * 1e6 * 1e18 / 1e18 = 1e6. We can see that instead of a delta decimal of 1e12, we use 1, thus causing loss to the users. The opposite can be done to increase the schnibbles to favour other users. The third risk associated is that the admin can activate and deactivate the token at any time, thus DOSing users from locking tokens. Lastly, the admin can push a lot of contracts to the configuredTokenContracts array to cause DOS to not only the price feed approvals/disapprovals but also lock and unlock mechanisms since they use getLockedWeightedValue() for schnibbles calculation and loop through the configured token contracts array.
 - [setUSDThresholds()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L129) - Allows admin to increase or decrease the approval and disapproval threshold. This could be used along with the price feed roles to approve and manipulate the price against other price feed roles' decision.

### Config Storage contract powers
 - [configUpdated()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L85) - This function is only callable by the config storage contract to reconfigure the state in the BaseBlastManager contract. The call from the config storage contract originates from functions having the onlyOwner() modifier though. Due to this, there is a possible centralization risk, where reconfigurations could occur at any time. 

### Price feed role powers
 - [proposeUSDPrice()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L142) - Either of the 5 price feed role addresses can propose any price for any token contract. This price can be manipulated and can be different from the actual price since there is no rate limit as well as validation mechanism onchain. This can affect the schnibbles users receive. The safety net is that this requires approval from 2 more price feed roles, which is still relatively easy if three price feed roles conjoin.
 - [approveUSDPrice()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L177) and [disapproveUSDPrice()](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L210) - Allows price feed roles to either approve or disapprove prices proposed by a price feed role. 

## Non-Critical issues

| ID     | Non-Critical issues                                                    |
|--------|------------------------------------------------------------------------|
| [N-01] | Snuggery manager variable is never used in LockManager                 |
| [N-02] | Consider removing redundant check from function approveUSDPrice()      |
| [N-03] | Consider removing auto approval mechanism on price proposal            |
| [N-04] | Consider removing redundant checks from function _execUSDPriceUpdate() |

### [N-01] Snuggery manager variable is never used in LockManager

[Link to instance](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L74)

```solidity
File: LockManager.sol
75:         snuggeryManager = ISnuggeryManager(
76:             configStorage.getAddress(StorageKey.SnuggeryManager)
77:         );
``` 

### [N-02] Consider removing redundant check from function approveUSDPrice()

[Link to instance](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L177)

The check on Line 196 is not necessary. This is because the check on Line 198 ensures that the proposer cannot approve the price one again (**Context:** The price feed's approval is counted during proposal itself - see [here](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L170C1-L171C44)).
```solidity
File: LockManager.sol
180:     function approveUSDPrice(
181:         uint256 _price
182:     )
183:         external
184:         onlyOneOfRoles(
185:             [
186:                 Role.PriceFeed_1,
187:                 Role.PriceFeed_2,
188:                 Role.PriceFeed_3,
189:                 Role.PriceFeed_4,
190:                 Role.PriceFeed_5
191:             ]
192:         )
193:     {
194:         if (usdUpdateProposal.proposer == address(0)) revert NoProposalError();
195:         
196:         if (usdUpdateProposal.proposer == msg.sender)
197:             revert ProposerCannotApproveError();
198:         if (usdUpdateProposal.approvals[msg.sender] == _usdProposalId)
199:             revert ProposalAlreadyApprovedError();
```

### [N-03] Consider removing auto approval mechanism on price proposal

[Link to instance](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L169)

Currently the proposer is denied from disapproving the price on their own proposal. The price feed who is the proposer should be allowed to perform disapprovals and approvals separately from their proposal. This would make the process more modular and separate instead of auto approving during proposal itself as seen [here](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L170C1-L171C44). 

Solution: This solution separates the proposing and approving/disapproving on the proposal, which allows the proposer to either approve or disapprove on their price proposed. They can disapprove in case the price proposed is incorrect. 

1. Remove these two lines from proposeUSDPrice() function:
```solidity
File: LockManager.sol
171:         usdUpdateProposal.approvals[msg.sender] = _usdProposalId; 
172:         usdUpdateProposal.approvalsCount++;
```

2. Remove this check from the approveUSDPrice() function:
```solidity
File: LockManager.sol
195:         if (usdUpdateProposal.proposer == msg.sender)
196:             revert ProposerCannotApproveError();
```

### [N-04] Consider removing redundant checks from function _execUSDPriceUpdate()

[Link to instance](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L508)

The first check can be removed since _execUSDPriceUpdate() is only executed if approval threshold is met as seen [here](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L203).

The second check can be removed since disapprovalsCount will always be less than the DISAPPROVE_THRESHOLD. Because if it was not, the proposal would've been deleted already - see [here](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/src/managers/LockManager.sol#L238).

```solidity
File: LockManager.sol
525:         if (
526:             usdUpdateProposal.approvalsCount >= APPROVE_THRESHOLD &&
527:             usdUpdateProposal.disapprovalsCount < DISAPPROVE_THRESHOLD
528:         ) {
```
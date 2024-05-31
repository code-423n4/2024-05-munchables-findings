# QA Report

## Summary

### Low Issues

Total **15 instances** over **6 issues**:

|ID|Issue|Instances|
|:--:|:---|:--:|
| [[L-01]](#l-01-array-is-pushed-but-not-poped) | Array is `push()`ed but not `pop()`ed | 1 |
| [[L-02]](#l-02-state-variables-not-limited-to-reasonable-values) | State variables not limited to reasonable values | 2 |
| [[L-03]](#l-03-constructor--initialization-function-lacks-parameter-validation) | Constructor / initialization function lacks parameter validation | 1 |
| [[L-04]](#l-04-functions-calling-contractsaddresses-with-transfer-hooks-should-be-protected-by-reentrancy-guard) | Functions calling contracts/addresses with transfer hooks should be protected by reentrancy guard | 1 |
| [[L-05]](#l-05-critical-functions-should-be-controlled-by-time-locks) | Critical functions should be controlled by time locks | 1 |
| [[L-06]](#l-06-code-does-not-follow-the-best-practice-of-check-effects-interaction) | Code does not follow the best practice of check-effects-interaction | 9 |

### Non Critical Issues

Total **88 instances** over **29 issues**:

|ID|Issue|Instances|
|:--:|:---|:--:|
| [[N-01]](#n-01-import-declarations-should-import-specific-identifiers-rather-than-the-whole-file) | Import declarations should import specific identifiers, rather than the whole file | 9 |
| [[N-02]](#n-02-visibility-of-state-variables-is-not-explicitly-defined) | Visibility of state variables is not explicitly defined | 4 |
| [[N-03]](#n-03-names-of-privateinternal-state-variables-should-be-prefixed-with-an-underscore) | Names of `private`/`internal` state variables should be prefixed with an underscore | 4 |
| [[N-04]](#n-04-the-nonreentrant-modifier-should-occur-before-all-other-modifiers) | The `nonReentrant` `modifier` should occur before all other modifiers | 3 |
| [[N-05]](#n-05-use-of-override-is-unnecessary) | Use of `override` is unnecessary | 1 |
| [[N-06]](#n-06-consider-providing-a-ranged-getter-for-array-state-variables) | Consider providing a ranged getter for array state variables | 1 |
| [[N-07]](#n-07-consider-splitting-complex-checks-into-multiple-steps) | Consider splitting complex checks into multiple steps | 3 |
| [[N-08]](#n-08-consider-adding-a-blockdeny-list) | Consider adding a block/deny-list | 1 |
| [[N-09]](#n-09-upper_case-names-should-be-reserved-for-constantimmutable-variables) | UPPER_CASE names should be reserved for `constant`/`immutable` variables | 2 |
| [[N-10]](#n-10-contract-timekeeping-will-break-earlier-than-the-ethereum-network-itself-will-stop-working) | Contract timekeeping will break earlier than the Ethereum network itself will stop working | 8 |
| [[N-11]](#n-11-consider-emitting-an-event-at-the-end-of-the-constructor) | Consider emitting an event at the end of the constructor | 1 |
| [[N-12]](#n-12-events-are-emitted-without-the-sender-information) | Events are emitted without the sender information | 1 |
| [[N-13]](#n-13-imports-could-be-organized-more-systematically) | Imports could be organized more systematically | 2 |
| [[N-14]](#n-14-functions-should-be-named-in-mixedcase-style) | Functions should be named in mixedCase style | 4 |
| [[N-15]](#n-15-named-imports-of-parent-contracts-are-missing) | Named imports of parent contracts are missing | 3 |
| [[N-16]](#n-16-constants-should-be-put-on-the-left-side-of-comparisons) | Constants should be put on the left side of comparisons | 16 |
| [[N-17]](#n-17-high-cyclomatic-complexity) | High cyclomatic complexity | 1 |
| [[N-18]](#n-18-unnecessary-casting) | Unnecessary casting | 1 |
| [[N-19]](#n-19-events-that-mark-critical-parameter-changes-should-contain-both-the-old-and-the-new-value) | Events that mark critical parameter changes should contain both the old and the new value | 1 |
| [[N-20]](#n-20-multiple-mappings-with-same-keys-can-be-combined-into-a-single-struct-mapping-for-readability) | Multiple mappings with same keys can be combined into a single struct mapping for readability | 3 |
| [[N-21]](#n-21-duplicated-requirerevert-checks-should-be-refactored) | Duplicated `require()`/`revert()` checks should be refactored | 4 |
| [[N-22]](#n-22-consider-adding-emergency-stop-functionality) | Consider adding emergency-stop functionality | 1 |
| [[N-23]](#n-23-missing-checks-for-uint-state-variable-assignments) | Missing checks for uint state variable assignments | 2 |
| [[N-24]](#n-24-use-the-modern-upgradeable-contract-paradigm) | Use the Modern Upgradeable Contract Paradigm | 1 |
| [[N-25]](#n-25-large-or-complicated-code-bases-should-implement-invariant-tests) | Large or complicated code bases should implement invariant tests | 1 |
| [[N-26]](#n-26-the-default-value-is-manually-set-when-it-is-declared) | The default value is manually set when it is declared | 1 |
| [[N-27]](#n-27-contracts-should-have-all-publicexternal-functions-exposed-by-interfaces) | Contracts should have all `public`/`external` functions exposed by `interface`s | 1 |
| [[N-28]](#n-28-top-level-declarations-should-be-separated-by-at-least-two-lines) | Top-level declarations should be separated by at least two lines | 7 |
| [[N-29]](#n-29-consider-adding-formal-verification-proofs) | Consider adding formal verification proofs | 1 |

## Low Issues

### [L-01] Array is `push()`ed but not `pop()`ed

Array entries are added but are never removed. Consider whether this should be the case, or whether there should be a maximum, or whether old entries should be removed. Cases where there are specific potential problems will be flagged separately under a different issue.

There is 1 instance:

- *LockManager.sol* ( [122-122](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L122-L122) ):

```solidity
122:             configuredTokenContracts.push(_tokenContract);
```

### [L-02] State variables not limited to reasonable values

Consider adding appropriate minimum/maximum value checks to ensure that the following state variables can never be used to excessively harm users, including via griefing.

There are 2 instances:

- *LockManager.sol* ( [135-135](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L135-L135), [136-136](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L136-L136) ):

```solidity
135:         APPROVE_THRESHOLD = _approve;

136:         DISAPPROVE_THRESHOLD = _disapprove;
```

### [L-03] Constructor / initialization function lacks parameter validation

Constructors and initialization functions play a critical role in contracts by setting important initial states when the contract is first deployed before the system starts. The parameters passed to the constructor and initialization functions directly affect the behavior of the contract / protocol. If incorrect parameters are provided, the system may fail to run, behave abnormally, be unstable, or lack security. Therefore, it's crucial to carefully check each parameter in the constructor and initialization functions. If an exception is found, the transaction should be rolled back.

There is 1 instance:

- *LockManager.sol* ( [60-60](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L60-L60) ):

```solidity
/// @audit `_configStorage`
60:     constructor(address _configStorage) {
```

### [L-04] Functions calling contracts/addresses with transfer hooks should be protected by reentrancy guard

Even if the function follows the best practice of check-effects-interaction, not using a reentrancy guard when there may be transfer hooks opens the users of this protocol up to [read-only reentrancy vulnerability](https://chainsecurity.com/curve-lp-oracle-manipulation-post-mortem/) with no way to protect them except by block-listing the entire protocol.

There is 1 instance:

- *LockManager.sol* ( [376-376](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L376-L376) ):

```solidity
376:             token.transferFrom(_tokenOwner, address(this), _quantity);
```

### [L-05] Critical functions should be controlled by time locks

It is a good practice to give time for users to react and adjust to critical changes. A timelock provides more guarantees and reduces the level of trust required, thus decreasing risk for users. It also indicates that the project is legitimate (less risk of a malicious owner making a sandwich attack on a user).

There is 1 instance:

- *LockManager.sol* ( [115-118](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L115-L118) ):

```solidity
115:     function configureToken(
116:         address _tokenContract,
117:         ConfiguredToken memory _tokenData
118:     ) external onlyAdmin {
```

### [L-06] Code does not follow the best practice of check-effects-interaction

Code should follow the best-practice of [check-effects-interaction](https://blockchain-academy.hs-mittweida.de/courses/solidity-coding-beginners-to-intermediate/lessons/solidity-11-coding-patterns/topic/checks-effects-interactions/), where state variables are updated before any external calls are made. Doing so prevents a large class of reentrancy bugs.

<details>
<summary>There are 9 instances (click to show):</summary>

- *LockManager.sol* ( [350-350](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L350-L350), [363-363](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L363-L363), [364-364](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L364-L364), [379-379](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L379-L379), [380-380](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L380-L380), [381-381](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L381-L381), [382-384](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L382-L384), [387-387](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L387-L387), [416-416](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L416-L416) ):

```solidity
/// @audit `getPlayer()` is called on line 320
350:             _lockDuration = lockdrop.minLockDuration;

/// @audit `getPlayer()` is called on line 320
363:                 remainder = quantity % configuredToken.nftCost;

/// @audit `getPlayer()` is called on line 320
364:                 numberNFTs = (quantity - remainder) / configuredToken.nftCost;

/// @audit `getPlayer()` is called on line 320
379:         lockedToken.remainder = remainder;

/// @audit `getPlayer()` is called on line 320
380:         lockedToken.quantity += _quantity;

/// @audit `getPlayer()` is called on line 320
381:         lockedToken.lastLockTime = uint32(block.timestamp);

/// @audit `getPlayer()` is called on line 320
382:         lockedToken.unlockTime =
383:             uint32(block.timestamp) +
384:             uint32(_lockDuration);

/// @audit `getPlayer()` is called on line 320
387:         playerSettings[_lockRecipient].lockDuration = _lockDuration;

/// @audit `forceHarvest()` is called on line 414
416:         lockedToken.quantity -= _quantity;
```

</details>

## Non Critical Issues

### [N-01] Import declarations should import specific identifiers, rather than the whole file

Using import declarations of the form `import {<identifier_name>} from "some/file.sol"` avoids polluting the symbol namespace making flattened files smaller, and speeds up compilation (but does not save any gas).

There are 9 instances:

- *LockManager.sol* ( [4](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L4), [5](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L5), [6](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L6), [7](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L7), [8](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L8), [9](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L9), [10](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L10), [11](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L11), [12](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L12) ):

```solidity
4: import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

5: import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

6: import "../interfaces/ILockManager.sol";

7: import "../interfaces/IConfigStorage.sol";

8: import "../interfaces/IAccountManager.sol";

9: import "../interfaces/IMigrationManager.sol";

10: import "./BaseBlastManager.sol";

11: import "../interfaces/ISnuggeryManager.sol";

12: import "../interfaces/INFTOverlord.sol";
```

### [N-02] Visibility of state variables is not explicitly defined

To avoid misunderstandings and unexpected state accesses, it is recommended to explicitly define the visibility of each state variable.

There are 4 instances:

- *LockManager.sol* ( [16-16](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L16-L16), [18-18](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L18-L18), [26-26](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L26-L26), [30-30](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L30-L30) ):

```solidity
16:     uint8 APPROVE_THRESHOLD = 3;

18:     uint8 DISAPPROVE_THRESHOLD = 3;

26:     mapping(address => PlayerSettings) playerSettings;

30:     USDUpdateProposal usdUpdateProposal;
```

### [N-03] Names of `private`/`internal` state variables should be prefixed with an underscore

It is recommended by the [Solidity Style Guide](https://docs.soliditylang.org/en/v0.8.20/style-guide.html#underscore-prefix-for-non-external-functions-and-variables).

There are 4 instances:

- *LockManager.sol* ( [16-16](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L16-L16), [18-18](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L18-L18), [26-26](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L26-L26), [30-30](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L30-L30) ):

```solidity
16:     uint8 APPROVE_THRESHOLD = 3;

18:     uint8 DISAPPROVE_THRESHOLD = 3;

26:     mapping(address => PlayerSettings) playerSettings;

30:     USDUpdateProposal usdUpdateProposal;
```

### [N-04] The `nonReentrant` `modifier` should occur before all other modifiers

This is a best-practice to protect against reentrancy in other modifiers

<details>
<summary>There are 3 instances (click to show):</summary>

- *LockManager.sol* ( [275-285](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L275-L285), [297-306](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L297-L306), [401-404](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L401-L404) ):

```solidity
275:     function lockOnBehalf(
276:         address _tokenContract,
277:         uint256 _quantity,
278:         address _onBehalfOf
279:     )
280:         external
281:         payable
282:         notPaused
283:         onlyActiveToken(_tokenContract)
284:         onlyConfiguredToken(_tokenContract)
285:         nonReentrant

297:     function lock(
298:         address _tokenContract,
299:         uint256 _quantity
300:     )
301:         external
302:         payable
303:         notPaused
304:         onlyActiveToken(_tokenContract)
305:         onlyConfiguredToken(_tokenContract)
306:         nonReentrant

401:     function unlock(
402:         address _tokenContract,
403:         uint256 _quantity
404:     ) external notPaused nonReentrant {
```

</details>

### [N-05] Use of `override` is unnecessary

[Starting from Solidity 0.8.8](https://docs.soliditylang.org/en/v0.8.20/contracts.html#function-overriding), the override keyword is not required when overriding an interface function, except for the case where the function is defined in multiple bases.

There is 1 instance:

- *LockManager.sol* ( [85-85](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L85-L85) ):

```solidity
85:     function configUpdated() external override onlyConfigStorage {
```

### [N-06] Consider providing a ranged getter for array state variables

While the compiler automatically provides a getter for accessing single elements within a public state variable array, it doesn't provide a way to fetch the whole array, or subsets thereof. Consider adding a function to allow the fetching of slices of the array, especially if the contract doesn't already have multicall functionality.

There is 1 instance:

- *LockManager.sol* ( [22-22](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L22-L22) ):

```solidity
22:     address[] public configuredTokenContracts;
```

### [N-07] Consider splitting complex checks into multiple steps

Assign the expression's parts to intermediate local variables, and check against those instead.

There are 3 instances:

- *LockManager.sol* ( [353-354](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L353-L354), [357-359](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L357-L359), [508-509](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L508-L509) ):

```solidity
353:             lockdrop.start <= uint32(block.timestamp) &&
354:             lockdrop.end >= uint32(block.timestamp)

357:                 _lockDuration < lockdrop.minLockDuration ||
358:                 _lockDuration >
359:                 uint32(configStorage.getUint(StorageKey.MaxLockDuration))

508:             usdUpdateProposal.approvalsCount >= APPROVE_THRESHOLD &&
509:             usdUpdateProposal.disapprovalsCount < DISAPPROVE_THRESHOLD
```

### [N-08] Consider adding a block/deny-list

Doing so will significantly increase centralization, but will help to prevent hackers from using stolen tokens

There is 1 instance:

- *LockManager.sol* ( [14](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L14) ):

```solidity
14: contract LockManager is BaseBlastManager, ILockManager, ReentrancyGuard {
```

### [N-09] UPPER_CASE names should be reserved for `constant`/`immutable` variables

If the variable needs to be different based on which class it comes from, a `view`/`pure` _function_ should be used instead (e.g. like [this](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/76eee35971c2541585e05cbf258510dda7b2fbc6/contracts/token/ERC20/extensions/draft-IERC20Permit.sol#L59)).

There are 2 instances:

- *LockManager.sol* ( [16](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L16), [18](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L18) ):

```solidity
16:     uint8 APPROVE_THRESHOLD = 3;

18:     uint8 DISAPPROVE_THRESHOLD = 3;
```

### [N-10] Contract timekeeping will break earlier than the Ethereum network itself will stop working

When a timestamp is downcast from `uint256` to `uint32`, the value will wrap in the year 2106, and the contracts will break. Other downcasts will have different endpoints. Consider whether your contract is intended to live past the size of the type being used.

There are 8 instances:

- *LockManager.sol* ( [104-104](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L104-L104), [166-166](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L166-L166), [257-257](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L257-L257), [353-353](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L353-L353), [354-354](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L354-L354), [381-381](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L381-L381), [383-383](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L383-L383), [410-410](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L410-L410) ):

```solidity
104:                 uint32(block.timestamp)

166:         usdUpdateProposal.proposedDate = uint32(block.timestamp);

257:                     uint32(block.timestamp) + uint32(_duration) <

353:             lockdrop.start <= uint32(block.timestamp) &&

354:             lockdrop.end >= uint32(block.timestamp)

381:         lockedToken.lastLockTime = uint32(block.timestamp);

383:             uint32(block.timestamp) +

410:         if (lockedToken.unlockTime > uint32(block.timestamp))
```

### [N-11] Consider emitting an event at the end of the constructor

This will allow users to easily exactly pinpoint when and by whom a contract was constructed.

There is 1 instance:

- *LockManager.sol* ( [60-60](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L60-L60) ):

```solidity
60:     constructor(address _configStorage) {
```

### [N-12] Events are emitted without the sender information

When an action is triggered based on a user's action, not being able to filter based on who triggered the action makes event processing a lot more cumbersome. Including the `msg.sender` the events of these types of action will make events much more useful to end users, especially when `msg.sender` is not `tx.origin`.

There is 1 instance:

- *LockManager.sol* ( [389-397](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L389-L397) ):

```solidity
389:         emit Locked(
390:             _lockRecipient,
391:             _tokenOwner,
392:             _tokenContract,
393:             _quantity,
394:             remainder,
395:             numberNFTs,
396:             _lockDuration
397:         );
```

### [N-13] Imports could be organized more systematically

The contract's interface should be imported first, followed by each of the interfaces it uses, followed by all other files. The examples below do not follow this layout.

There are 2 instances:

- *LockManager.sol* ( [6-6](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L6-L6), [11-11](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L11-L11) ):

```solidity
/// @audit Out of order with the prev import️ ⬆
6: import "../interfaces/ILockManager.sol";

/// @audit Out of order with the prev import️ ⬆
11: import "../interfaces/ISnuggeryManager.sol";
```

### [N-14] Functions should be named in mixedCase style

As the [Solidity Style Guide](https://docs.soliditylang.org/en/v0.8.21/style-guide.html#function-names) suggests: functions should be named in mixedCase style.

There are 4 instances:

- *LockManager.sol* ( [142](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L142), [177](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L177), [210](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L210), [506](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L506) ):

```solidity
142:     function proposeUSDPrice(

177:     function approveUSDPrice(

210:     function disapproveUSDPrice(

506:     function _execUSDPriceUpdate() internal {
```

### [N-15] Named imports of parent contracts are missing

Consider adding named imports for all parent contracts.

There are 3 instances:

- *LockManager.sol* ( [14-14](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L14-L14) ):

```solidity
/// @audit BaseBlastManager
/// @audit ILockManager
/// @audit ReentrancyGuard
14: contract LockManager is BaseBlastManager, ILockManager, ReentrancyGuard {
```

### [N-16] Constants should be put on the left side of comparisons

Putting constants on the left side of comparison statements is a best practice known as [Yoda conditions](https://en.wikipedia.org/wiki/Yoda_conditions).
Although solidity's static typing system prevents accidental assignments within conditionals, adopting this practice can improve code readability and consistency, especially when working across multiple languages.

<details>
<summary>There are 16 instances (click to show):</summary>

- *LockManager.sol* ( [48](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L48), [119](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L119), [120](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L120), [133](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L133), [157](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L157), [159](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L159), [191](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L191), [224](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L224), [289](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L289), [322](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L322), [324](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L324), [327](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L327), [349](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L349), [374](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L374), [419](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L419), [514](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L514) ):

```solidity
/// @audit put `0` on the left
48:         if (configuredTokens[_tokenContract].nftCost == 0)

/// @audit put `0` on the left
119:         if (_tokenData.nftCost == 0) revert NFTCostInvalidError();

/// @audit put `0` on the left
120:         if (configuredTokens[_tokenContract].nftCost == 0) {

/// @audit put `address(0)` on the left
133:         if (usdUpdateProposal.proposer != address(0))

/// @audit put `address(0)` on the left
157:         if (usdUpdateProposal.proposer != address(0))

/// @audit put `0` on the left
159:         if (_contracts.length == 0) revert ProposalInvalidContractsError();

/// @audit put `address(0)` on the left
191:         if (usdUpdateProposal.proposer == address(0)) revert NoProposalError();

/// @audit put `address(0)` on the left
224:         if (usdUpdateProposal.proposer == address(0)) revert NoProposalError();

/// @audit put `address(0)` on the left
289:         if (_onBehalfOf != address(0)) {

/// @audit put `0` on the left
322:         if (_player.registrationDate == 0) revert AccountNotRegisteredError();

/// @audit put `address(0)` on the left
324:         if (_tokenContract == address(0)) {

/// @audit put `0` on the left
327:             if (msg.value != 0) revert InvalidMessageValueError();

/// @audit put `0` on the left
349:         if (_lockDuration == 0) {

/// @audit put `address(0)` on the left
374:         if (_tokenContract != address(0)) {

/// @audit put `address(0)` on the left
419:         if (_tokenContract == address(0)) {

/// @audit put `0` on the left
514:                 if (configuredTokens[tokenContract].nftCost != 0) {
```

</details>

### [N-17] High cyclomatic complexity

Consider breaking down these blocks into more manageable units, by splitting things into utility functions, by reducing nesting, and by using early returns.

<details>
<summary>There is 1 instance (click to show):</summary>

- *LockManager.sol* ( [311-398](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L311-L398) ):

```solidity
311:     function _lock(
312:         address _tokenContract,
313:         uint256 _quantity,
314:         address _tokenOwner,
315:         address _lockRecipient
316:     ) private {
317:         (
318:             address _mainAccount,
319:             MunchablesCommonLib.Player memory _player
320:         ) = accountManager.getPlayer(_lockRecipient);
321:         if (_mainAccount != _lockRecipient) revert SubAccountCannotLockError();
322:         if (_player.registrationDate == 0) revert AccountNotRegisteredError();
323:         // check approvals and value of tx matches
324:         if (_tokenContract == address(0)) {
325:             if (msg.value != _quantity) revert ETHValueIncorrectError();
326:         } else {
327:             if (msg.value != 0) revert InvalidMessageValueError();
328:             IERC20 token = IERC20(_tokenContract);
329:             uint256 allowance = token.allowance(_tokenOwner, address(this));
330:             if (allowance < _quantity) revert InsufficientAllowanceError();
331:         }
332: 
333:         LockedToken storage lockedToken = lockedTokens[_lockRecipient][
334:             _tokenContract
335:         ];
...... OMITTED ......
374:         if (_tokenContract != address(0)) {
375:             IERC20 token = IERC20(_tokenContract);
376:             token.transferFrom(_tokenOwner, address(this), _quantity);
377:         }
378: 
379:         lockedToken.remainder = remainder;
380:         lockedToken.quantity += _quantity;
381:         lockedToken.lastLockTime = uint32(block.timestamp);
382:         lockedToken.unlockTime =
383:             uint32(block.timestamp) +
384:             uint32(_lockDuration);
385: 
386:         // set their lock duration in playerSettings
387:         playerSettings[_lockRecipient].lockDuration = _lockDuration;
388: 
389:         emit Locked(
390:             _lockRecipient,
391:             _tokenOwner,
392:             _tokenContract,
393:             _quantity,
394:             remainder,
395:             numberNFTs,
396:             _lockDuration
397:         );
398:     }
```

</details>

### [N-18] Unnecessary casting

Unnecessary castings can be removed.

There is 1 instance:

- *LockManager.sol* ( [384-384](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L384-L384) ):

```solidity
/// @audit uint32(_lockDuration)
384:             uint32(_lockDuration);
```

### [N-19] Events that mark critical parameter changes should contain both the old and the new value

This should especially be done if the new value is not required to be different from the old value.

There is 1 instance:

- *LockManager.sol* ( [518-521](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L518-L521) ):

```solidity
518:                     emit USDPriceUpdated(
519:                         tokenContract,
520:                         usdUpdateProposal.proposedPrice
521:                     );
```

### [N-20] Multiple mappings with same keys can be combined into a single struct mapping for readability

Well-organized data structures make code reviews easier, which may lead to fewer bugs. Consider combining related mappings into mappings to structs, so it's clear what data is related.

There are 3 instances:

- *LockManager.sol* ( [20-20](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L20-L20), [24-24](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L24-L24), [26-26](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L26-L26) ):

```solidity
20:     mapping(address => ConfiguredToken) public configuredTokens;

24:     mapping(address => mapping(address => LockedToken)) public lockedTokens;

26:     mapping(address => PlayerSettings) playerSettings;
```

### [N-21] Duplicated `require()`/`revert()` checks should be refactored

Refactoring duplicate `require()`/`revert()` checks into a modifier or function can make the code more concise, readable and maintainable, and less likely to make errors or omissions when modifying the `require()` or `revert()`.

There are 4 instances:

- *LockManager.sol* ( [134-134](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L134-L134), [191-191](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L191-L191), [195-195](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L195-L195), [197-197](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L197-L197) ):

```solidity
/// @audit Duplicated on line 158
134:             revert ProposalInProgressError();

/// @audit Duplicated on line 224
191:         if (usdUpdateProposal.proposer == address(0)) revert NoProposalError();

/// @audit Duplicated on line 226
195:             revert ProposalAlreadyApprovedError();

/// @audit Duplicated on line 230
197:             revert ProposalPriceNotMatchedError();
```

### [N-22] Consider adding emergency-stop functionality

Adding a way to quickly halt protocol functionality in an emergency, rather than having to pause individual contracts one-by-one, will make in-progress hack mitigation faster and much less stressful.

There is 1 instance:

- *LockManager.sol* ( [14-14](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L14-L14) ):

```solidity
14: contract LockManager is BaseBlastManager, ILockManager, ReentrancyGuard {
```

### [N-23] Missing checks for uint state variable assignments

Consider whether reasonable bounds checks for variables would be useful.

There are 2 instances:

- *LockManager.sol* ( [135-135](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L135-L135), [136-136](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L136-L136) ):

```solidity
135:         APPROVE_THRESHOLD = _approve;

136:         DISAPPROVE_THRESHOLD = _disapprove;
```

### [N-24] Use the Modern Upgradeable Contract Paradigm

Modern smart contract development often employs upgradeable contract structures, utilizing proxy patterns like OpenZeppelin’s Upgradeable Contracts. This paradigm separates logic and state, allowing developers to amend and enhance the contract's functionality without altering its state or the deployed contract address. Transitioning to this approach enhances long-term maintainability.
Resolution: Adopt a well-established proxy pattern for upgradeability, ensuring proper initialization and employing transparent proxies to mitigate potential risks. Embrace comprehensive testing and audit practices, particularly when updating contract logic, to ensure state consistency and security are preserved across upgrades. This ensures your contract remains robust and adaptable to future requirements.

There is 1 instance:

- *LockManager.sol* ( [14](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L14) ):

```solidity
14: contract LockManager is BaseBlastManager, ILockManager, ReentrancyGuard {
```

### [N-25] Large or complicated code bases should implement invariant tests

This includes: large code bases, or code with lots of inline-assembly, complicated math, or complicated interactions between multiple contracts.
Invariant fuzzers such as Echidna require the test writer to come up with invariants which should not be violated under any circumstances, and the fuzzer tests various inputs and function calls to ensure that the invariants always hold.
Even code with 100% code coverage can still have bugs due to the order of the operations a user performs, and invariant fuzzers may help significantly.

There is 1 instance:

- Global finding

### [N-26] The default value is manually set when it is declared

In instances where a new variable is defined, there is no need to set it to it's default value.

There is 1 instance:

- *LockManager.sol* ( [464-464](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L464-L464) ):

```solidity
464:         uint256 lockedWeighted = 0;
```

### [N-27] Contracts should have all `public`/`external` functions exposed by `interface`s

All `external`/`public` functions should extend an `interface`. This is useful to ensure that the whole API is extracted and can be more easily integrated by other projects.

There is 1 instance:

- *LockManager.sol* ( [14](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L14) ):

```solidity
14: contract LockManager is BaseBlastManager, ILockManager, ReentrancyGuard {
```

### [N-28] Top-level declarations should be separated by at least two lines

-

<details>
<summary>There are 7 instances (click to show):</summary>

- *LockManager.sol* ( [2-4](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L2-L4), [12-14](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L12-L14), [63-65](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L63-L65), [83-85](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L83-L85), [127-129](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L127-L129), [309-311](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L309-L311), [494-496](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol#L494-L496) ):

```solidity
2: pragma solidity 0.8.25;
3: 
4: import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

12: import "../interfaces/INFTOverlord.sol";
13: 
14: contract LockManager is BaseBlastManager, ILockManager, ReentrancyGuard {

63:     }
64: 
65:     function _reconfigure() internal {

83:     }
84: 
85:     function configUpdated() external override onlyConfigStorage {

127:     }
128: 
129:     function setUSDThresholds(

309:     }
310: 
311:     function _lock(

494:     }
495: 
496:     function getPlayerSettings(
```

</details>

### [N-29] Consider adding formal verification proofs

Formal verification is the act of proving or disproving the correctness of intended algorithms underlying a system with respect to a certain formal specification/property/invariant, using formal methods of mathematics.

Some tools that are currently available to perform these tests on smart contracts are [SMTChecker](https://docs.soliditylang.org/en/latest/smtchecker.html) and [Certora Prover](https://www.certora.com/).

There is 1 instance:

- [*LockManager.sol*](https://github.com/code-423n4/2024-05-munchables/blob/57dff486c3cd905f21b330c2157fe23da2a4807d/./src/managers/LockManager.sol)

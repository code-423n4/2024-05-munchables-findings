| ID    | Title                                                                                                                 |
| ----- | --------------------------------------------------------------------------------------------------------------------- |
| L-01  | For usd price proposals, the same account can't approve then disapprove, but it's possible to disapprove then approve |
| L-02  | Pausability for `LockManager`                                                                                         |
| L-03  | Downcasting block.timestamp                                                                                           |
| NC-01 | Consider using named imports                                                                                          |
| NC-02 | Variable that computes `_quantity + remainder` could be moved inside block where it's only used                       |
| NC-03 | Consider commenting fields for structs used in `LockManager`                                                          |
| NC-04 | Instantiation of IERC20 could be done once in `lock()`                                                                |

## [L-01] For usd price proposals, the same account can't approve then disapprove, but it's possible to disapprove then approve

Consider disallowing approvals once an account has already disapproved.

https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L225-L226

https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L177-L207

## [L-02] Pausability for `LockManager`

Currently it's possible to pause functions relating to locking and unlocking. Consider allowing to pause the other external functions in the contract as well, and allowing to pause lock and unlock granually, e.g. allowing to pause lock but let unlock open.

## [L-03] Downcasting block.timestamp

Consider adding a bigger type for block.timestamp, currently it's set to stop working after 2106.

https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L381

## [NC-01] Consider using named imports

https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L4-L12

## [NC-02] Variable that computes `_quantity + remainder` could be moved inside block where it's only used

https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L344

https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L352-L355

## [NC-03] Consider commenting fields for structs used in `LockManager`

Consider commenting all struct fields to improve the code documentation.

https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L30

https://github.com/code-423n4/2024-05-munchables/blob/main/src/interfaces/ILockManager.sol#L55-L69

## [NC-04] Instantiation of IERC20 could be done once in `_lock()`

It's possible to instantiate IERC20 only once in `_lock()` if the two conditionals where merged into one.

https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L326-L331

https://github.com/code-423n4/2024-05-munchables/blob/main/src/managers/LockManager.sol#L374-L377

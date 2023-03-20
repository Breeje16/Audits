---
title: Numoen
created: '2023-03-20T07:04:34.303Z'
modified: '2023-03-20T07:05:11.696Z'
---

# Numoen

### [M-1] DIVISION BEFORE MULTIPLICATION ERROR IN CALCULATING INTEREST CAN LEAD TO LARGER PRECISION LOSS

## Impact

There is a division before multiplication bug in `_accrueInterest()` method of `Lendgine.sol` which may result in loss of interest being accrued due to precision. There is same error in `invariant` method of `Pair.sol` as well which can cause larger Precision Loss.

## Proof of Concept

```solidity
File: Lendgine.sol

252:      uint256 dilutionLPRequested = (FullMath.mulDiv(borrowRate, _totalLiquidityBorrowed, 1e18) * timeElapsed) / 365 days;

```
[Link to Code](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/Lendgine.sol#L252)

```solidity
File: Pair.sol

56:      uint256 scale0 = FullMath.mulDiv(amount0, 1e18, liquidity) * token0Scale;
57:      uint256 scale1 = FullMath.mulDiv(amount1, 1e18, liquidity) * token1Scale;

```
[Link to Code](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/Pair.sol#L56-L57)

As in the above 2 cases, Division is taking place before multiplication. In `Lendgine`, `timeElapsed` is multiplied on the result of a division while in `Pair`, `token0Scale` and `token1Scale` are multiplied on the result of a division. This causes Incorrect calculation which can lead to the protocol functioning incorrectly.

## Tools Used

Manual Review

## Recommended Mitigation Steps

consider multiplying all the numerators first before dividing. 

Mitigated code:

```solidity
File: Lendgine.sol

252:      uint256 dilutionLPRequested = FullMath.mulDiv(borrowRate * timeElapsed, _totalLiquidityBorrowed, 1e18) / 365 days;

```
[Link to Code](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/Lendgine.sol#L252)

```solidity
File: Pair.sol

56:      uint256 scale0 = FullMath.mulDiv(amount0 * token0Scale, 1e18, liquidity);
57:      uint256 scale1 = FullMath.mulDiv(amount1 * token1Scale, 1e18, liquidity);

```
[Link to Code](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/Pair.sol#L56-L57)

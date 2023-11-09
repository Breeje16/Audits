## Medium and High Risk Issues

| Count | Explanation |
|:--:|:-------|
| [M-1] | `FlywheelCore.setBooster()` can be used to steal unclaimed rewards |
| [M-2] | Interactions with AMMs do not use deadlines for operations |
| [M-3] | Many contracts are suspicious of the reorg attack |
| [M-4] | No Deadline for operations in `UlyssesRouter` |
| [H-1] | First ERC4626 deposit exploit can break share calculation |
| [H-2] | Lack of valid Slippage control parameter in `increaseLiquidity` calls in Talos |
| [H-3] | `doRebalance` in Talos is vulnerable to Flash loan Attacks resulting loss of funds |

| Issue Type | Count |
|:--:|:--:|
| Total Medium Risk Issues | 4 |
| Total High Risk Issues | 3 |

### [M-01] FlywheelCore’s setFlywheelRewards can remove access to reward funds from current users introducing Rug Pull Attack Vector

## Impact

FlywheelCore.setFlywheelRewards can remove current reward funds from the current users’ reach as it doesn’t check that newFlywheelRewards’ FlywheelCore is this contract.

If it’s not, by mistake or with a malicious intent, the users will lose the access to reward funds as this FlywheelCore will not be approved for any fund access to the new flywheelRewards, while all the reward funds be moved there.

Presence of a rug pool attack vector introduces the risk of negatively impacting the protocol’s reputation.

## Proof of Concept

FlywheelCore.setFlywheelRewards doesn’t check that newFlywheelRewards’ FlywheelCore is this FlywheelCore instance:

```solidity
File: FlywheelCore.sol

  function setFlywheelRewards(IFlywheelRewards newFlywheelRewards) external requiresAuth {
      uint256 oldRewardBalance = rewardToken.balanceOf(address(flywheelRewards));
      if (oldRewardBalance > 0) {
          rewardToken.safeTransferFrom(address(flywheelRewards), address(newFlywheelRewards), oldRewardBalance);
      }

      flywheelRewards = newFlywheelRewards;

      emit FlywheelRewardsUpdate(address(newFlywheelRewards));
  }

```
[Link to Code](https://github1s.com/code-423n4/2023-05-maia/blob/main/src/rewards/base/FlywheelCore.sol#L125-L134)

FlywheelCore is immutable within flywheelRewards and its access to the flywheelRewards’ funds is set on construction:

```solidity
File: BaseFlywheelRewards.sol

  rewardToken.safeApprove(_uniswapV3Staker, type(uint256).max);

```

This way if new flywheelRewards contract have any different FlywheelCore then current users’ access to reward funds will be irrevocably lost as both claiming functionality and next run of setFlywheelRewards will revert, not being able to transfer any funds from flywheelRewards with `rewardToken.safeTransferFrom(address(flywheelRewards), ...)`:

```solidity
File: FlywheelCore.sol

  function claimRewards(address user) external {
      uint256 accrued = rewardsAccrued[user];

      if (accrued != 0) {
          rewardsAccrued[user] = 0;

@->       rewardToken.safeTransferFrom(address(flywheelRewards), user, accrued);

          emit ClaimRewards(user, accrued);
      }
  }

```

```solidity
File: FlywheelCore.sol

  function setFlywheelRewards(address newFlywheelRewards) external onlyOwner {
        uint256 oldRewardBalance = rewardToken.balanceOf(address(flywheelRewards));
        if (oldRewardBalance > 0) {
@->         rewardToken.safeTransferFrom(address(flywheelRewards), address(newFlywheelRewards), oldRewardBalance);
        }

        flywheelRewards = newFlywheelRewards;

        emit FlywheelRewardsUpdate(address(newFlywheelRewards));
    }

```

As FlywheelCore holds user funds accounting via rewardsAccrued mapping, all these accounts became non-operational, as all the unclaimed rewards will be lost for the users.

## Tools Used

VS Code

## Recommended Mitigation Steps

Consider adding the require for `address(newFlywheelRewards.flywheel) == address(flywheelRewards.flywheel)` in `setFlywheelRewards` so that users always retain funds access.

### [M-02] Interactions with Pool do not use valid deadlines for operations

## Impact

Miner can potentially hold the transaction which results in loss of funds for users.

## Proof of Concept

```solidity
File: TalosBaseStrategy.sol

    (liquidityDifference, amount0, amount1) = nonfungiblePositionManager.increaseLiquidity(
        INonfungiblePositionManager.IncreaseLiquidityParams({
            tokenId: _tokenId,
            amount0Desired: amount0Desired,
            amount1Desired: amount1Desired,
            amount0Min: 0,     
            amount1Min: 0,
 @->        deadline: block.timestamp        
        })
    );

```
[Link to code](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L208)

AMMs provide their users with an option to limit the execution of their pending actions, such as swaps or adding and removing liquidity. The most common solution is to include a deadline timestamp as a parameter (for example in Uniswap V2 and Uniswap V3). If such an option is not present, users can unknowingly perform bad trades as interaction with the pools is very time sensitive, if transaction gets hold for sometime than it can lead to bad trades unknowingly by the Users and can potentially result in loss of funds.

All the functions in the codebase that interact with uniswap pools have a deadline parameter passed as `block.timestamp`, which means that whenever the miner decides to include the transaction in a block, it will be valid at that time, since `block.timestamp` will be the current timestamp at the time of mining. but here, there is a possibility of a malicious miner to hold the transaction how whatever amount he/she wants.

This is tbe reason, instead of `block.timestamp`, an extra arguments should be passed with the functions and user should be allowed to pass a valid deadline for it so that in case a malicious miner tries to hold it, then the transaction reverts as per the deadline specified by the user. Having slippage is an issue for any AMMs but having outdated Slippage protection can also be a dangerous attack vector.

Link to a similar issue reported in [Paraspace Audit](https://code4rena.com/reports/2022-11-paraspace#m-13-interactions-with-amms-do-not-use-deadlines-for-operations) by Code4rena.

## Tools Used

VS Code

## Recommended Mitigation Steps

Add deadline arguments in all functions that interact with uniswap pools, and pass it along to these calls.


### [M-03] Many contracts are suspicious of the reorg attack

## Proof of Concept

There are many instance of this, but to understand things better, taking the example of `createTalosV3Strategy` method.

The `createTalosV3Strategy` function deploys a new `TalosStrategyStaked` contract using the create, where the address derivation depends only on the arguments passed.

At the same time, some of the chains like Arbitrum and Polygon are suspicious of the reorg attack.

```solidity
File: TalosStrategyStaked.sol

  function createTalosV3Strategy(
        IUniswapV3Pool pool,
        ITalosOptimizer optimizer,
        BoostAggregator boostAggregator,
        address strategyManager,
        FlywheelCoreInstant flywheel,
        address owner
    ) public returns (TalosBaseStrategy) {
        return new TalosStrategyStaked( // @audit-issue Reorg Attack
                pool,
                optimizer,
                boostAggregator,
                strategyManager,
                flywheel,
                owner
            );
    }

```
[Link to Code](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/TalosStrategyStaked.sol#L28)

Even more, the reorg can be couple of minutes long. So, it is quite enough to create the `TalosStrategyStaked` and transfer funds to that address using `deposit` method, especially when someone uses a script, and not doing it by hand.

Optimistic rollups (Optimism/Arbitrum) are also suspect to reorgs since if someone finds a fraud the blocks will be reverted, even though the user receives a confirmation.

Same issue can affect factory contracts in Ulysses omnichain contracts as well with more severe consequences.

Can refer this an issue previously report [here](https://code4rena.com/reports/2023-04-frankencoin#m-14-re-org-attack-in-factory) to have more understanding about it.

## Impact

Exploits involving Stealing of funds.

## Tools Used

VS Code

## Recommended Mitigation Steps

Deploy such contracts via `create2` with `salt`.


### [M-04] No Deadline check in `UlyssesRouter`, allowing outdated slippage and allow pending transaction to be unexpected executed

## Proof of Concept

AMMs provide their users with an option to limit the execution of their pending actions, such as swaps or adding and removing liquidity. The most common solution is to include a deadline timestamp as a parameter (for example: Uniswap V2 and Uniswap V3). If such an option is not present, users can unknowingly perform bad trades.

Currently there are 3 external methods in `UlyssesRouter`:

1. `addLiquidity`
2. `removeLiquidity`
3. `swap`

2 things are most important while interacting with AMMs:

1. Slippage parameter: Which is controlled here by user by passing the value of `minOutput` as a function parameter.
2. Deadline of Execution: In case, the transaction is submitted to the mempool and transaction fee is too low for miners to be interested in including the transaction in a block immediately, then the transaction can stay in the mempool for some time. If it is executed at a later time, there is high chance that the slippage parameter earlier passed might result in higher slippage than what user originally anticipated. This can result in loss of funds.

A MEV bot can detect the pending transactions. Since the outdated maximum slippage value now allows for high slippage, the bot sandwiches the transaction, resulting in significant profit for the bot and significant loss for user.

This second part is not handled in `UlyssesRouter`.

## Impact

Potential loss of funds

## Tools Used

VS Code

## Recommended Mitigation Steps

Add a deadline check exactly how uniswap does in functions related to adding liquidity, removing liquidity and swapping.

### [H-01] First ERC4626 deposit exploit can break share calculation

## Impact

A well known attack vector for almost all shares based liquidity pool contracts, where an early user can manipulate the price per share and profit from late users' deposits because of the precision loss caused by the rather large value of price per share.

## Proof of Concept

Problems with the code:
1. Integer division negatively affect the user.
2. Can be manipulated to cause a large loss, specifically for first victim.

Consider the following situation:
1. Attacker deposits 1 wei of Token.
2. Next, Attacker transfers 100 Token to the contract.
3. Victim deposits 200 Token.
3. Attacker withdraws 1 share.

Have a look at this table to understand the complete PoC:

|                                              | Before      | Before          |              | After       | After           |
|----------------------------------------------|-------------|-----------------|--------------|-------------|-----------------|
| Tx                                           | totalSupply | balanceOf       | sharesGiven | totalSupply | balanceOf       |
| BeforeAttacker deposits 1 wei of Token.       | 0           | 0               | 1            | 1           | 1               |
| Attacker transfers 100 Token to the contract. | 1           | 1               | N/A          | 1           | 1 + 100 x 10^18 |
| Victim deposits 200 Token.                    | 1           | 1 + 100 x 10^18 | =1.99 = 1    | 2           | 1 + 300 x 10^18 |
| Attacker withdraws 1 share.                  | 2           | 1 + 300 x 10^18 | N/A          | 1           | 1 + 150 x 10^18 |

```solidity
File: ERC4626.sol

    function convertToShares(uint256 assets) public view virtual returns (uint256) {
        uint256 supply = totalSupply; // Saves an extra SLOAD if totalSupply is non-zero.

        return supply == 0 ? assets : assets.mulDiv(supply, totalAssets());
    }

```
[Link to Code](https://github.com/code-423n4/2023-05-maia/blob/main/src/erc-4626/ERC4626.sol#L106-L110)

## Tools Used

VS Code

## Recommended Mitigation Steps

1. Need to Enforce a minimum deposit that can't be withdrawn.
2. So, add some of the initial amount to the zero address.

### [H-02] Lack of valid Slippage control parameter in `increaseLiquidity` calls in Talos

## Impact

Loss of funds due to Allowing Maximum Slippage.

## Proof of Concept

There are many instances throughout the codebase where slippage protection has not been used and `0` has been passed to slippage control instead.

Eg: `deposit` method in `TalosBaseStrategy`:

```solidity
File: TalosBaseStrategy.sol

  (liquidityDifference, amount0, amount1) = nonfungiblePositionManager.increaseLiquidity(
        INonfungiblePositionManager.IncreaseLiquidityParams({
            tokenId: _tokenId,
            amount0Desired: amount0Desired,
            amount1Desired: amount1Desired,
@->         amount0Min: 0,      // @audit-issue No Slippage Protection
@->         amount1Min: 0,
            deadline: block.timestamp        // @audit-issue Miner controlled Deadline
        })
    );

```
[Link to code](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L144-L145)

In Uniswap's example docs [here](https://docs.uniswap.org/contracts/v3/guides/providing-liquidity/increase-liquidity), it is clearly stated that:

>We cannot change the boundaries of a given liquidity position using the Uniswap v3 protocol; increaseLiquidity can only increase the liquidity of a position.

>In production, amount0Min and amount1Min should be adjusted to create slippage protections.

It is worth noting the second point that `amount0Min` and `amount1Min` acts as a slippage protection and it should be adjusted in the production in order to protect the users from High Slippage.

But in the following instance, this slippage parameter is set to zero which is exposing the transaction to maximum slippage:

* [TalosStrategyVanilla.sol](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/TalosStrategyVanilla.sol#L146-L147)
* [TalosBaseStrategy.sol 1](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L144-L145)
* [TalosBaseStrategy.sol 2](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L206-L207)
* [TalosBaseStrategy.sol 3](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L357-L358)
* [PoolActions.sol](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/libraries/PoolActions.sol#L82-L83)

## Tools Used

VS Code

## Recommended Mitigation Steps

Allow users to pass the value of `amountXMin` as parameter.

### [H-03] `doRebalance` in Talos is vulnerable to Flash loan Attacks resulting loss of funds

## Impact

Loss of funds due to MEV Sandwich attacks.

## Proof of Concept

Rebalancing is done using `doRebalance` method in `TalosStrategySimple`. 

```solidity
File: TalosStrategySimple.sol

  function doRebalance() internal override returns (uint256 amount0, uint256 amount1) {
      int24 baseThreshold = tickSpacing * optimizer.tickRangeMultiplier();

      PoolActions.ActionParams memory actionParams =
          PoolActions.ActionParams(pool, optimizer, token0, token1, tickSpacing);

@->   PoolActions.swapToEqualAmounts(actionParams, baseThreshold);

      (tickLower, tickUpper, amount0, amount1, tokenId, liquidity) =
          nonfungiblePositionManager.rerange(actionParams, poolFee);
  }

```
[Link to code](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/strategies/TalosStrategySimple.sol#L42)

Line in focus here is:

```solidity
    PoolActions.swapToEqualAmounts(actionParams, baseThreshold);
```

`swapToEqualAmounts` method in `PoolActions.sol` library:

```solidity
File: PoolActions.sol

  function swapToEqualAmounts(ActionParams memory actionParams, int24 baseThreshold) internal {
@->   (bool zeroForOne, int256 amountSpecified, uint160 sqrtPriceLimitX96) = actionParams
          .pool
          .getSwapToEqualAmountsParams(
          actionParams.optimizer, actionParams.tickSpacing, baseThreshold, actionParams.token0, actionParams.token1
      );

      //Swap imbalanced token as long as we haven't used the entire amountSpecified and haven't reached the price limit
      actionParams.pool.swap(
          address(this),
          zeroForOne,
          amountSpecified,
 @->      sqrtPriceLimitX96,
          abi.encode(SwapCallbackData({zeroForOne: zeroForOne}))
      );
  }

```
[Link to Code](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/libraries/PoolActions.sol#L50)

Here, the value of `sqrtPriceLimitX96` is calculated through `getSwapToEqualAmountsParams` method in `PoolVariables.sol` library. It basically represents the square root of the lowest or highest price that you are willing to perform the trade at. So basically it is a slippage control parameter.

```solidity
File: PoolVariables.sol

232:    (uint160 sqrtPriceX96, int24 currentTick,,,,,) = _pool.slot0();

251:    uint160 exactSqrtPriceImpact = (sqrtPriceX96 * (_strategy.priceImpactPercentage() / 2)) / GLOBAL_DIVISIONER;
252:    sqrtPriceLimitX96 = zeroForOne ? sqrtPriceX96 - exactSqrtPriceImpact : sqrtPriceX96 + exactSqrtPriceImpact;

```

`getSwapToEqualAmountsParams` method uses the `_pool.slot0` to determine The current price of the pool as a sqrt. [slot0](https://docs.uniswap.org/contracts/v3/reference/core/interfaces/pool/IUniswapV3PoolState#slot0) is the most recent data point and can easily be manipulated using Flash Loans.

Because of this, the value of `sqrtPriceLimitX96` can be manipulated and that value will be passed in `swap` method.

With this knowledge, a malicious MEV bot could watch for these transactions in the mempool. When it sees such a transaction, it could perform a "sandwich attack", trading massively in the same direction as the trade in advance of it to push the price out of whack, and then trading back after us, so that they end up pocketing a profit at our expense.

## Tools Used

VS Code

## Recommended Mitigation Steps

Use `TWAP` rather than `slot0`.

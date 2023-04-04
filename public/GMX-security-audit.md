# GMX Security Audit

## Medium and High Risk Issues
| Count | Explanation |
|:--:|:-------|
| [M-1] | Oracle data feed can be outdated |
| [M-2] | Missing deadline checks allow pending transactions to be maliciously executed |
| [M-3] | Multiplication after Division error leading to larger precision loss |
| [H-1] | No slippage protection in `createAdlOrder` allows MEV Attacks and Loss of funds |
| [H-2] | No Slippage protection in `swap` allows MEV Attack and Loss of Funds while executing Deposit |

| Issue Type | Count |
|:--:|:--:|
| Total Medium Risk Issues | 3 |
| Total High Risk Issues | 2 |

### [M-1] Oracle data feed can be outdated

## Summary

Price used by the contract through the oracle can be stale because of Lack of Validation on Oracle price.

## Vulnerability Detail

In `Oracle.sol` contract, `_setPricesFromPriceFeeds` method uses latestRoundData() function to get price from chainlink.

However, neither round completeness or the quoted timestamp are checked to ensure that the reported price is not stale. 

```
function latestRoundData() external view
    returns (
        uint80 roundId,
        int256 answer, 
        uint256 startedAt,
        uint256 updatedAt,
        uint80 answeredInRound
    )
```

That's the reason Chainlink recommends using their data feeds along with some controls to prevent mismatches with the retrieved data.

## Impact

The retrieved price of the priceFeed can be outdated and used anyways as a valid data because no timestamp tolerance of the update source time is checked while storing the return parameters of priceFeed.latestRoundData(). The usage of outdated data can impact on how the further logics of that price are implemented.

## Code Snippet

```solidity
File: Oracle.sol

    IPriceFeed priceFeed = getPriceFeed(dataStore, token);

    (
        /* uint80 roundID */,
        int256 _price,
        /* uint256 startedAt */,
        /* uint256 timestamp */,
        /* uint80 answeredInRound */
    ) = priceFeed.latestRoundData();

    uint256 price = SafeCast.toUint256(_price);

```
[Link to Code](https://github.com/sherlock-audit/2023-02-gmx-Breeje16/blob/master/gmx-synthetics/contracts/oracle/Oracle.sol#L569-L579)


## Tool used

Manual Review

## Recommendation

As Chainlink recommends:

>Your application should track the latestTimestamp variable or use the updatedAt value from the latestRoundData() function to make sure that the latest answer is recent enough for your application to use it. If your application detects that the reported answer is not updated within the heartbeat or within time limits that you determine are acceptable for your application, pause operation or switch to an alternate operation mode while identifying the cause of the delay.

Add a check for timestamp such that last price update has happened within the acceptable `heartbeat` period.

Mitigated Code:

```diff
File: Oracle.sol

    IPriceFeed priceFeed = getPriceFeed(dataStore, token);

    (
-       /* uint80 roundID */,
+       uint80 roundID,
        int256 _price,
-       /* uint256 startedAt */,
+       uint256 startedAt,
-       /* uint256 timestamp */,
+       uint256 timestamp,
-       /* uint80 answeredInRound */
+       uint80 answeredInRound
    ) = priceFeed.latestRoundData();
+   require(answer > 0, "Chainlink: Incorrect Price");
+   require(block.timestamp - updatedAt < HEARTBEAT_PERIOD, "Chainlink: Stale Price");
    uint256 price = SafeCast.toUint256(_price);

```

### [M-2] Missing deadline checks allow pending transactions to be maliciously executed

## Summary

In `SwapUtils` contract, `swap` does not allow users to submit a deadline for their action. This missing feature enables pending transactions to be maliciously executed at a later point.

## Vulnerability Detail

Here's how MEV Bot can exploit it without deadline check:

1. The swap transaction is still pending in the mempool. The price of 1 token has gone up significantly since the transaction was signed, meaning Alice would receive a lot more second token when the swap is executed. But that also means that her minimum Output Amount value is outdated and would allow for significant slippage.
2. A MEV bot detects the pending transaction. Since the outdated minimum Output Amount now allows for high slippage, the bot sandwiches Alice, resulting in significant profit for the bot and significant loss for Alice.

This is the reason any critical functionality where user is going to interact with the pool should be passed with a deadline of execution. As pool balance and result out of interact with pool is time dependent, there should not be any attack vector left for attackers to manipulate. By setting deadline such attack vector will be mitigated.

## Impact

Loss of funds.

## Code Snippet

```solidity
File: SwapUtils.sol

98:    function swap(SwapParams memory params) external returns (address, uint256) {

```
[Link to Code](https://github.com/sherlock-audit/2023-02-gmx-Breeje16/blob/master/gmx-synthetics/contracts/swap/SwapUtils.sol#L98)

## Tool used

Manual Review

## Recommendation

Introduce a deadline parameter to the mentioned functions.

### [M-3] Multiplication after Division error leading to larger precision loss

## Summary

There are couple of instance of using result of a division for multiplication while can cause larger precision loss.

## Vulnerability Detail

```solidity
File: MarketUtils.sol

948:    cache.fundingUsd = (cache.sizeOfLargerSide / Precision.FLOAT_PRECISION) * cache.durationInSeconds * cache.fundingFactorPerSecond;

952:    if (result.longsPayShorts) {
953:          cache.fundingUsdForLongCollateral = cache.fundingUsd * cache.oi.longOpenInterestWithLongCollateral / cache.oi.longOpenInterest;
954:          cache.fundingUsdForShortCollateral = cache.fundingUsd * cache.oi.longOpenInterestWithShortCollateral / cache.oi.longOpenInterest;
955:      } else {
956:          cache.fundingUsdForLongCollateral = cache.fundingUsd * cache.oi.shortOpenInterestWithLongCollateral / cache.oi.shortOpenInterest;
957:          cache.fundingUsdForShortCollateral = cache.fundingUsd * cache.oi.shortOpenInterestWithShortCollateral / cache.oi.shortOpenInterest;
958:      }

```
[Link to Code](https://github.com/sherlock-audit/2023-02-gmx-Breeje16/blob/master/gmx-synthetics/contracts/market/MarketUtils.sol#L948)


In above case, value of `cache.fundingUsd` is calculated by first dividing `cache.sizeOfLargerSide` with `Precision.FLOAT_PRECISION` which is `10**30`. Then the resultant is multiplied further. This result in larger Loss of precision.

Later the same `cache.fundingUsd` is used to calculate `cache.fundingUsdForLongCollateral` and `cache.fundingUsdForShortCollateral` by multiplying further which makes the precision error even more big.

Same issue is there in calculating `cache.positionPnlUsd` in `PositionUtils`.

```solidity
File: PositionUtils.sol

    if (position.isLong()) {
            cache.sizeDeltaInTokens = Calc.roundUpDivision(position.sizeInTokens() * sizeDeltaUsd, position.sizeInUsd());
        } else {
            cache.sizeDeltaInTokens = position.sizeInTokens() * sizeDeltaUsd / position.sizeInUsd();
        }
    }

    cache.positionPnlUsd = cache.totalPositionPnl * cache.sizeDeltaInTokens.toInt256() / position.sizeInTokens().toInt256();

```
[Link to Code](https://github.com/sherlock-audit/2023-02-gmx-Breeje16/blob/master/gmx-synthetics/contracts/position/PositionUtils.sol#L217-L224)

## Impact

Precision Loss in accounting.

## Code Snippet

Given above.

## Tool used

Manual Review

## Recommendation

First Multiply all the numerators and then divide it by the product of all the denominator.

### [H-1] No slippage in `createAdlOrder` allows MEV Attacks and Loss of funds

## Summary

In `AdlUtils`, `Order.Numbers` data type has `0` value set for the `minOutputAmount` which can lead to sandwich attack and loss of funds. 

## Vulnerability Detail

In `createAdlOrder` method of  `AdlUtils`, there is no slippage control which means that a malicious actor could, e.g., trivially insert transactions before and after the naive transaction (using the infamous "sandwich" attack), causing the smart contract to trade at a radically worse price, profit from this at the caller's expense, and then return the contracts to their original state, all at a low cost.

## Impact

Loss of Funds.

## Code Snippet

```solidity
File: AdlUtils.sol

  Order.Numbers memory numbers = Order.Numbers(
      Order.OrderType.MarketDecrease, // orderType
      Order.DecreasePositionSwapType.NoSwap, // decreasePositionSwapType
      params.sizeDeltaUsd, // sizeDeltaUsd
      0, // initialCollateralDeltaAmount
      0, // triggerPrice
      position.isLong() ? 0 : type(uint256).max, // acceptablePrice
      0, // executionFee
      0, // callbackGasLimit
      0, // minOutputAmount
      params.updatedAtBlock // updatedAtBlock
  );

```
[Link to Code](https://github.com/sherlock-audit/2023-02-gmx-Breeje16/blob/master/gmx-synthetics/contracts/adl/AdlUtils.sol#L142-L153)

## Tool used

Manual Review

## Recommendation

Add a valid value for `minOutputAmount` to protect users against slippage.

### [H-2] No Slippage protection in `swap` allows MEV Attack and Loss of Funds while executing Deposit

## Summary

In `ExecuteDepositUtils`, `swap` method is used in `executeDeposit` methods which is vulnerable to MEV Attacks.

## Vulnerability Detail

In `swap` method, the `SwapParams` passed contains `minOutputAmount` as Zero. There is no slippage control here which means that a malicious actor could, e.g., trivially insert transactions before and after the naive transaction (using the infamous "sandwich" attack), causing the smart contract to trade at a radically worse price, profit from this at the caller's expense, and then return the contracts to their original state, all at a low cost.

Here's how the attack can happen:

1. MEV Bot or any attacker Detect this `swap` transaction in mempool.
2. It Front-Run the victim’s transaction. Can possibly use Flash Loans for it to maximise the damage.
3. As the Pool balance is manipulated, Victim transacts with this pool and suffers higher slippage.
4. The attacker then back-runs the victim, and maximize it's profit from the loss of victim which is this migration is our case.

## Impact

Loss of Funds.

## Code Snippet

```solidity
File: ExecuteDepositUtils.sol

  (address outputToken, uint256 outputAmount) = SwapUtils.swap(
      SwapUtils.SwapParams(
          params.dataStore, // dataStore
          params.eventEmitter, // eventEmitter
          params.oracle, // oracle
          params.depositVault, // bank
          initialToken, // tokenIn
          inputAmount, // amountIn
          swapPathMarkets, // swapPathMarkets
          0, // minOutputAmount
          market, // receiver
          false // shouldUnwrapNativeToken
      )
  );

```
[Link to Code](https://github.com/sherlock-audit/2023-02-gmx-Breeje16/blob/master/gmx-synthetics/contracts/deposit/ExecuteDepositUtils.sol#L429-L442)

## Tool used

Manual Review

## Recommendation

Add a valid value for slippage or add a parameter `expectedOutputAmount` and check that should be equal to `outputToken`.

## Medium and High Risk Issues

| Count | Explanation |
|:--:|:-------|
| [M-1] | Hard-coded slippage may freeze user funds during market turbulence |
| [M-2] | liquidators can hinder users from restoring their positions to a healthy state and force liquidation |
| [M-3] | No check if Arbitrum L2 sequencer is down in Chainlink feeds. |
| [H-1] | Anyone can call `withdrawAllMarketFees` and steal Majority of the fees |
| [H-2] | Curve oracle manipulation with read only reentrancy in pool's `get_virtual_price` |
| [H-3] | Unprotected `delegatecall` allows stealing of funds from `BaseTOFTLeverageModule` |

| Issue Type | Count |
|:--:|:--:|
| Total Medium Risk Issues | 3 |
| Total High Risk Issues | 3 |

### [M-01] Hard-coded slippage may freeze user funds during market turbulence

## Impact

DoS in critical Functionality like Withdrawal or emergency withdrawal.

## Proof of Concept

A call to `compound` function is made from critical functions like:

* `emergencyWithdraw`
* `_withdraw`

The issue here is with respect to Slippage parameter.

It is important to note that for in case `minAmount` value is not reached, which is possible during the market turbulance, the transaction will revert.

```solidity
File: AaveStrategy.sol

193:    uint256 minAmount = calcAmount - (calcAmount * 50) / 10_000;
194:    swapper.swap(swapData, minAmount, address(this), "");

```
[Link to code](https://github.com/Tapioca-DAO/tapioca-yieldbox-strategies-audit/blob/05ba7108a83c66dada98bc5bc75cf18004f2a49b/contracts/aave/AaveStrategy.sol#L193-L194)

The code sets a small slippage of 0.5% which is good enough in normal market conditions. While this may protect users from losing funds due to slippage, during times of high volatility when slippage is unavoidable it will also cause this function of `compound` to always revert. 

If there is a DoS because of this in `compound` function, `emergencyWithdraw` and `_withdraw` function will DoSed too because of using `compound` function.

If a project uses a default slippage, there should always be a way to override this slippage with an external parameter to ensure that this transaction can go through even during times of high volatility.

This issue exists in almost every Strategies.

## Tools Used

VS Code

## Recommended Mitigation Steps

Have a slippage control parameter that can be set by the Owner with a Max Cap. This will make sure than during market turbulance, the Owner will have option to change the slippage value to Mitigate DoS Issues.

```solidity

  function setMaxSlippage(uint256 _slippage) external onlyOwner {
      maxSlippage = _slippage;

      emit SetMaxSlippage(msg.sender, slippage);
  }
```
```solidity
File: AaveStrategy.sol

193:    uint256 minAmount = calcAmount * maxSlippage / 10_000;
194:    swapper.swap(swapData, minAmount, address(this), "");

```


### [M-02] liquidators can hinder users from restoring their positions to a healthy state and force liquidation

## Impact

The liquidator can unfairly liquidate user positions as soon as the protocol is unpaused after a pause. This causes loss of funds to the user.

## Proof of Concept

In the `BigBang` contract, during a paused state, key operations are halted.

The `conservator` address can pause the BigBang functionalities using the `Market.updatePause` function. Once the contract is paused all these operations cannot be performed by users:

* `borrow`
* `repay`
* `addCollateral`
* `removeCollateral`
* `liquidate`
* `buyCollateral`
* `sellCollateral`

It is important to note that during the paused period, actual oracle prices can experience fluctuations. If the oracle prices go down, the users won't be allowed to add more collateral to their positions or close their positions.

Upon unpausing, liquidators can preemptively initiate the liquidation of user positions before they have an opportunity to stabilize their positions.

The absence of a grace period upon unpausing allows liquidators to exploit this window and trigger liquidations immediately, depriving users of the chance to restore their positions.

This vulnerability exposes users to financial loss.

```solidity
File: BigBang.sol

    function repay(
        address from,
        address to,
        bool,
        uint256 part
@-> ) public notPaused allowedBorrow(from, part) returns (uint256 amount) {
        updateExchangeRate();

        accrue();

        amount = _repay(from, to, part);
    }

    function liquidate(
        address[] calldata users,
        uint256[] calldata maxBorrowParts,
        ISwapper swapper,
        bytes calldata collateralToAssetSwapData
@-> ) external notPaused {

```
[Link to code](https://github.com/Tapioca-DAO/tapioca-bar-audit/blob/2286f80f928f41c8bc189d0657d74ba83286c668/contracts/markets/bigBang/BigBang.sol#L314)

Also Sharing a Ref to similar issue reported for Blueberry protocol: [Link](https://github.com/sherlock-audit/2023-02-blueberry-judging/issues/290)

## Tools Used

VS Code

## Recommended Mitigation Steps

Consider implementing a post-unpause grace period during which liquidation remains restricted. This would allow users time to rectify their positions and prevent unfair liquidation.

### [M-03] No check if Arbitrum L2 sequencer is down in Chainlink feeds.

## Impact

Stale Prices from the oracle.

## Proof of Concept

Using Chainlink in L2 chains such as Arbitrum requires to check if the sequencer is down to avoid prices from looking like they are fresh although they are not.

The bug could be leveraged by malicious actors to take advantage of the sequencer downtime.

There is no check in `SGOracle` that the sequencer is down:

```solidity
File: SGOracle.sol

    function _get() internal view returns (uint256 _maxPrice) {
        uint256 lpPrice = (SG_POOL.totalLiquidity() *
            uint256(UNDERLYING.latestAnswer())) / SG_POOL.totalSupply();

        return lpPrice;
    }

```
[Link to code](https://github.com/Tapioca-DAO/tapioca-periph-audit/blob/a3b45512580f8a76be45c19f635689f48c0128c3/contracts/oracle/implementations/SGOracle.sol#L51)

## Tools Used

VS Code

## Recommended Mitigation Steps

It is recommended to follow the code example of Chainlink:
https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-code

### [M-04] Oracle Data is insufficiently Validated

## Impact

Stale Data from the oracle.

## Proof of Concept

Chainlink oracles updates the value when one of the 2 Trigger happens:

1. If `Deviation Threshold` exceeds. Deviation Threshold is the specific amount of deviation in prices which happens after which a new aggregation round will start and the price will be updated.
2. If `Heartbeat Threshold` exceeds. Heartbeat Threshold is a specific amount of time from the last update after which a new aggregation round will start and the price will be updated.
Link: https://docs.chain.link/architecture-overview/architecture-decentralized-model/

But in `_get()` function, Heartbeat Threshold is not implemented to check whether the last observed price was stale or not.

```solidity
File: SGOracle.sol

    function _get() internal view returns (uint256 _maxPrice) {
        uint256 lpPrice = (SG_POOL.totalLiquidity() *
            uint256(UNDERLYING.latestAnswer())) / SG_POOL.totalSupply();

        return lpPrice;
    }

```
[Link to code](https://github.com/Tapioca-DAO/tapioca-periph-audit/blob/a3b45512580f8a76be45c19f635689f48c0128c3/contracts/oracle/implementations/SGOracle.sol#L51)

Chainlink in there docs have clearly written: "Your application should track the latestTimestamp variable or use the updatedAt value from the latestRoundData() function to make sure that the latest answer is recent enough for your application to use it. If your application detects that the reported answer is not updated within the heartbeat or within time limits that you determine are acceptable for your application, pause operation or switch to an alternate operation mode while identifying the cause of the delay."

So by adding the require check as recommended, it can stop the protocol from using the stale price which is even recommended by chainlink.

Link: https://docs.chain.link/data-feeds/#check-the-timestamp-of-the-latest-answer

## Tools Used

VS Code

## Recommended Mitigation Steps

As Chainlink recommends:

> Your application should track the latestTimestamp variable or use the updatedAt value from the latestRoundData() function to make sure that the latest answer is recent enough for your application to use it. If your application detects that the reported answer is not updated within the heartbeat or within time limits that you determine are acceptable for your application, pause operation or switch to an alternate operation mode while identifying the cause of the delay.

Recommended Mitigation:

Add the Following 2 more require Statements in the `_get()` method to mitigate the issue after setting the HEARTBEAT_PERIOD as set by chainlink.

```diff
File: SGOracle.sol

    function _get() internal view returns (uint256 _maxPrice) {

+       (uint80 roundID, int256 price, , uint256 updatedAt, uint80 answeredInRound) = UNDERLYING.latestRoundData();   
+       require(price > 0, "invalid price");
+       require(block.timestamp - updatedAt < HEARTBEAT_PERIOD, "Chainlink: Stale Price");
+       require(answeredInRound >= roundID, "Chainlink: Stale Price"); 
        uint256 lpPrice = (SG_POOL.totalLiquidity() *
-           uint256(UNDERLYING.latestAnswer())) / SG_POOL.totalSupply();
+           uint256(price)) / SG_POOL.totalSupply();

        return lpPrice;
    }

```
[Link to code](https://github.com/Tapioca-DAO/tapioca-periph-audit/blob/a3b45512580f8a76be45c19f635689f48c0128c3/contracts/oracle/implementations/SGOracle.sol#L51)

### [H-01] Anyone can call `withdrawAllMarketFees` and steal Majority of the fees

## Impact

Stealing of all Market Fees.

## Proof of Concept

`Penrose` contract have a `public` accessed function: `withdrawAllMarketFees`. 

```solidity
File: Penrose.sol

    function withdrawAllMarketFees(
          IMarket[] calldata markets_,
          ISwapper[] calldata swappers_,
          IPenrose.SwapData[] calldata swapData_
 @->  ) public notPaused {

```

After input validation, it calls `_withdrawAllProtocolFees` which handles the main functionality by Looping through the master contracts and call `_depositFeesToYieldBox()` to each one of their clones.

```solidity
File: Penrose.sol

    function _depositFeesToYieldBox(
        IMarket market,
        ISwapper swapper,
        IPenrose.SwapData calldata dexData
    ) private {
        //----SNIP: Input Validation-----//

        uint256 feeShares = market.refreshPenroseFees(feeTo);
        if (feeShares == 0) return;

        uint256 assetId = market.assetId();
        uint256 amount = 0;
        if (assetId != wethAssetId) {
            //----SNIP: Transfer asset received from market to swapper-----//
            
            (amount, ) = swapper.swap(
                swapData,
  @->           dexData.minAssetAmount,
                feeTo,
                ""
            );
        } else {
            yieldBox.transfer(address(this), feeTo, assetId, feeShares);
        }

        emit LogYieldBoxFeesDeposit(feeShares, amount);
    }

```

Here, in case asset is not weth, it uses a `swapper` to swap the asset which came from market to `TAP`.

Now consider the following situation:

1. As Anyone can call `withdrawAllMarketFees`, Alice calls it with a whitelisted swapper and market (to bypass validation) but uses `IPenrose.SwapData[] calldata swapData_` = 0 for every market.
2. SwapData contains only one field: `minAssetAmount` which is the slippage protection.
3. Now with value of `dexData.minAssetAmount = 0`, the swap done in `_depositFeesToYieldBox` is vulnerable to Sandwich attack.
4. This allows an attacker Alice to steal a significant portion of fees, exploiting the transaction at the expense of the fee recipient.

## Tools Used

VS Code

## Recommended Mitigation Steps

I recommend to either add an access control on `withdrawAllMarketFees` or add a require check to ensure `swapData_` value is atleast above a minimum percentage to prevent potential misuse.

### [H-02] Curve oracle manipulation with read only reentrancy in pool's `get_virtual_price`

## Impact

Curve Oracle manipulation will allow attacker to wrongfully liquididate users and profit from them.

## Proof of Concept

`ARBTriCryptoOracle` call to `get_virtual_price()` returns a deflated price when it is reentered after calling the Curve pool's `remove_liquidity` function. This allows an attacker to liquidate healthy accounts.

`get_virtual_price` gives the value of an LP token relative to the pool stable asset by dividing the total value of the pool by the `totalSupply()` of LP tokens:

```solidity

  @view
  @external
  def get_virtual_price() -> uint256:
      """
      @notice The current virtual price of the pool LP token
      @dev Useful for calculating profits
      @return LP token virtual price normalized to 1e18
      """
      D: uint256 = self.get_D(self._xp(self._stored_rates()), self._A())
      # D is in the units similar to DAI (e.g. converted to precision 1e18)
      # When balanced, D = n * x_u - total virtual value of the portfolio
      token_supply: uint256 = ERC20(self.lp_token).totalSupply()
      return D * PRECISION / token_supply

```

In the Curve pool function `remove_liquidity` when ETH is withdrawn, `raw_call()` allows the caller to reenter after the coin balances have been updated, but before the LP tokens are burned, so during the callback a reentrant call to `get_virtual_price()` will return a deflated value.

```solidity

  @external
  @nonreentrant('lock')
  def remove_liquidity(_amount: uint256, min_amounts: uint256[N_COINS]) -> uint256[N_COINS]:
      """
      @notice Withdraw coins from the pool
      @dev Withdrawal amounts are based on current deposit ratios
      @param _amount Quantity of LP tokens to burn in the withdrawal
      @param min_amounts Minimum amounts of underlying coins to receive
      @return List of amounts of coins that were withdrawn
      """
      _lp_token: address = self.lp_token
      total_supply: uint256 = ERC20(_lp_token).totalSupply()
      amounts: uint256[N_COINS] = empty(uint256[N_COINS])

      for i in range(N_COINS):
          _balance: uint256 = self.balances[i]
          value: uint256 = _balance * _amount / total_supply
          assert value >= min_amounts[i], "Withdrawal resulted in fewer coins than expected"
          self.balances[i] = _balance - value
          amounts[i] = value
          if i == 0:
              raw_call(msg.sender, b"", value=value)
          else:
              assert ERC20(self.coins[1]).transfer(msg.sender, value)

      CurveToken(_lp_token).burnFrom(msg.sender, _amount)  # Will raise if not enough

      log RemoveLiquidity(msg.sender, amounts, empty(uint256[N_COINS]), total_supply - _amount)

      return amounts

```

Attacker can use flash loan and reentrancy to carry out this attack. Checkout this References for more details:

This type of vulnerability has been reported here: https://chainsecurity.com/curve-lp-oracle-manipulation-post-mortem/

Post-mortem from MakerDAO here: https://forum.makerdao.com/t/curve-lp-token-oracle-manipulation-vulnerability-technical-postmortem/18009

```solidity

    function _get() internal view returns (uint256 _maxPrice) {
@->     uint256 _vp = TRI_CRYPTO.get_virtual_price();

        // Get the prices from chainlink and add 10 decimals
        uint256 _btcPrice = uint256(BTC_FEED.latestAnswer()) * 1e10; // @audit-issue Use of 
        uint256 _wbtcPrice = uint256(WBTC_FEED.latestAnswer()) * 1e10;
        uint256 _ethPrice = uint256(ETH_FEED.latestAnswer()) * 1e10;
        uint256 _usdtPrice = uint256(USDT_FEED.latestAnswer()) * 1e10;

        uint256 _minWbtcPrice = (_wbtcPrice < 1e18)
            ? (_wbtcPrice * _btcPrice) / 1e18
            : _btcPrice;

        uint256 _basePrices = (_minWbtcPrice * _ethPrice * _usdtPrice);

        _maxPrice = (3 * _vp * FixedPointMathLib.cbrt(_basePrices)) / 1 ether;

        // ((A/A0) * (gamma/gamma0)**2) ** (1/3)
        uint256 _g = (TRI_CRYPTO.gamma() * 1 ether) / GAMMA0;
        uint256 _a = (TRI_CRYPTO.A() * 1 ether) / A0;
        uint256 _discount = Math.max((_g ** 2 / 1 ether) * _a, 1e34); // handle qbrt nonconvergence
        // if discount is small, we take an upper bound
        _discount = (FixedPointMathLib.sqrt(_discount) * DISCOUNT0) / 1 ether;

        _maxPrice -= (_maxPrice * _discount) / 1 ether;
    }

```
[Link to code](https://github.com/Tapioca-DAO/tapioca-periph-audit/blob/a3b45512580f8a76be45c19f635689f48c0128c3/contracts/oracle/implementations/ARBTriCryptoOracle.sol#L118)

## Tools Used

VS Code

## Recommended Mitigation Steps

Add a reentrancy guard to protect against reentrancy.

### [H-03] Unprotected `delegatecall` allows stealing of funds from `BaseTOFTLeverageModule`

## Impact

Stealing of funds.

## Proof of Concept

There is a `public` function in `BaseTOFTLeverageModule` contract: `leverageDown`.

Issue here is there is:

* No Access control (public function).
* No Validation on `module` address.

```solidity

  function leverageDown(
        address module,
        uint16 _srcChainId,
        bytes memory _srcAddress,
        uint64 _nonce,
        bytes memory _payload
    ) public {
        //-----SNIP: Decode _payload----//

        uint256 balanceBefore = balanceOf(address(this));
        bool credited = creditedPackets[_srcChainId][_srcAddress][_nonce];
        if (!credited) {
            _creditTo(_srcChainId, address(this), amount);
            creditedPackets[_srcChainId][_srcAddress][_nonce] = true;
        }
        uint256 balanceAfter = balanceOf(address(this));

@->     (bool success, bytes memory reason) = module.delegatecall(
            abi.encodeWithSelector(
                this.leverageDownInternal.selector,
                amount,
                swapData,
                externalData,
                lzData,
                leverageFor
            )
        );

        if (!success) {
            if (balanceAfter - balanceBefore >= amount) {
                IERC20(address(this)).safeTransfer(leverageFor, amount);
            }
            revert(_getRevertMsg(reason)); //forward revert because it's handled by the main executor
        }

        emit ReceiveFromChain(_srcChainId, leverageFor, amount);
    }

```
[Link to Code](https://github.com/Tapioca-DAO/tapiocaz-audit/blob/master/contracts/tOFT/modules/BaseTOFTLeverageModule.sol#L184)

Leveraging these 2 issues, any attacker can use the following step to exploit the vulnerability:

1. An attacker can deploy their own malicious `module` contract with a `leverageDownInternal` function.
2. An attacker can call the `leverageDown` function, passing their malicious module's address.
3. The malicious `leverageDownInternal` function is designed to transfer funds from the `BaseTOFTLeverageModule` contract to the attacker's address.

This way the attacker's module can exploit the unprotected `delegatecall` to withdraw funds from the `BaseTOFTLeverageModule` contract without proper authorization.

## Tools Used

VS Code

## Recommended Mitigation Steps

Have an access control on the function and validate the `module` address parameter before performing any critical operations.

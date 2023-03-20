# Ethos

# QA Report

## Low Risk Issues
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [L-01] | `pragma experimental ABIEncoderV2` Used is deprecated | 1 |
| [L-02] | Upgradeable contract is missing a `__gap[50]` storage variable to allow for new storage variables in later versions | 1 |
| [L-03] | Use of `ecrecover` can lead to signature mallebility vulnerability | 1 |

| Total Low Risk Issues | 3 |
|:--:|:--:|

## Non-Critical Issues
| Count | Explanation | Instances |
|:--:|:-------|:--:|
| [N-01] | Function state mutability can be restricted to pure | 2 |
| [N-02] | Variable name should be in CamelCase | 1 |
| [N-03] | No Error Message provided for `require` | 3 |
| [N-04] | Spelling Errors in Natspec | 1 |
| [N-05] | Recommended to use 2 step while Updating Critical addresses | 2 |

| Total Non-Critical Issues | 5 |
|:--:|:--:|

### [L-01] `pragma experimental ABIEncoderV2` Used is deprecated

## Description

`pragma experimental ABIEncoderV2` Used is deprecated. Should use `pragma abicoder v2` instead which supports more types than v1 and performs more sanity checks on the inputs.

ABI coder v2 is activated by default in Solidity Version ^0.8.0. So it is already Enabled without explictly enabling it.

## Reference

* [Breaking Changes in Solidity ^0.8.0](https://github.com/ethereum/solidity/blob/69411436139acf5dbcfc5828446f18b9fcfee32c/docs/080-breaking-changes.rst#silent-changes-of-the-semantics)

## Recommendation Mitigation Step

Remove pragma experimental ABIEncoderV2 from the following code instance.

*Instance (1):*
```solidity
File: DistributionTypes.sol

2:    pragma experimental ABIEncoderV2;

```
[Link to code](https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/libraries/DistributionTypes.sol#L2)

### [L-02] Upgradeable contract is missing a `__gap[50]` storage variable to allow for new storage variables in later versions

See [this](https://docs.openzeppelin.com/contracts/4.x/upgradeable#storage_gaps) link for a description of this storage variable. While some contracts may not currently be sub-classed, adding the variable now protects against forgetting to add it in the future.

*Instance (1):*
```solidity
File: ReaperBaseStrategyv4.sol

14:   abstract contract ReaperBaseStrategyv4 is

```
[Link to code](https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L14-L17)

### [L-03] Use of `ecrecover` can lead to signature mallebility vulnerability

It is also recommended to use OpenZeppelin’s ECDSA library instead of ecrecover: [ECDSA.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/cryptography/ECDSA.sol)

Can check Latest `permit` method of Openzeppelin [here](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/ERC20Permit.sol).

*Instance (1):*
```solidity
File: LUSDToken.sol

    address recoveredAddress = ecrecover(digest, v, r, s);
    require(recoveredAddress == owner, 'LUSD: invalid signature');

```
[Link to Code](https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Core/contracts/LUSDToken.sol)

### [N-01] Function state mutability can be restricted to pure

Recommend to Change the state mutability of the following function to pure instead of view.

*Instances (2):*
```solidity
File: ReaperVaultV2.sol

659:    function _cascadingAccessRoles() internal view override returns (bytes32[] memory) {

```
[Link to Code](https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/ReaperVaultV2.sol#L659)

```solidity
File: ReaperBaseStrategyv4.sol

203:    function _cascadingAccessRoles() internal view override returns (bytes32[] memory) {

```
[Link to Code](https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L203)

### [N-02] Variable name should be in CamelCase

*Instance (1):*
```solidity
File: ReaperVaultV2.sol

473:    struct LocalVariables_report {

```
[Link to Code](https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/ReaperVaultV2.sol#L473)

### [N-03] No Error Message provided for `require`

Provide an error message for `require`.

*Instances (3):*
```solidity
File: ReaperStrategyGranarySupplyOnly.sol

167:    require(step[0] != address(0));
168:    require(step[1] != address(0));

```
[Link to Code](https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/ReaperStrategyGranarySupplyOnly.sol#L167-L168)

```solidity
File: VeloSolidMixin.sol

99:     require(
100:        _tokenIn != _tokenOut && _path.length >= 2 && _path[0] == _tokenIn && _path[_path.length - 1] == _tokenOut
101:    );

```
[Link to Code](https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/mixins/VeloSolidMixin.sol#L99-L101)

### [N-04] Spelling Errors in Natspec

*Instance (1):*

Correct the Spelling of `Ammount` to `Amount` and `Stricly` to `Strictly`.

```solidity
File: ReaperBaseStrategyv4.sol

98:     require(_amount <= balanceOf(), "Ammount must be less than balance");

203:    * {KEEPER} - Stricly permissioned trustless access for off-chain programs or third party keepers.

```
[Link to Code](https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L203)

### [N-05] Recommended to use 2 step while Updating Critical addresses

It is recommended to Use 2 Step ownership transfer for critical functions to avoid any foul circumstances. Here if governance address is set to a wrong address, `LUSDToken` can never be `unpause` from `pause` state.

*Instance (2):*
```solidity
File: LUSDToken.sol

146:    function updateGovernance(address _newGovernanceAddress) external {

153:    function updateGuardian(address _newGuardianAddress) external {

```
[Link to Code](https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Core/contracts/LUSDToken.sol#L146)

## Medium and High Risk Issues

| Count | Explanation |
|:--:|:-------|
| [M-1] | Oracle price can be Stale / Outdated used by the Protocol |
| [M-2] | `_withdraw` logic in `ReaperVaultV2` doesn't cover case of Strategy taking withdrawal fees |
| [M-3] | Multiplication after division in calculating Fees to be charged can lead to large precision loss |
| [H-1] | Classic First Depositor Attack Vulnerability in `ReaperVaultERC4626` |
| [H-2] | MEV Bot can make risk free profit by interacting with `ReaperVaultERC4626` |
| [H-3] | Attacker can exploit No Slippage check in swapping through Sandwich Attack |

| Issue Type | Count |
|:--:|:--:|
| Total Medium Risk Issues | 3 |
| Total High Risk Issues | 3 |


### [M-1] Oracle price used by the Protocol can be Stale / Outdated 

## Impact
There is lack of Validation of Oracle Price resulting in the Price can be Stale / Outdated which is eventually Used by the protocol.

## Proof of Concept

```solidity
File: PriceFeed.sol

    function _badChainlinkResponse(ChainlinkResponse memory _response) internal view returns (bool) {
         // Check for response call reverted
        if (!_response.success) {return true;}
        // Check for an invalid roundId that is 0
        if (_response.roundId == 0) {return true;}
        // Check for an invalid timeStamp that is 0, or in the future
        if (_response.timestamp == 0 || _response.timestamp > block.timestamp) {return true;}
        // Check for non-positive price
        if (_response.answer <= 0) {return true;}

        return false;
    }

```
[Link to Code](https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Core/contracts/PriceFeed.sol#L411-L422)

Chainlink oracle uses 2 triggers to update the Price:

1. If `Deviation Threshold` exceeds which means the Price has changed Drastically.
2. If `Heartbeat Threshold` exceeds. Heartbeat Threshold is a specific amount of time from the last update after which a new aggregation round will start and the price will be updated.

According to Chainlink documentation:

> If your application detects that the reported answer is not updated within the heartbeat or within time limits that you determine are acceptable for your application, pause operation or switch to an alternate operation mode while identifying the cause of the delay.


Link: https://docs.chain.link/data-feeds/#check-the-timestamp-of-the-latest-answer

Here, the checks for `_response.timestamp` are for zero value or greater value than current timestamp. But There is no check to find out whether the value provided by Chainlink is within the `Heartbeat Threshold` or not. If Not, the value will be a stale value which will be used by the Protocol.

## Tools Used

VS Code

## Recommended Mitigation Steps

Add a check as written below after adding a state variable for Heartbeat Threshold as suggested by chainlink:

```solidity
  if( block.timestamp - _response.timestamp > heartbeatThreshold) {return true;}
```

### [M-2] `_withdraw` logic in `ReaperVaultV2` doesn't cover case of Strategy taking withdrawal fees

## Impact

Some strategies charges few percent of the total amount as fees on withdrawal. But the logic of `_withdraw` method is such that it will have residual value of `strategies[stratAddr].allocated` and `totalAllocated` which can never be withdrawn.

## Proof of Concept

```solidity
File: ReaperVaultV2.sol

384:    uint256 remaining = value - vaultBalance;
385:    uint256 loss = IStrategy(stratAddr).withdraw(Math.min(remaining, strategyBal));
386:    uint256 actualWithdrawn = token.balanceOf(address(this)) - vaultBalance;

395:    strategies[stratAddr].allocated -= actualWithdrawn;
396:    totalAllocated -= actualWithdrawn;

```
[Link to code](https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/ReaperVaultV2.sol#L384-L396)

1. `IStrategy(stratAddr).withdraw` calls the withdraw method in strategy with lets say amount `x` [Let `x` be equal to `strategyBal` which means there is complete withdrawal from strategy].
2. It is possible that some strategy charges fees `f` on withdrawal.
3. `actualWithdrawn` calculated will be the actual withdrawn amount i.e. `actualWithdrawn` = `x` - `f`
4. While updating the state variable, `strategies[stratAddr].allocated` and `totalAllocated` are subtracted by `actualWithdrawn`.
5. Issue with Step 4 is that when there is a complete withdrawal from a particular strategy, the `strategies[stratAddr].allocated` should be zero. But here it will be left with [`x` - (`x` - `f`)] = `f` value which is already charged by strategy.

This result is residual value of `strategies[stratAddr].allocated` and `totalAllocated` which can never be withdrawn from the strategy as it is completely withdrawn.

## Tools Used

VS Code

## Recommended Mitigation Steps

Can add a state variable to track the fees charged by each strategies and deduct `amount + fees % of that amount` from the `strategies[stratAddr].allocated` and `totalAllocated`.

### [M-3] Multiplication after division in calculating Fees to be charged can lead to large precision loss

## Impact

There is a division before multiplication bug in `_chargeFees()` method of `ReaperVaultV2.sol` which may result in loss of fees being charged due to precision.

## Proof of Concept

```solidity
File: ReaperVaultV2.sol

463:    uint256 performanceFee = (gain * strategies[strategy].feeBPS) / PERCENT_DIVISOR;

466:    uint256 shares = supply == 0 ? performanceFee : (performanceFee * supply) / _freeFunds();

```
[Link to code](https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/ReaperVaultV2.sol#L463-L466)

In the above case, Division is taking place before multiplication.

First, to calculate `performanceFee`, product of `gain` and `strategies[strategy].feeBPS` is divided by `PERCENT_DIVISOR`. Same preicision loss happens here. Next, to calculate the `shares` if `supply` is not equal to zero which will be the case most often, then `performanceFee` is multiplied by `supply` which can result in larger precision loss.

## Tools Used

VS Code

## Recommended Mitigation Steps

Consider multiplying all the numerators first before dividing. 

Mitigated code:

```solidity
File: 

463:    uint256 performanceFee = (gain * strategies[strategy].feeBPS) / PERCENT_DIVISOR;

466:    uint256 shares = supply == 0 ? performanceFee : (gain * strategies[strategy].feeBPS * supply) / (PERCENT_DIVISOR * _freeFunds());

```
[Link to code](https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/ReaperVaultV2.sol#L463-L466)

### [H-1] Classic First Depositor Attack Vulnerability in `ReaperVaultERC4626`

## Description

Classic issue with vaults. First depositor can deposit a single wei then donate to the vault to greatly inflate share ratio. Due to truncation when converting to shares this can be used to steal funds from later depositors.

## Impact

Stealing of funds of other depositors.

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

## Tools Used

VS Code

## Recommended Mitigation Steps

1. Need to Enforce a minimum deposit that can't be withdrawn.
2. So, add some of the initial amount to the zero address.
3. Most legit first depositors will mint thousands of shares, so not a big cost.


### [H-2] MEV Bot can make risk free profit by interacting with `ReaperVaultERC4626`

## Impact

MEV Bot can make huge risk free profits which can lead to withdrawal of strategies and loss of funds for normal users.

## Proof of Concept

```solidity
File: ReaperVaultV2.sol

For Deposit:

330:    _amount = balAfter - balBefore;

334:    shares = (_amount * totalSupply()) / freeFunds;

336:    _mint(_receiver, shares);

For Withdrawal:

365:    value = (_freeFunds() * _shares) / totalSupply();

366:    _burn(_owner, _shares);

410:    token.safeTransfer(_receiver, value);

Free Fund method:

287:    function _freeFunds() internal view returns (uint256) {
288:        return balance() - _calculateLockedProfit();
289:    }

calculate Locked Profit method:

418:    function _calculateLockedProfit() internal view returns (uint256) {
419:        uint256 lockedFundsRatio = (block.timestamp - lastReport) * lockedProfitDegradation;
420:        if (lockedFundsRatio < DEGRADATION_COEFFICIENT) {
421:            return lockedProfit - ((lockedFundsRatio * lockedProfit) / DEGRADATION_COEFFICIENT);
422:        }
423:
424:        return 0;
425:    }

```
[Link to code](https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/ReaperVaultV2.sol#L418-L425)

To understand this attack, we need to understand how different values of methods throughout the lifecyle of locked profit affect the shares to be minted in deposit and value to be transafered in withdraw.

|                       | _calculateLockedProfit() | _freeFunds() | shares | value  |
|-----------------------|--------------------------|--------------|--------|--------|
| At start of the Cycle | x                        | b - x        | y      | z      |
| At end of the Cycle   | 0                        | b            | y - dy | z + dz |

Here: `x` is `lockedProfit`, `b` is `balance of token`, `y` is the shares that can be minted, `z` is the `value` that can be transferred initially and `dy` & `dz` be the change throughout a complete lifecycle of Locked Profit.

Here's how Bot can earn risk free:
1. It keeps the track of new funds getting added as profit through `report`.
2. It can add deposit a large amount [Let's say `z`] as soon as profit is added in locked state.
3. As it's start of that Locked Profit cycle, shares minted to Bot will be `y`.
4. Once that cycle comes to an end, Bot can withdraw `z + dz` amount with less than `y` shares (`y` - `dy` to be precise).

This way Bot can earn without taking any risk

## Tools Used

VS Code

## Recommended Mitigation Steps

It is recommended to not use dynamic predictative value of `_freeFund()` in the logic to calculate `share` or `value` to be deposited or withdrawn.

### [H-3] Attacker can exploit No Slippage check in swapping through Sandwich Attack

## Impact

Loss of Funds due to sandwich attack by attacker.

## Proof of Concept

```solidity
File: VeloSolidMixin.sol

41:     router.swapExactTokensForTokens(_amount, 0, routes, address(this), block.timestamp);

```
[Link to Code](https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/mixins/VeloSolidMixin.sol#L41)

There is a call to `_swapVelo` from the `_harvestCore` which uses `swapExactTokensForTokens` call to swap the token.

The arguments it is taking are:

```solidity

    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        route[] calldata routes,
        address to,
        uint deadline
    )

```

Here, the second argument `amountOutMin` is there to protect the caller against the slippage. In case the `amountOut` value is less than the `amountOutMin`, the transaction reverts.

But in the `_swapVelo` method, `0` is passed for `amountOutMin` which means any MEV Bot or any Attacker can sandwich attack the transaction first by front running and then back running to take maximum advantage of slippage. It will lead to loss of funds to caller which is strategy in our case which is indirectly uses the User Funds.

## Tools Used

VS Code

## Recommended Mitigation Steps

Use the Max Slippage value as a parameter rather than zero.

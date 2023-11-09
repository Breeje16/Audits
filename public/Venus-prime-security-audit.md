# Analysis

## Summary

| Count | Topic | 
|:--:|:-------|
| 1 | Introduction |  
| 2 | High Level Contract Flow |  
| 3 | Audit Approach |  
| 4 | Codebase Quality |   
| 5 | Architecture Recommendation |  
| 6 | Centralization Risk |
| 7 | Systemic Risks |
| 8 | Code Review Insights |
| 9 | Time Spent |

## Introduction

This technical analysis report examines the security aspects of Venus Prime, a vital component of the Venus Protocol's incentive program. The analysis explores the underlying smart contracts and different components, including reward distribution calculations, claiming mechanism, and unique Soulbound Tokens, while identifying security insights and potential vulnerabilities within the Venus Prime contracts.

## High Level Contract Flow

![High Level Contract Flow](https://res.cloudinary.com/davyfibzy/image/upload/v1696261747/nzz6hhgqd3edr6l3fslz.png)

## Audit Approach

1. **Initial Scope and Documentation Review**: Thoroughly examined the Contest Readme File and Venus Prime Readme file to understand the contract's objectives and functionalities.
2. **High-level Architecture Understanding**: Performed an initial architecture review of the codebase by skimming through all files without exploring function details.
3. **Test Environment Setup and Analysis**: Set up a test environment and executed all tests.
4. **Comprehensive Code Review**: Conducted a line-by-line code review focusing on understanding code functionalities. 
    * **Understand Codebase Functionalities**: Began by comprehending the functionalities of the codebase to gain a clear understanding of its operations.
    * **Analyze Value Transfer Functions**: Adopted an attacker's mindset while inspecting value transfer functions, aiming to identify potential vulnerabilities that could be exploited.
    * **Identify Access Control Issues**: Thoroughly examined the codebase for any access control problems that might allow unauthorized users to execute critical functions.
    * **Detect Reentrancy Vulnerabilities**: Looked for potential reentrancy attacks where malicious contracts could exploit reentrant calls to manipulate the contract's state and behavior.
    * **Evaluate Function Execution Order**: Examined the sequence of function executions to ensure the protocol's logic cannot be disrupted by changing the order of calls.
    * **Assess State Variable Handling**: Identified state variables and assessed the possibility of breaking assumptions to manipulate them into exploitable states, leading to unintended exploitation of the protocol's functionality.

5. **Report Writing**: Write Report by compiling all the insights I gained throughout the line by line code review.


## Codebase Quality

| Codebase Quality Categories  | Comments |
| --- | --- |
| *Unit Testing*  | The codebase has undergone extensive testing. |
| *Code Comments*  | In general, the code had comments for what each function does, which is great. However, it could be better if we also added comments for the custom data types (struct) used in the code. This would help make it clearer how each data type is used in the contract. |
| *Documentation* | Prime Documentation was to the point, containing all the major details about how the contract is expected to perform. |
| *Organization* | Code orgranization was great for sure. The flow from one contract was perfectly smooth. |

## Architecture Recommendation

* `Block.number Vs Block.timestamp`: Since the contracts are planned for deployment on multiple chains, including those like Optimism where `block.number` may not accurately represent time, it is advisable to use `block.timestamp` consistently. In the `PrimeLiquidityProvider` (PLP) contract, variations in average block time can lead to fluctuations in the rate at which tokens accrue.

* `Solidity Version Used`: Consider upgrading the Solidity version from 0.8.13 to 0.8.15 or a higher version. Although the current version does not impact any contracts within scope, it's a important to avoid any potential issues which 0.8.13 version brings related to ABI-encoding and optimizer bugs regarding memory side effects of inline assembly.

* `Handling Multiple Markets with Same Underlying Token`: Address the limitation in the Prime contract, which currently cannot handle multiple markets with the same underlying token. This issue could have long-term implications, especially if multiple markets with identical underlying tokens need to be added to the `Prime` program simultaneously. It's recommended to enhance the contract's ability to handle such scenarios for improved flexibility and scalability.

## Centralization Risk

* `Token Sweep Power`: The owner has the authority to sweep all tokens held in the PrimeLiquidityProvider contract. This centralized control could potentially pose serious risk.

* `Claim Functionality Pause`: The owner retains the capability to temporarily "pause" the claim functionality whenever necessary. This centralized ability to halt operations could affect user access and functionality.

* `Privileged Role Impact`: Privileged roles have the ability to modify the values of `alpha` and `multiplier`, which can have a substantial impact on the interest earned by users.

## Systemic risks

* `Sybil Risk`: The contract employs a cap named `MAXIMUM_XVS_CAP` to prevent users from earning interest on amounts exceeding this limit. However, in practice, a user can potentially exploit this by using multiple accounts to accumulate more interest. As a result, the existing protection measures in place are not entirely immune to Sybil attacks.

* `DoS in Key Functionities`: `sweepToken` function is intended to recover accidentally transferred ERC-20 tokens to the contract. While this function is restricted to the contract owner (`onlyOwner` modifier), it does not account for previously accrued tokens, which poses a significant risk to the protocol's stability. In case it sweeps the token which are previously added to total accrued tokens but not released to Prime yet, then it will lead to DoS across both this contracts.

* `Gas Limit Issues`: There is no way to update Max Loops value in Future. While `addMarket` controls the number of markets that can be add so it not that big a risk. But still as there is no way to even remove the market, it is a possible risk if block limit is reached.

## Code Review Insights

**Key Contracts:**

1. `Prime.sol`: Soulbound token, the central contract of this audit, allows Prime holders to accumulate rewards, which they can later claim at their convenience.

2. `PrimeLiquidityProvider.sol`: This contract offers liquidity to the Prime token consistently over a specific duration. This liquidity serves as one of the two fund sources supported by Prime tokens.

### Insights

* The rate at which tokens accumulate in `PrimeLiquidityProvider` may not be consistent across different blockchain networks.

* The `sweepToken` function has the potential to sweep already accrued rewards, which could result in Denial-of-Service (DoS) vulnerabilities and financial losses for users.

* There is a risk of encountering an infinite loop within the `updateScores` function.

* The value of `BLOCKS_PER_YEAR` may be unpredictable during deployment, particularly on blockchain networks with inconsistent block times.

* Tokens that have fees associated with them may lead to exaggerated interest claims for some users, potentially causing issues for the remaining users.

* Irrevocable tokens can be burned.

* There is currently no mechanism in place to update the maximum loop count in the future.

* The `issue` function allows the issuance of Prime Tokens to users without enforcing the 90-day staking condition.

* The `Prime` contract does not support the use of multiple markets with the same underlying tokens.

## Time Spent

| Total Number of Hours | 16 |
|:--:|:--:|

-----------------------------------------------------------

# QA Report

## Low Risk Issues
| Count | Explanation | 
|:--:|:-------|
| [L-01] | Solidity Version 0.8.13 used is vulnerable to optimizer issues |
| [L-02] | No functionality to change `maxLoopsLimit` can be problematic in long term |  
| [L-03] | `issue` doesn't check the 90 days staking condition to issue Prime Token |  
| [L-04] | Adding Multiple Markets with same underlying Tokens can be problematic |

| Total Low Risk Issues | 4 |
|:--:|:--:|

### [L-01] Solidity Version 0.8.13 used is vulnerable to optimizer issues

The solidity version 0.8.13 has below two issues:

1. Vulnerability related to ABI-encoding. ref : https://blog.soliditylang.org/2022/05/18/solidity-0.8.14-release-announcement/
2. Vulnerability related to 'Optimizer Bug Regarding Memory Side Effects of Inline Assembly' ref : https://blog.soliditylang.org/2022/06/15/solidity-0.8.15-release-announcement/

Use recent Solidity version or atleast above 0.8.15 version which has the fix for these issues.

### [L-02] No functionality to change `maxLoopsLimit` can be problematic in long term

In `initialize` function of `Prime.sol` contract, `maxLoopsLimit` is set to `_loopsLimit` value through calling function `_setMaxLoopsLimit`.

```solidity
File: Prime.sol

`initialize` function:

164:    _setMaxLoopsLimit(_loopsLimit);

```

Issue here is that `_setMaxLoopsLimit` function is `internal` access function which cannot be called from outside and there is no other method in the contract which uses this function to change the value of `maxLoopsLimit` again in future to make sure no DoS issues happens.

Not having this access can lead to issues in future. So it is recommended to add a function that allows priviledged account to update this value.

### [L-03] `issue` doesn't check the 90 days staking condition to issue Prime Token

`issue` is a priviledged function used to Directly issue prime tokens to users.

But in implementation, the condition is not checked to make sure that user has staked the xvs tokens for 90 Days. This can lead to issuing of tokens even if user fails to satisfy the condition required to get the token.

```solidity
File: Prime.sol

  function issue(bool isIrrevocable, address[] calldata users) external {

    ***

            for (uint256 i = 0; i < users.length; ) {
                _mint(false, users[i]);
                _initializeMarkets(users[i]);
                delete stakedAt[users[i]]; // @audit no check for 90 Days staking condition.

                unchecked {
                    i++;
                }
     ***
    }

```

### [L-4] Adding Multiple Markets with same underlying Tokens can be problematic

`addMarket` function in `Prime` contract is used to Add a market to prime program.

Also there is a possibility that multiple markets have same underlying tokens.

If case it happens, the line below will overwrite any existing market.

```solidity
File: Prime.sol

    function addMarket(address vToken, uint256 supplyMultiplier, uint256 borrowMultiplier) external {

      ***

@->   vTokenForAsset[_getUnderlying(vToken)] = vToken;

      ***

    }

```

This can lead to issues as there will now be 2 markets in existance (`markets[vToken].exists = true`) with same underlying token but `vTokenForAsset` will only point to new market added.

---------------------------------------------------------

## Medium and High Risk Issues
| Count | Explanation |
|:--:|:-------|
| [M-1] | Rate of Accured Tokens in `PrimeLiquidityProvider` can be skewed on some Chains |
| [M-2] | `calculateAPR` and `estimateAPR` will give wrong calculation on some chains |
| [M-3] | Tokens with fees can lead to inflated interest claims for some users while leaving the last few with DoSed |
| [M-4] | `updateScores` function has possible infinite Loop because of incorrect implementation |
| [M-5] | `sweepToken` can sweep already Accrued reward which can lead to DoS Issues & Loss of funds for Users |
| [H-1] | Incorrect Decimals Precision leading to incorrect calculation of Score |

| Issue Type | Count |
|:--:|:--:|
| Total Medium Risk Issues | 5 |
| Total High Risk Issues | 1 |

### [M-01] Rate of Accrued Tokens in `PrimeLiquidityProvider` can be skewed on some Chains

## Impact

Inaccuracy in the way Tokens are accrued leading to lower tokens accrued than intended.

## Proof of Concept

`accrueTokens` function in `PrimeLiquidityProvider` works as the following:

Eg: 1000 XTokens have been added to the contract with speed 100 tokens per block, then 100 tokens will be accrued per block which Prime contract can redeem by calling `releaseFunds`.

```solidity
File: PrimeLiquidityProvider.sol

  function accrueTokens(address token_) public {
      _ensureZeroAddress(token_);

      _ensureTokenInitialized(token_);

      uint256 blockNumber = getBlockNumber();
      uint256 deltaBlocks = blockNumber - lastAccruedBlock[token_];

      if (deltaBlocks > 0) {
          uint256 distributionSpeed = tokenDistributionSpeeds[token_];
          uint256 balance = IERC20Upgradeable(token_).balanceOf(address(this));

          uint256 balanceDiff = balance - tokenAmountAccrued[token_];
          if (distributionSpeed > 0 && balanceDiff > 0) {
              uint256 accruedSinceUpdate = deltaBlocks * distributionSpeed;
              uint256 tokenAccrued = (balanceDiff <= accruedSinceUpdate ? balanceDiff : accruedSinceUpdate);

              tokenAmountAccrued[token_] += tokenAccrued;
              emit TokensAccrued(token_, tokenAccrued);
          }

          lastAccruedBlock[token_] = blockNumber;
      }
  }

```
[Link to code](https://github.com/code-423n4/2023-09-venus/blob/main/contracts/Tokens/Prime/PrimeLiquidityProvider.sol#L249)

Graph looks like:

![Accured Tokens Vs Block Number](https://res.cloudinary.com/davyfibzy/image/upload/v1696265391/nl735rabkoir11mjscnx.png)

But, As per the contest Readme:

> Blockchains where this code will be deployed: BNB Chain, Ethereum mainnet, Arbitrum, Polygon zkEVM, opBNB.

On Optimism, the `block.number` is not a reliable source of timing information and the time between each block is also different from Ethereum. This is because each transaction on L2 is placed in a separate block and blocks are not produce at a constant rate. This will cause the accuring token calculation using `accrueTokens()` to fluctuate. (see Optimism [docs](https://community.optimism.io/docs/developers/build/differences/#block-numbers-and-timestamps))

On BNB Chain as well, there are fluctuation:

Eg: Between 10th May to 19th May 2023, There are 5 instances where it took close to 3,000 Seconds for a block to mine instead of average of 3 seconds.

Link: https://bscscan.com/chart/blocktime

This inaccuracies related to `block.number` can lead to Rate of Accured Tokens in `PrimeLiquidityProvider` not work as intended. The Graph of accrue tokens Vs Time won't be straightline in such scenario.

## Tools Used

VS Code

## Recommended Mitigation Steps

I would recommend to use `block.timestamp` instead of `block.number` in this contract which will be fair way of accruing tokens wrt time.

### [M-02] `calculateAPR` and `estimateAPR` will give wrong calculation on some chains

## Impact

`calculateAPR` and `estimateAPR` will give wrong calculation on some chains.

## Proof of Concept

`calculateAPR` and `estimateAPR` calls `_calculateUserAPR` function and returns the value returned from that function.

`_calculateUserAPR` calls `_incomeDistributionYearly` to get the total income that's going to be distributed in a year to prime token holders.

```solidity
File: Prime.sol

  function _incomeDistributionYearly(address vToken) internal view returns (uint256 amount) {
      uint256 totalIncomePerBlockFromMarket = _incomePerBlock(vToken);
      uint256 incomePerBlockForDistributionFromMarket = (totalIncomePerBlockFromMarket * _distributionPercentage()) /
          IProtocolShareReserve(protocolShareReserve).MAX_PERCENT();
@->   amount = BLOCKS_PER_YEAR * incomePerBlockForDistributionFromMarket;

      uint256 totalIncomePerBlockFromPLP = IPrimeLiquidityProvider(primeLiquidityProvider)
          .getEffectiveDistributionSpeed(_getUnderlying(vToken));
@->   amount += BLOCKS_PER_YEAR * totalIncomePerBlockFromPLP;
  }

```
[Link to Code](https://github.com/code-423n4/2023-09-venus/blob/main/contracts/Tokens/Prime/Prime.sol#L974)

What this function does is, it calculates the income per block from Prime Liquidity Provider and Income from the Market and just multiply it with `BLOCKS_PER_YEAR`.

As per the contest Readme:

> Blockchains where this code will be deployed: BNB Chain, Ethereum mainnet, Arbitrum, Polygon zkEVM, opBNB.

As it is getting deployed in multiple chains, including chains where `block.timestamp` is not correlated to `block.number`, this will lead to the value of `BLOCKS_PER_YEAR` impossible to predict and initialize it correctly at the time of deployment. This will lead to inaccuracy in the calculation of `calculateAPR` and `estimateAPR`.

Eg: On Optimism, the `block.number` is not a reliable source of timing information and the time between each block is also different from Ethereum. This is because each transaction on L2 is placed in a separate block and blocks are not produce at a constant rate. This will cause the calculation using `calculateAPR()` & `estimateAPR()` to be incorrect. (see Optimism [docs](https://community.optimism.io/docs/developers/build/differences/#block-numbers-and-timestamps))

## Tools Used

VS Code

## Recommended Mitigation Steps

Use `block.timestamp` instead for calculations.

### [M-03] Tokens with fees can lead to inflated interest claims for some users while leaving the last few with DoSed

## Impact

Incorrectly permitting higher interest claims for early claimers and potentially causing denial-of-service (DoS) issues for those claiming later

## Proof of Concept

As per Contest Readme:

> focusing on markets including USDT, USDC, BTC and ETH.

It is possible that the reward token can be `USDC` or `USDT` which are upgradable contracts in most of the chains and can charge fees on transfer in future.

Now Consider the following scenario:

1. `USDT` starts charging `1%` fees on every transfer.
2. While accruing, the `tokenAmountAccrued` of PLP will give let's say `x` amount and reward for that market will be added as per that value.

```solidity
File: Prime.sol

  function accrueInterest(address vToken) public {
      ***

@->   uint256 totalIncomeUnreleased = IProtocolShareReserve(protocolShareReserve).getUnreleasedFunds(
          comptroller,
          IProtocolShareReserve.Schema.SPREAD_PRIME_CORE,
          address(this),
          underlying
      );

      uint256 distributionIncome = totalIncomeUnreleased - unreleasedPSRIncome[underlying];

      _primeLiquidityProvider.accrueTokens(underlying);
@->   uint256 totalAccruedInPLP = _primeLiquidityProvider.tokenAmountAccrued(underlying);
      uint256 unreleasedPLPAccruedInterest = totalAccruedInPLP - unreleasedPLPIncome[underlying];

      distributionIncome += unreleasedPLPAccruedInterest;
      
      ***

      unreleasedPSRIncome[underlying] = totalIncomeUnreleased;
      unreleasedPLPIncome[underlying] = totalAccruedInPLP;

      uint256 delta;
      if (markets[vToken].sumOfMembersScore > 0) {
          delta = ((distributionIncome * EXP_SCALE) / markets[vToken].sumOfMembersScore);
      }

@->   markets[vToken].rewardIndex = markets[vToken].rewardIndex + delta;
  }

```

3. But when User will claim the reward and when Prime contract will call `releaseFunds` from PLP, it will transfer only 99% of `x`. Same will be true for Protocol Share Reserve as well.

4. This situation will lead to an issue where initial users will be able to claim inflated rewards which includes that extra `1%` while those who claims last will be denied any reward as contract won't have any left because of giving out more rewards than required to others.

## Tools Used

VS Code

## Recommended Mitigation Steps

Having an extra variable for `Fees` whose value can be added in future and subtracting that value from `totalAccruedInPLP` and `totalIncomeUnreleased` in `accrueInterest` can solve this issue.

### [M-04] `updateScores` function has possible infinite Loop because of incorrect implementation

## Impact

Frequent Loss of High Gas costs for Protocol due to infinite Loop

## Proof of Concept

As per the Readme Doc inside Prime Folder:

> Market multipliers and alpha can be updated at anytime and need to be propagated to all users. Changes will be gradually applied to users as they borrow/supply assets and their individual scores are recalculated. This strategy has limitations because the scores will be wrong in aggregate.

> To mitigate this issue, Venus will supply a script that will use the permission-less function updateScores to update the scores of all users. This script won’t pause the market or Prime contract. Scores need to be updated in multiple transactions because it will run out of gas trying to update all scores in 1 transaction.

Venus is going to use a script to call `updateScores` to update scores for all the users.

Consider the following Situation now:

1. Before running the script, Alice calls the `updateScores` function to update her score for round `x`.

```solidity
File: Prime.sol

    function updateScores(address[] memory users) external {
        ***

        for (uint256 i = 0; i < users.length; ) {
            address user = users[i];

            if (!tokens[user].exists) revert UserHasNoPrimeToken();
 @->        if (isScoreUpdated[nextScoreUpdateRoundId][user]) continue; // @audit Infinite Loop

            ***

 @->        isScoreUpdated[nextScoreUpdateRoundId][user] = true;

            unchecked {
 @->            i++;
            }

            emit UserScoreUpdated(user);
        }
    }

```
[Link to code](https://github.com/code-423n4/2023-09-venus/blob/main/contracts/Tokens/Prime/Prime.sol#L208)

2. After Alice's call, her `isScoreUpdated` status is set to `true`.
3. When the script runs, it checks if a user's score has already been updated with the line: `if (isScoreUpdated[nextScoreUpdateRoundId][user]) continue;`. The idea is not to update scores again if they've already been updated.
4. However, there's an issue here. To optimize gas usage, `i++` is used with the `unchecked` keyword at the end. But because of the `continue` keyword, `i++` is effectively skipped, resulting in the same user being processed repeatedly, creating an infinite loop.

In simple terms, the code tries to be efficient with gas usage but makes a mistake. Instead of increasing the counter `i` within the `for` loop itself, it's done later with `unchecked` to save gas. However, it ends up getting stuck in an infinite loop for some users like Alice.

## Tools Used

VS Code

## Recommended Mitigation Steps

Update the code as following:

```diff
File: Prime.sol

    function updateScores(address[] memory users) external {
        ***

        for (uint256 i = 0; i < users.length; ) {
            address user = users[i];

            if (!tokens[user].exists) revert UserHasNoPrimeToken();
-           if (isScoreUpdated[nextScoreUpdateRoundId][user]) continue;
+           if (isScoreUpdated[nextScoreUpdateRoundId][user]) {
+               unchecked {
+                   i++;
+               }
+               continue;
+           }

            ***

            unchecked {
                i++;
            }

            emit UserScoreUpdated(user);
        }
    }
```

### [M-05] `sweepToken` can sweep already Accrued reward which can lead to DoS Issues & Loss of funds for Users

## Impact

This issue could potentially result in a DoS scenario for many functions within the `Prime` contract, which relies on the `accrueInterest` function. Consequently, users may be unable to claim any interest.

## Proof of Concept

In the `PrimeLiquidityProvider` contract, there exists a `sweepToken` function intended to recover accidentally transferred ERC-20 tokens to the contract. While this function is restricted to the contract owner (`onlyOwner` modifier), it does not account for previously accrued tokens, which poses a significant risk to the protocol's stability.

```solidity
File: PrimeLiquidityProvider.sol

    function sweepToken(IERC20Upgradeable token_, address to_, uint256 amount_) external onlyOwner {
        uint256 balance = token_.balanceOf(address(this));
        if (amount_ > balance) {
            revert InsufficientBalance(amount_, balance);
        }

        emit SweepToken(address(token_), to_, amount_);

        token_.safeTransfer(to_, amount_);
    }

```
[Link to code](https://github.com/code-423n4/2023-09-venus/blob/main/contracts/Tokens/Prime/PrimeLiquidityProvider.sol#L216)

Here's a scenario to illustrate the issue:

1. Let's assume there is an accidental transfer of `x` amount of some reward token.
2. The `tokenAmountAccrued` keeps increasing with each block.
3. The `Prime` contract relies on `tokenAmountAccrued` to calculate the interest claimable by users.
4. If these tokens are removed via `sweepToken`, it creates an inconsistency because `tokenAmountAccrued` exceeds the contract's balance.
5. This leads to a DoS in the `accrueTokens` function due to underflow:

```solidity
211:    uint256 balanceDiff = balance - tokenAmountAccrued[token_];
```

6. It also results in a DoS in the `releaseFunds` function within PLP (PrimeLiquidityProvider) and in the `accrueInterest` function in the `Prime` contract. Consequently, this blocks most critical functions, including `claimInterest`.

While this issue is dependent on the owner making this mistake, it highlights an implementation flaw in handling states. If the intended purpose of this function is solely to recover accidentally transferred ERC-20 tokens, it is recommended to implement the proposed mitigation code to prevent the smart contract from reaching a state where this problem can occur.

Given the severe consequences and the potential of incorrect implementation (allowing the contract to reach a problematic state), a Medium Severity classification should be justified, similar to a previous classification for a related issue [here](https://code4rena.com/reports/2023-05-maia#m-14-boostaggregator-owner-can-set-fees-to-100-and-steal-all-of-the-users-rewards).

## Tools Used

VS Code

## Recommended Mitigation Steps

```diff

    function sweepToken(IERC20Upgradeable token_, address to_, uint256 amount_) external onlyOwner {
        uint256 balance = token_.balanceOf(address(this));
-       if (amount_ > balance) {
+       if (amount_ > balance - tokenAmountAccrued[token_]) {
+           revert InsufficientBalance(amount_, balance - tokenAmountAccrued[token_]);
-           revert InsufficientBalance(amount_, balance);
        }

        emit SweepToken(address(token_), to_, amount_);

        token_.safeTransfer(to_, amount_);
    }

```


### [H-01] Incorrect Decimals Precision leading to incorrect calculation of Score

## Impact

Scores are getting incorrectly calculated.

## Proof of Concept

Scores are the critical part of `Prime` codebase as Scores are directly correlated to the interest users can claim.

As per the Natspec in `Scores.sol`:

```solidity
14:   * @param xvs amount of xvs (xvs, 1e18 decimal places)
15:   * @param capital amount of capital (1e18 decimal places)
```

The `xvs` and `capital` passed in `calculateScore` function must have 18 decimal places.

Now let's focus on `_calculateScore` function in `Prime.sol`. This internal function calculates the score based on `market` and `user` and returns the actual `score`.

```solidity

    function _calculateScore(address market, address user) internal returns (uint256) {
        uint256 xvsBalanceForScore = _xvsBalanceForScore(_xvsBalanceOfUser(user));

        IVToken vToken = IVToken(market);
        uint256 borrow = vToken.borrowBalanceStored(user); 
        uint256 exchangeRate = vToken.exchangeRateStored(); 
        uint256 balanceOfAccount = vToken.balanceOf(user);
        uint256 supply = (exchangeRate * balanceOfAccount) / EXP_SCALE;

        address xvsToken = IXVSVault(xvsVault).xvsAddress();
        oracle.updateAssetPrice(xvsToken);
        oracle.updatePrice(market);

        (uint256 capital, , ) = _capitalForScore(xvsBalanceForScore, borrow, supply, market);
        capital = capital * (10 ** (18 - vToken.decimals()));

        return Scores.calculateScore(xvsBalanceForScore, capital, alphaNumerator, alphaDenominator);
    }

```

Now let's understand decimals precision of each variable in this function:

1. `xvsBalanceForScore` has the decimals of xvs token which is 18 decimals.

2. `borrow` has the number of decimals of the underlying token (for example 18 for USDT and 6 for TRX).

3. `balanceOfAccount` has the number of decimals of the VTokens (every VToken in Venus has 8 decimals now)

4. `exchangeRate` has the decimals of the underlying token + 10 (for example, 28 for USDT and 16 for TRX).

5. `supply` which is `(exchangeRate * balanceOfAccount) / EXP_SCALE` will have always the decimals of the underlying token.

Now there is a call on `_capitalForScore` to calculate the capital required for calculation of score.

```solidity

  function _capitalForScore(
        uint256 xvs, 
        uint256 borrow,
        uint256 supply, 
        address market
    ) internal view returns (uint256, uint256, uint256) {
        address xvsToken = IXVSVault(xvsVault).xvsAddress();

        uint256 xvsPrice = oracle.getPrice(xvsToken)
        uint256 borrowCapUSD = (xvsPrice * ((xvs * markets[market].borrowMultiplier) / EXP_SCALE)) / EXP_SCALE;
        uint256 supplyCapUSD = (xvsPrice * ((xvs * markets[market].supplyMultiplier) / EXP_SCALE)) / EXP_SCALE;

        uint256 tokenPrice = oracle.getUnderlyingPrice(market); 
        uint256 supplyUSD = (tokenPrice * supply) / EXP_SCALE
        uint256 borrowUSD = (tokenPrice * borrow) / EXP_SCALE;

        if (supplyUSD >= supplyCapUSD) {
            supply = supplyUSD > 0 ? (supply * supplyCapUSD) / supplyUSD : 0;
        }

        if (borrowUSD >= borrowCapUSD) {
            borrow = borrowUSD > 0 ? (borrow * borrowCapUSD) / borrowUSD : 0;
        }

        return ((supply + borrow), supply, borrow);
    }

```

Here, the response of the `ResilientOracle` has 36 - the decimals of the underlying token (for example 18 for USDT and 30 for TRX). This way `(tokenPrice * supply) / EXP_SCALE` will have always 18 decimals (same for the borrow).

As `supply` and `borrow` both have same decimals as underlying token, the return value of `capital` will also have same decimals of underlying token.

Now focus on this last 3 lines of `_calculateScore`:

```solidity

      (uint256 capital, , ) = _capitalForScore(xvsBalanceForScore, borrow, supply, market);
@->   capital = capital * (10 ** (18 - vToken.decimals()));

      return Scores.calculateScore(xvsBalanceForScore, capital, alphaNumerator, alphaDenominator);

```

In line above (@->), capital which has decimals of underlying token is multiplied by:

- 10 ** (18 - vToken.decimals())
- 10 ** (18 - 8) [As vToken.decimals() = 8]
- 10 ** 10

As shown earlier, `capital` passed in ` Scores.calculateScore` must have 18 decimals But the decimal of capital passed will be:

For underlying tokens having 18 Decimals (USDT): 18 + 10 = 28 Decimals.
For underlying tokens having 6 Decimals (TRX): 6 + 10 = 16 Decimals.

This means the value of `xvs` will have 18 decimal precision but `capital` will have precision ranging from 16 to 28 decimals depending on underlying token.

This will lead to incorrect calculation of Scores.

## Tools Used

VS Code

## Recommended Mitigation Steps

Update the codebase in such a way that you subtract decimal of underlying token instead of `vToken.decimals()`.

```diff

    function _calculateScore(address market, address user) internal returns (uint256) {
        uint256 xvsBalanceForScore = _xvsBalanceForScore(_xvsBalanceOfUser(user));

        IVToken vToken = IVToken(market);
        uint256 borrow = vToken.borrowBalanceStored(user); 
        uint256 exchangeRate = vToken.exchangeRateStored(); 
        uint256 balanceOfAccount = vToken.balanceOf(user);
        uint256 supply = (exchangeRate * balanceOfAccount) / EXP_SCALE;

        address xvsToken = IXVSVault(xvsVault).xvsAddress();
        oracle.updateAssetPrice(xvsToken);
        oracle.updatePrice(market);

        (uint256 capital, , ) = _capitalForScore(xvsBalanceForScore, borrow, supply, market);
-       capital = capital * (10 ** (18 - vToken.decimals()));
+       capital = capital * (10 ** (18 - decimalOfUnderlyingToken));

        return Scores.calculateScore(xvsBalanceForScore, capital, alphaNumerator, alphaDenominator);
    }

```

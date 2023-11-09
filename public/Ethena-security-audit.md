# Analysis

## Summary

| Count | Topic | 
|:--:|:-------|
| 1 | Introduction |  
| 2 | Audit Approach |  
| 3 | Architecture Recommendation |  
| 4 | Centralization Risk |   
| 5 | Systemic risks |
| 6 | Time Spent |

## Introduction

This technical analysis report dig into the underlying smart contracts that constitute the stablecoin and it's staking contracts of Ethena. This report highlights insights and different Risks that User and Protocol must be aware of.

## Audit Approach

1. **Initial Scope and Documentation Review**: Thoroughly examine the Contest Readme File and project documentation to understand the protocol's objectives and functionalities.
2. **High-level Architecture Understanding**: Perform an initial architecture review of the codebase by skimming through all files without going into function details.
3. **Test Environment Setup and Analysis**: Set up a test environment and execute all tests. Additionally, use Static Analyzer tools like Slither to identify potential vulnerabilities.
4. **Comprehensive Code Review**: Conduct a line-by-line code review focusing on understanding code functionalities. 
    * **Understand Codebase Functionalities**: Begin by comprehending the functionalities of the codebase to gain a clear understanding of its operations.
    * **Analyze Value Transfer Functions**: Adopt an attacker's mindset while inspecting value transfer functions, aiming to identify potential vulnerabilities that could be exploited.
    * **Identify Access Control Issues**: Thoroughly examine the codebase for any access control problems that might allow unauthorized users to execute critical functions.
    * **Detect Reentrancy Vulnerabilities**: Look for potential reentrancy attacks where malicious contracts could exploit reentrant calls to manipulate the contract's state and behavior.
    * **Evaluate Function Execution Order**: Scrutinize the sequence of function executions to ensure the protocol's logic cannot be disrupted by changing the order of calls.
    * **Assess State Variable Handling**: Identify state variables and assess the possibility of breaking assumptions to manipulate them into exploitable states, leading to unintended exploitation of the protocol's functionality.

5. **Report Writing**: Write Report by compiling all the insights I gained throughout the line by line code review.
    

## Architecture Recommendation

`Handling of stETH can be improved`: As `stETH` is a rebasing token, it can lead to couple of issues in the `EthenaMinting` contract like:

1. `Slippage Due to Rebase Events`: Slippage in value because of rebasing when transaction is pending.
2. `Underlying Value Mismatch`: Value received by custodian addresses will always be less than intended which can break the logics used further.

I have describe about these 2 issues in detail in my QA Report.

The Architecture change I would recommend here is to use `transferShares` instead of `transferfrom` when `asset` used is `stETH`.

Reason: `stETH` balances changes upon transfers, mints/burns, and rebases, while shares balances can only change upon transfers and mints/burns. This is why it is recommended even by Lido in their [docs](https://docs.lido.fi/guides/lido-tokens-integration-guide/#steth-internals-share-mechanics) here that Use `transferShares` instead of `transferfrom` to avoid handling issues which can arise because of rebasing mechanism.

## Centralization Risk

* `EthenaMinting`: While minting `USDe`, User's `stETH` are transferred to `custodianAddress`. User needs to trust Ethena that They will be allowed to `redeem` their tokens for a fair value by giving back `USDe`. If they don't trust Ethena, they should avoid minting `USDe` as this is a potential Rug pull attack vector.

* `StakedUSDe`: Users needs to be aware of:
    * `Blacklisting`: Any user can be added to a blacklist by the Blacklist Manager, which may restrict their ability to transfer funds.
    * `redistributeLockedAmount function`: Once Blacklisted, `ADMIN` holds the power to burn User's `stUSDe` and mint it to any other address.

## Systemic risks

* The way funds are managed from `custodianAddress` will determine whether user will be able to redeem the assets at a fair valuation.

* While `stETH` is not Fees on transfer token, but transferring it through `transferFrom` will still lead to loss of few wei because the way it works is:

  1. User A transfers 1 `stETH` to User B.
  2. Under the hood, `stETH` balance gets converted to shares, integer division happens and rounding down applies using formula: `shares[account] = balanceOf(account) * totalShares / totalPooledEther`.
  3. The corresponding amount of shares gets transferred from User A to User B.
  4. Shares balance gets converted to stETH balance for User B.
  5. In many cases, the actually transferred amount is 1-2 wei less than expected.

## Time Spent

| Total Number of Hours | 18 |
|:--:|:--:|

-----------------------------------------------------------

# QA Report

## Low Risk Issues
| Count | Explanation | 
|:--:|:-------|
| [L-01] | Multiple Issues can occur while using `stETH` in `EthenaMinting` |  
| [L-02] | Accidental/Deliberate griefing Attack possible in `StakedUSDeV2` | 
| [L-03] | Malicious User can frontrun to avoid getting Blacklisted |

| Total Low Risk Issues | 3 |
|:--:|:--:|

## Non-Critical Issues
| Count | Explanation | 
|:--:|:-------|
| [N-01] | Improvement in `verifyRoute` function in `EthenaMinting` | 

| Total Non-Critical Issues | 1 |
|:--:|:--:|

### [L-01] Multiple Issues can occur while using `stETH` in `EthenaMinting`

To start off, `asset` used to mint `USDe` will be `stETH` which is a rebasing token. So it can lead to 2 following issues:

1. Slippage in value becuase of rebasing when transaction is pending.
2. Value received by custodian addresses will always be less than intended which can break the logics used further.

#### How first Issue can occur:

Assume Alice signed a transaction to mint `USDe` for `x` amount of `stETH`. While the transaction is in mempool, rebase occurs and:

  * **If positive rebase occurs (most likely)**
    * Protocol will get less amount of shares for `x` amount of `stETH` which was originally thought off, leading to loss of funds for Protocol.

  * **If negative rebase occurs**
    * User will be force to pay more shares than originally intended leading to loss of funds for User.

#### To understand second issue, Let's Understand how transfer works underhood in `stETH`:

1. User A transfers 1 `stETH` to User B.
2. Under the hood, `stETH` balance gets converted to shares, integer division happens and rounding down applies using formula: `shares[account] = balanceOf(account) * totalShares / totalPooledEther`.
3. The corresponding amount of shares gets transferred from User A to User B.
4. Shares balance gets converted to stETH balance for User B.
5. In many cases, the actually transferred amount is 1-2 wei less than expected.

In this example, User A is user and User B is custodian Address. So in case custodian address is a contract where further logics are used to allocate this funds, it can break the assumption that exact value of `stETH` has been transferred.

These are common issues for protocol integrating `stETH` with their logic. That is why, Lido has provided solution for it.

#### Recommendation

Use  `transferShares` instead of using `transferfrom` while transfering  `stETH` because during Rebasing event, underlying shares remain same but the balance of `stETH` can increase or decrease.

This Solves:
  * In case of Rebasing when the transaction is pending, the shares will be transferred whose value will also increase or decrease proportionally leading to no Slippage for any party involve.
  * There will be no Dust amount lost because of rounding error in `stETH`.

Lido has officially recommended this in their [doc](https://docs.lido.fi/guides/lido-tokens-integration-guide/#steth-internals-share-mechanics) here which I would recommend you to go through.

> When integrating stETH as a token into any dApp, it's highly recommended to store and operate shares rather than stETH public balances directly, because stETH balances change both upon transfers, mints/burns, and rebases, while shares balances can only change upon transfers and mints/burns.

### [L-02] Accidental/Deliberate griefing Attack possible in `StakedUSDeV2`

During Cooldown `ON` State of `StakedUSDeV2`, Anyone with allowance from a User can call `cooldownAssets` on User's behalf.

Consider the following scenario:

1. Alice staked some `USDe` in the vault and got some `stUSDe` minted for her.
2. As `stUSDe` itself is an `ERC20` standard token, Alice gives allowance to use some amount of `stUSDe` to someone else let's say Bob.
3. Alice wants to unstake some tokens so she calls `cooldownAssets` making her `cooldownEnd` time as 90 days (considering default `cooldownDuration`).
4. Now Bob, can anytime call `cooldownAssets` with the smaller amount and `owner` as Alice to increase the `cooldownEnd` period my another 90 days. In worst case, Bob can call it on 90th day which increase the `cooldownEnd` period to effectively 180 days for Alice.

```solidity
File: StakedUSDeV2.sol

  function cooldownAssets(uint256 assets, address owner) external ensureCooldownOn returns (uint256) { // @audit Griefing Attack Possible with allowance from owner
    if (assets > maxWithdraw(owner)) revert ExcessiveWithdrawAmount();

    uint256 shares = previewWithdraw(assets);

    cooldowns[owner].cooldownEnd = uint104(block.timestamp) + cooldownDuration;
    cooldowns[owner].underlyingAmount += assets;

    _withdraw(_msgSender(), address(silo), owner, assets, shares);

    return shares;
  }

```

#### Recommendation

2 Options:

* Don't allow anyone to pass arbitrary `owner` addresses, instead use `cooldowns[msg.sender]` to update state variables.
* Add this risk in Docs so that User can be well aware of it before giving allowance to any other party.

### [L-03] Malicious User can frontrun to avoid getting Blacklisted

Here's how:

* `BLACKLIST_MANAGER_ROLE` has power to blacklist any address.
* Let's assume it wants to blacklist Alice's address.
* Blacklist Manager will call `addToBlacklist` with Alice's address.
* Alice tracks this transaction in mempool.
* Alice frontruns and transfer all her funds to other address.

This way any User can manage to avoid getting blacklisted. Possible solution here is to use Flashbots to privately add this transaction to blockchain.

### [N-01] Improvement in `verifyRoute` function in `EthenaMinting`

* `verifyRoute` is called only in `mint` function where the funds from the user can be passed to the different `custodianAddresses`.

* So it is unnecessary to check the order type here as it can never be `OrderType.REDEEM`. So it is unnecessary check which can be removed.

```solidity
File: EthenaMinting.sol

    function verifyRoute(Route calldata route, OrderType orderType) public view override returns (bool) {
      // routes only used to mint
@->   if (orderType == OrderType.REDEEM) {
        return true;
      }
      
      //---SNIP----//
    }

```

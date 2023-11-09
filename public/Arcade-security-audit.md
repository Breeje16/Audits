# Analysis

## Introduction

This technical analysis report delves into the underlying smart contracts that constitute the token-based governance system of Arcade.xyz. This report explores various contract categories, including Voting Vaults, Core Voting Contracts, Token, and NFT, while highlighting insights and potential attack vectors in the governance ecosystem.

## Architecture

1. Voting Vaults Contracts Inheritance Graph:

![Vesting Vaults](https://res.cloudinary.com/davyfibzy/image/upload/v1690553946/nogvvqu3t5mdkvip2zxk.png)

2. Core Voting Contracts Inheritance Graph:

![Core Voting](https://res.cloudinary.com/davyfibzy/image/upload/v1690553800/axavsr5jmwccr4om3qjz.png)

## NFTs

Arcade.xyz uses ERC1155 tokens to grant multipliers to users' voting power. `ReputationBadge.sol` contract consists the majority of the Core logic accompanying by tokenURI descriptor contract.

**Contracts:**

1. `ReputationBadge.sol`: This is the main contract which helps users mint the NFTs by providing merkle proofs. This NFT Badge can be used in governance to give a multiplier to a user's voting power.
2. `BadgeDescriptor.sol`: Basic descriptor contract for badge NFTs which helps setting and getting `tokenURI`.

### Insights

* Badge Manager is supposed to add claim data first which will be validated when user mints the NFTs.
* There is no Check for `_claimData[i].tokenId` to be not equal to zero.
* In case, a user sent more than the `mintPrice`, then the remain amount is not refunded to the User.
* Badge Manager can Withdraw all the Eth fees anytime which includes any excess pay done by the user.
* Users have option to claim the Claimable amount in 1 transaction or can claim over time. But it should be claimed before the expiration time.

### Possible Attack Vectors

* `_mint` is used instead of `_safeMint` which can lead to loss of NFTs incase user's address doesn't accept ERC1155.
* In Case, Badge Manager adds `_claimData[i].tokenId` = 0, then the User who claims that NFTs will not be able to get `multiplier` in their voting rights.
* `.transfer` used which uses hardcoded gas. `.call` should be used instead with a check for reentrancy.

## Tokens

This contracts are responsible for the token's initial deployment and distribution, including the airdrop and initial distributor contracts. It provides users with voting power and influence.

**Contracts:**

1. `ArcadeToken.sol`: This contract is an ERC20 token implementation for the Arcade token [ARCD] with an inflationary cap of 2% per year.
2. `ArcadeTokenDistributor.sol`: This contract is used for distribution of Arcade Tokens to the various Stake holders of the Protocol.
3. `ArcadeAirdrop.sol`: This contract receives tokens from the ArcadeTokenDistributor and facilitates airdrop claims.

### Insights

* Initially, 100 Million `ARCD` Tokens are minted to `ArcadeTokenDistributor.sol`.
* There is a Cooldown period of 365 days between mints such that no tokens can be minted in between.
* Mint function has an inflation capped at 2% per year.
* Every function inside `ArcadeTokenDistributor` can be called just once that too only by the owner.
* For Airdrops, in case no one claims, Owner can reclaim it but only after expiration.

### Possible Attack Vectors

* Use of `_mint` instead of `_safeMint` in `ArcadeToken` contract.
* In `ArcadeTokenDistributor`, if by mistake owner call any other function before `setToken` function, then `ARCD` tokens which were supposed to be send through that function will be frozen forever.

## Arcade Treasury

This contract hold the treasury funds and manages the amount that needs to be approved or spend with 3 threshold levels which includes: large amount, medium amount and small amount.

### Insights

* Thresholds are modifiable via the governance timelock (ADMIN role).
* `CORE_VOTING_ROLE` can call spend functions with custom quorums based on the spend thresholds.
* The GSC can spend within its allowance for each token, which is updated by the ADMIN role.
* In case new threshold.small is less than the current allowance for GSC, then the current allowance will be slashed to new threshold.small value.
* Allowance updates have a 7-day cooldown and cannot exceed the small threshold to require governance votes for larger spends.

### Possible Attack Vectors

* No protection against Potential Huge return data values returning through `call` made in `batchCalls`.
* Wrong Implementation of `blockExpenditure` for a particular Block number. As per Natspec, it was supposed to limit the max tokens that can be spent/approved in a single block for a particular threshold. But Implementation differs from that.
* In case of Tokens with Multiple addresses, Cooldown period can be bypassed.

## ARCDVestingVault

This contract is a vesting vault which help creating and managing vesting grants for Arcade tokens over time. `ImmutableVestingVault` is also a `ARCDVestingVault` but without the functionality to revoke a grant.

### Graph Representation

![Graph](https://res.cloudinary.com/davyfibzy/image/upload/v1690555651/uikr1zygtzlaa586bo9p.png)

### Insights

* The manager can add or remove grants and is changeable via the timelock.
* The manager can deposit the tokens to increase the `unassigned` amount and withdraw only the left over `unassigned` amount after giving grant.
* Grants have a delegatee address that receives voting power, updatable by the grant recipient.
* Three time parameters define each grant: "created," "cliff," and "expiration" block numbers.
* Tokens are unlocked gradually from the "cliff" block to the "expiration" block.
* No emergency withdrawal exists, and only deposited funds are recoverable.
* In case of revoke, Funds upto that particular time is sent to the grant owner and remaining funds are transferred back to manager instead of adding it to `unassigned` number.
* User have option to use `queryVotePower` and clear everything more than `staleBlockLag` into the past.

### Possible Attack Vectors

First of all, Code is written really well. Still hypothetical attack vector possible here:

* In case, value of `staleBlockLag` is very low, then a grant owner can potentially do a Griefing Attack where he calls `queryVotePower` to clear everything more than `staleBlockLag` into the past which might lead to voting power becoming `0`. And because of this, there will be DoS in `_syncVotingPower` resulting in DoS in `claim` and `revokeGrant` function.

## NFTBoostVault

This contract enables Users to enhance their voting power in governance through ERC1155 NFT multipliers.

### Insights

* Users deposit ERC20 tokens and provide ERC1155 NFTs as calldata to confirm ownership.
* Their Voting power multiplier will depend on the `multiplierData.multiplier` value set by the Manager for that particular NFT, ID pair.
* Users can register or update their registration with more token through Airdrop Contract.
* Option to Withdraw or add new tokens or NFTs.
* Voting Power will be sync corresponding to the action performed.
* No emergency withdrawal exists, and funds not sent via `addNftAndDelegate()` are unrecoverable.
* Once entire registration amount is withdrawn, NFT will be withdrawn too and user will need to register again before doing any other operation.

### Possible Attack Vectors

* `ERC1155` tokens outside the ones set in Multiplier Set are not allowed. But in UpdateNFT function, it is possible to lock any random ERC1155 in the contract because of lack of check.
* Same attack vector as `ARCDVestingVault` exist in case `staleBlockLag` is very low.
* Trusting `Timelock` contract to call `unlock`. In case, it doesn't user won't be able to withdraw tokens.

-----------------------------------------------------------

# QA Report

## Low Risk Issues
| Count | Explanation |
|:--:|:-------|
| [L-01] | User can fail to get Voting power multiplier despite having `ReputationBadge` NFT |
| [L-02] | Risk of Funds getting stuck in `ArcadeTokenDistributor` |
| [L-03] | No defence against return data bombs in `batchCalls` |  
| [L-04] | Cooldown period for GSC allowance can be bypassed in case of token with multiple addresses |   
| [L-05] | `NFTBoostVault` can accept Random NFTs whose `multiplier` is not set |   
| [L-06] | Potential DoS in `claim` and `revokeGrant` function because of underflow if `staleBlockLag` is too low |   

| Total Low Risk Issues | 6 |
|:--:|:--:|

**NOTE: Multiple Low Risk Issues were Upgraded to Medium in Code4rena.**

### [L-01] User can fail to get Voting power multiplier despite having `ReputationBadge` NFT

## Description

Users can claim their Reputation Badge NFT by providing merkle Proofs. But before claiming, Badge Manager is expected to add data related to the Root, expiration and prices which is going to be used during `mint` process.

```solidity
File: ReputationBadge.sol

  function publishRoots(ClaimData[] calldata _claimData) external onlyRole(BADGE_MANAGER_ROLE) {
      if (_claimData.length == 0) revert RB_NoClaimData();
      if (_claimData.length > 50) revert RB_ArrayTooLarge();

      for (uint256 i = 0; i < _claimData.length; i++) {
          // ----SNIP: expiration check------

          claimRoots[_claimData[i].tokenId] = _claimData[i].claimRoot;
          claimExpirations[_claimData[i].tokenId] = _claimData[i].claimExpiration;
          mintPrices[_claimData[i].tokenId] = _claimData[i].mintPrice;
      }
  }

```

Issue here: There is no Checks for `_claimData[i].tokenId != 0`.

In case Badge Manager uses `_claimData[i].tokenId = 0`, then:

1. User will pay the price to mint the NFT.
2. User will call `addNftAndDelegate` in `NFTBoostVault` and expect to get a multiplier to his/her voting power.
3. Further `addNftAndDelegate` calls 2 functions: `_registerAndDelegate` and `_lockTokens`.
4. User will fail to transfer the NFT and get the Voting Rights multiplier because of Line 659 in `NFTBoostVault`.

```solidity
File: NFTBoostVault.sol

      function _lockTokens(
          address from,
          uint256 amount,
          address tokenAddress,
          uint128 tokenId,
          uint128 nftAmount
      ) internal {
          token.transferFrom(from, address(this), amount);

659:      if (tokenAddress != address(0) && tokenId != 0) {
              _lockNft(from, tokenAddress, tokenId, nftAmount);
          }
      }

```

## Recommendation

Add a condition in `publishRoots` function of  `ReputationBadge` such that it reverts in case `_claimData[i].tokenId` is equal to `0`.

### [L-02] Risk of Funds getting stuck in `ArcadeTokenDistributor`

## Description

`ArcadeTokenDistributor` contract is used for the distribution of Arcade Tokens between various stakeholders of the protocol.

Few points about it:

1. `arcadeToken` is not set initially.
2. Every function such as `toDevPartner` can be called only once throughout it's lifetime.
3. For every Function, there is NO check of `arcadeToken != address(0)`.

```solidity
File: ArcadeTokenDistributor.sol

  function toDevPartner(address _devPartner) external onlyOwner {
      if (devPartnerSent) revert AT_AlreadySent();
      if (_devPartner == address(0)) revert AT_ZeroAddress("devPartner");

      devPartnerSent = true;

      //------SNIP: Transfer----//

      //------SNIP: Event----//
  }

  function setToken(IArcadeToken _arcadeToken) external onlyOwner {
      //------SNIP: Validation----//

      arcadeToken = _arcadeToken;
  }

```

So there exist a possible risk where:

1. Total initial funds which is equal to the sum of distribution through all function is deposited in contract.
2. Owner by mistake calls any one method before setting the `arcadeToken`.

In this case, the remaining tokens for that called function will be stuck in the contract forever.

### [L-03] No defence against return data bombs in `batchCalls`

## Description

`batchCalls` function is supposed to be called by the `ADMIN_ROLE`.

```solidity
File: ArcadeTreasury.sol

341:     (bool success, ) = targets[i].call(calldatas[i]);
342:     // revert if a single call fails
343:     if (!success) revert T_CallFailed();

```

Here `(bool success, )` is actually the same as writing `(bool success, bytes memory data)` which basically means that even though the `data` is omitted it doesn’t mean that the contract does not handle it. 

Actually, the way it works is the `bytes data` that was returned from the `target` will be copied to memory. Memory allocation becomes very costly if the payload is big, so this means that if a `target` implements a fallback function that returns a huge payload, then the `msg.sender` of the transaction, in our case the `ADMIN_ROLE`, will have to pay a huge amount of gas for copying this payload to memory.

## Recommendation

Use a low-level assembly `call` since it does not automatically copy return data to memory

```solidity
  bool success;
  assembly {
      success := call(gas(), targets[i], 0, add(calldatas[i], 0x20), mload(calldatas[i]), 0, 0)
  }
```

### [L-04] Cooldown period for GSC allowance can be bypassed in case of token with multiple addresses

## Description

There exist ERC20 tokens that have more than one address which you can check [Here](https://github.com/d-xo/weird-erc20#multiple-token-addresses).

As per Natspac of `setGSCAllowance` function:
> There is a cool down period of 7 days after this function has been called where it cannot be called again.

But In case of multiple address token, 

```solidity
File: ArcadeTreasury.sol

308:    if (uint48(block.timestamp) < lastAllowanceSet[token] + SET_ALLOWANCE_COOL_DOWN) { 
309:        revert T_CoolDownPeriod(block.timestamp, lastAllowanceSet[token] + SET_ALLOWANCE_COOL_DOWN);
310:    }

```

The above check will bypass and allow to set GSC allowance again within the cooldown period.

### [L-05] `NFTBoostVault` can accept Random NFTs whose `multiplier` is not set

## Description

There are 2 ways for users to register and add NFTs to enhance their voting power by a `multiplier`.

1. Using `addNftAndDelegate` Function
2. Receiving Airdrop through `airdropReceive` and then adding NFT through `updateNft`.

For the First process, In `_registerAndDelegate`, there is a check shown with (@->) below where in case if their is no Multiplier Set corresponding to `_tokenAddress` and `_tokenId` then the function reverts.

```solidity
File: NFTBoostVault.sol

  function _registerAndDelegate(
        address user,
        uint128 _amount,
        uint128 _tokenId,
        address _tokenAddress,
        address _delegatee
    ) internal {
        uint128 multiplier = 1e3;

        // confirm that the user is a holder of the tokenId and that a multiplier is set for this token
        if (_tokenAddress != address(0) && _tokenId != 0) {
            if (IERC1155(_tokenAddress).balanceOf(user, _tokenId) == 0) revert NBV_DoesNotOwn();

            multiplier = getMultiplier(_tokenAddress, _tokenId);

@->         if (multiplier == 0) revert NBV_NoMultiplierSet();
        }

        //---SNIP: Continuation---//
    }

```

This makes sure that only Selected NFTs and their corresponding tokenIds can be locked in the contract.

But in case a user uses second step or after doing first step, updates the NFT, then there is no check to make sure that the NFT locked in the contract is there in Multiplier Set.

```solidity
File: NFTBoostVault.sol

  function updateNft(uint128 newTokenId, address newTokenAddress) external override nonReentrant {
      if (newTokenAddress == address(0) || newTokenId == 0) revert NBV_InvalidNft(newTokenAddress, newTokenId);

      if (IERC1155(newTokenAddress).balanceOf(msg.sender, newTokenId) == 0) revert NBV_DoesNotOwn();

      NFTBoostVaultStorage.Registration storage registration = _getRegistrations()[msg.sender];

      //----SNIP: Continuation---//
  }

```

## Recommendation

Add the following 2 lines in `updateNft` function:

```solidity

  multiplier = getMultiplier(newTokenAddress, newTokenId); 
  if (multiplier == 0) revert NBV_NoMultiplierSet();

```

### [L-06] Potential DoS in `claim` and `revokeGrant` function because of underflow if `staleBlockLag` is too low

## Description

In case, value of `staleBlockLag` is very low, then anyone can potentially do a Griefing Attack where he/she calls `queryVotePower` to clear everything more than `staleBlockLag` into the past for a grant owner (Note: `queryVotePower` function has no access control and anyone can call it) which might lead to voting power becoming `0` in case grant owner's last voting power update was before `staleBlockLag` number of blocks.

Now, there will be difference between `delegateeVotes` and `change` in the below line.

```solidity
File: ARCDVestingVault.sol

344:    uint256 delegateeVotes = votingPower.loadTop(grant.delegatee);

350:    int256 change = int256(newVotingPower) - int256(grant.latestVotingPower);
351:    // we multiply by -1 to avoid underflow when casting
352:    votingPower.push(grant.delegatee, delegateeVotes - uint256(change * -1));

```

Here, `delegateeVotes` will be zero as user have removed every past data and change will be a positive number as `grant.latestVotingPower` is still as it is. So subtracting anything from `0` will lead to underflow.

So, The line at 352 will revert. This means `_syncVotingPower` will be permanently DoSed for the user which can lead to DoS in `claim` and `revokeGrant` function.

This means the fund for that grant will be stuck in the contract forever.

---------------------------------------------------------

## Medium and High Risk Issues

### [M-01] Wrong Implementation of `blockExpenditure` can lead to DoS for Small and Medium Spend despite not crossing limits.

## Impact

DoS for Small and medium spend/approve despite not crossing limits.

## Proof of Concept

As per Natspec of `_spend` and `_approve` in `ArcadeTreasury.sol` contract:

> @param limit:             max tokens that can be spent/approved in a single block for this threshold

Basically, there are 3 thresholds Small, Medium and Large to limit the amount of tokens spent or approved.

As per the Natspec, the parameter `limit` is supposed to the maximum amount of tokens that can be either spent or approved in a single block `for that particular threshold`.

This means in case of following situation:

1. Threshold set are: `Small = 100 Tokens`, `Medium = 500 Tokens` and `Large = 1000 Tokens`.
2. Now for a particular `block.number`, As per Natspec, Maximum spend/approve should be `100` Tokens through Small, `500` Tokens through Medium and `1000` Tokens through Large.

However, an inconsistency arises between the intended behavior, as described in the Natspec, and the actual implementation.

```solidity
File: ArcadeTreasury.sol

  function _spend(address token, uint256 amount, address destination, uint256 limit) internal {
      // check that after processing this we will not have spent more than the block limit
      uint256 spentThisBlock = blockExpenditure[block.number];
      if (amount + spentThisBlock > limit) revert T_BlockSpendLimit();
      blockExpenditure[block.number] = amount + spentThisBlock;

      //----SNIP: Transfer-----//
  }

```
[Link to code](https://github.com/code-423n4/2023-07-arcade/blob/main/contracts/ArcadeTreasury.sol#L361)

Consider the following situation:

1. A Large spend with of `1000` Tokens is done through `CORE_VOTING_ROLE`.
2. `blockExpenditure[block.number]` is updated to `1000`.
3. Now, any other Medium or Low Spend will result in DoS given that the limit it passes, will be either be `500` or `100` as per our example and it will always be less than `spentThisBlock` value.

This way small and medium Spend will DoS despite having their `particular threshold` not breached.

## Tools Used

VS Code

## Recommended Mitigation Steps

Add a Global Spend Limit for a block and Update the code as:

```diff
File: ArcadeTreasury.sol

  function _spend(address token, uint256 amount, address destination, uint256 limit) internal {
      // check that after processing this we will not have spent more than the block limit
      uint256 spentThisBlock = blockExpenditure[block.number];
-     if (amount + spentThisBlock > limit) revert T_BlockSpendLimit();
+     if (amount + spentThisBlock > GLOBAL_SPEND_LIMIT_PER_BLOCK) revert T_BlockSpendLimit();
+     if (amount > limit) revert T_ThresholdLimitExceeded();
      blockExpenditure[block.number] = amount + spentThisBlock;

      //----SNIP: Transfer-----//
  }

```

This way, there will a limit check for the spending/approval as well as the blockExpenditure check making sure that it doesn't cross the Block Spend Limit. Apply similar change in `_approve` function as well.

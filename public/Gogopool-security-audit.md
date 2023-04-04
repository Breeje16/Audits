# GoGoPool Audit Report

## Gas Optimizations

| |Issue|Instances|
|-|:-|:-:|
| [GAS-1](#GAS-1) | FUNCTIONS GUARANTEED TO REVERT WHEN CALLED BY NORMAL USERS CAN BE MARKED PAYABLE
| [GAS-2](#GAS-2) | USAGE OF UINTS/INTS SMALLER THAN 32 BYTES (256 BITS) INCURS OVERHEAD | 1 |
| [GAS-3](#GAS-3) | NOT USING THE NAMED RETURN VARIABLES WHEN A FUNCTION RETURNS, WASTES DEPLOYMENT GAS | 2 |
| [GAS-4](#GAS-4) | ++i/i++ should be unchecked{++i}/unchecked{++i} when it is not possible for them to overflow, as is the case when used in for- and while-loops | 7 |
| [GAS-5](#GAS-5) | Comparing to a constant (true or false) is a bit more expensive than directly checking the returned boolean value. | 3 |

###  [GAS-1] FUNCTIONS GUARANTEED TO REVERT WHEN CALLED BY NORMAL USERS CAN BE MARKED PAYABLE

If a function modifier such as onlyOwner is used, the function will revert if a normal user tries to pay the function. Marking the function as payable will lower the gas cost for legitimate callers because the compiler will not include checks for whether a payment was provided. 

The extra opcodes avoided are CALLVALUE(2), DUP1(3), ISZERO(3), PUSH2(3), JUMPI(10), PUSH1(3), DUP1(3), REVERT(0), JUMPDEST(1), POP(2), which costs an average of about 21 gas per call to the function, in addition to the extra deployment cost


*Instances*:
All the public and External Functions which uses onlyGuardian, guardianOrSpecificRegisteredContract, onlyMultisig, guardianOrRegisteredContract, onlyRegisteredNetworkContract or onlySpecificRegisteredContract modifier.

###  [GAS-2] USAGE OF UINTS/INTS SMALLER THAN 32 BYTES (256 BITS) INCURS OVERHEAD

Making lastSync as uint256 will help Saving gas as there will be no Overhead with the next 3 (rewardCycleLength, rewardsCycleEnd and lastRewardsAmt) getting packed in 1 storage slot.


*Instances (1)*:
```solidity
File: contract/tokens/TokenggAVAX.sol

	/// @notice the effective start of the current cycle
	uint32 public lastSync;

	/// @notice the maximum length of a rewards cycle
	uint32 public rewardsCycleLength;

	/// @notice the end of the current cycle. Will always be evenly divisible by `rewardsCycleLength`.
	uint32 public rewardsCycleEnd;

	/// @notice the amount of rewards distributed in a the most recent cycle.
	uint192 public lastRewardsAmt;

```
[Link to code](https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/tokens/TokenggAVAX.sol#L40-L50)

###  [GAS-3] NOT USING THE NAMED RETURN VARIABLES WHEN A FUNCTION RETURNS, WASTES DEPLOYMENT GAS

It is not necessary to have both a named return and a return statement.

*Instances (2)*:
```solidity
File: contract/MinipoolManager.sol

  function getMinipoolByNodeID(address nodeID) public view returns (Minipool memory mp) {
      int256 index = getIndexOf(nodeID);
      return getMinipool(index);
  }

```
[Link to code](https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/MinipoolManager.sol#L572-L575)

```solidity
File: contract/RewardsPool.sol#L66-L78

66:    function getInflationAmt() public view returns (uint256 currentTotalSupply, uint256 

77:    return (currentTotalSupply, newTotalSupply);

```
[Link to code](https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/RewardsPool.sol#L66-L78)

### [GAS-4] ++i/i++ should be unchecked{++i}/unchecked{++i} when it is not possible for them to overflow, as is the case when used in for for-loops.

*Instances (7)*:
```solidity
File: contract/MinipoolManager.sol

619:       for (uint256 i = offset; i < max; i++) {

File: contract/MultisigManager.sol

84:        for (uint256 i = 0; i < total; i++) {

File: contract/Ocyticus.sol

61:        for (uint256 i = 0; i < count; i++) {

File: contract/RewardsPool.sol

74:        for (uint256 i = 0; i < inflationIntervalsElapsed; i++) {

215:       for (uint256 i = 0; i < count; i++) {

230:       for (uint256 i = 0; i < enabledMultisigs.length; i++) {

File: contract/Staking.sol

428:       for (uint256 i = offset; i < max; i++) {

```
[Link to code](https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/)


### [GAS-5] Comparing to a constant (true or false) is more expensive than directly checking the returned boolean value.

I would recommend to use "!" operator ahead in the following if conditions instead of comparision.

*Instances (3)*:
```solidity
File: contract/BaseAbstraction.sol

25:    if (getBool(keccak256(abi.encodePacked("contract.exists", msg.sender))) == false) {

74:    if (enabled == false) {

File: contract/Storage.sol

29:    if (booleanStorage[keccak256(abi.encodePacked("contract.exists", msg.sender))] == false &&

```
[Link to code](https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/MinipoolManager.sol#L572-L575)


## QA Report

| |Issue|Instances|
|-|:-|:-:|
| [L-1](#L-1) | NO ZERO ADDRESS CHECK IN TRANSFERING OWNERSHIP OF GUARDIAN | 1 |
| [L-2](#L-2) | Single Point of Failure
| [NC-1](#NC-1) | Storage key collision
| [NC-2](#NC-2) | Reentrancy Gaurd on Critical functions | 2 |
| [NC-3](#NC-3) | Constants should be defined rather than using magic numbers | 1 |

###  [L-1] NO ZERO ADDRESS CHECK IN TRANSFERING OWNERSHIP OF GUARDIAN

Having a Zero Address check is Important while transfering the Ownership of Guardian. A Zero Address smart contract as guardian can lead to dangerous implecations.

I would suggest to Use assembly to check for address(0)

*Instances (1)*:
```solidity
File: contract/Storage.sol

  function setGuardian(address newAddress) external {
		// Check tx comes from current guardian
		if (msg.sender != guardian) {
			revert MustBeGuardian();
		}
		// Store new address awaiting confirmation
		newGuardian = newAddress;
	}

```
[Link to code](https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/Storage.sol#L41-L48)

### [L-2] Single Point of Failure

If an attacker can register a single malicious contract, It has all the controls of the network to do all Damage possible. It can register anyone as Multisig through Storage Access, it can get GGP price, it can change all the Minipool, Staking and RewardsPool data in its favour or can Re-initialise Contract like: `ProtocolDAO.sol` or `RewardsPool.sol`.

###  [NC-1] Storage key collision

In `Storage.sol` contract, The keys used are defined throughout the codebase, with some keys used across multiple contracts, others used across several but set only by one, and others used privately by a single contract.

There is some risk that maintainers accidentally reuse a key to store an unrelated value, resulting in a clash where storage values important to another contract are overwritten.
While no bugs have been identified in the current codebase associated with this issue, this can create issue in long run.

### [NC-2] Reentrancy Gaurd on critical payable functions

It is recommended to use Reentrancy Guard on Functions that can Potentially be Exploited by Reentrancy.

*Instances (2)*:
```solidity
File: contract/tokens/TokenggAVAX.sol

166:    function depositAVAX() public payable returns (uint256 shares) {

```
[Link to code](https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/tokens/TokenggAVAX.sol#L166)

```solidity
File: contract/ClaimNodeOp.sol

89:    function claimAndRestake(uint256 claimAmt) external

```
[Link to code](https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/ClaimNodeOp.sol#L89)

### [NC-3] Constants should be defined rather than using magic numbers

*Instances (1)*:
```solidity
File: contract/ProtocolDAO.sol

41:    setUint(keccak256("ProtocolDAO.InflationIntervalRate"), 1000133680617113500);

```
[Link to code](https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/ProtocolDAO.sol#L41)

## Medium / High Risk

| |Issue|
|-|:-|
| [M-1](#M-1) | IN TokenggAVAX.sol SOME USERS MAY NOT BE ABLE TO WITHDRAW UNTIL REWARDSCYCLEEND THE DUE TO UNDERFLOW IN BEFOREWITHDRAW()
| [M-2](#M-2) | REWARDS DELAY RELEASE COULD CAUSE YIELDS STEAL AND LOSS
| [H-1](#H-1) | FIRST TokenggAVAX DEPOSIT EXPLOIT CAN BREAK SHARE CALCULATION

###  [M-1] IN TokenggAVAX.sol SOME USERS MAY NOT BE ABLE TO WITHDRAW UNTIL REWARDSCYCLEEND THE DUE TO UNDERFLOW IN BEFOREWITHDRAW()

```solidity
File: contract/tokens/TokenggAVAX.sol

  function beforeWithdraw(
		uint256 amount,
		uint256 /* shares */
	) internal override {
		totalReleasedAssets -= amount;
	}

```

```solidity
File: contract/tokens/TokenggAVAX.sol

  function syncRewards() public {
		uint32 timestamp = block.timestamp.safeCastTo32();

		if (timestamp < rewardsCycleEnd) {
			revert SyncError();
		}

		uint192 lastRewardsAmt_ = lastRewardsAmt;
		uint256 totalReleasedAssets_ = totalReleasedAssets;
		uint256 stakingTotalAssets_ = stakingTotalAssets;

		uint256 nextRewardsAmt = (asset.balanceOf(address(this)) + stakingTotalAssets_) - totalReleasedAssets_ - lastRewardsAmt_;

		// Ensure nextRewardsCycleEnd will be evenly divisible by `rewardsCycleLength`.
		uint32 nextRewardsCycleEnd = ((timestamp + rewardsCycleLength) / rewardsCycleLength) * rewardsCycleLength;

		lastRewardsAmt = nextRewardsAmt.safeCastTo192();
		lastSync = timestamp;
		rewardsCycleEnd = nextRewardsCycleEnd;
		totalReleasedAssets = totalReleasedAssets_ + lastRewardsAmt_;
		emit NewRewardsCycle(nextRewardsCycleEnd, nextRewardsAmt);
	}

```
[Link to code](https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/tokens/RokenggAVAX.sol)

`totalReleasedAssets` is a cached value of total assets which will only include the unlockedRewards when the whole cycle ends.

This makes it possible for `totalReleasedAssets -= amount` to revert when the withdrawal amount exceeds `totalReleasedAssets`, as the withdrawal amount may include part of the unlockedRewards in the current cycle.

## Proof of Concept

Given:

* rewardsCycleLength = 14 days
* Alice deposit() 100 AVAX tokens
* 5 AVAX tokens are transferred as rewards and syncRewards() was called
* 1 day later, Alice redeem() with all shares, the transaction will revert at beforeWithdraw().

Alice’s shares worth 105 AVAX at this moment, but totalReleasedAssets = 100, making totalReleasedAssets -= amount reverts due to underflow.

Bob deposit() 5 AVAX tokens;
* Alice withdraw() 105 AVAX tokens, totalReleasedAssets becomes 0;
* Bob can’t even withdraw 1 wei of AVAX token, as totalReleasedAssets is now 0.
* If there are no new deposits, both Alice and Bob won’t be able to withdraw any of their funds until `rewardsCycleEnd`

## Recommended Mitigation Steps
Consider changing to:

```solidity
File: contract/tokens/TokenggAVAX.sol

  function beforeWithdraw(
		uint256 amount,
		uint256 /* shares */
	) internal override {
    uint256 _totalReleasedAssets = totalReleasedAssets;
    if (amount >= _totalReleasedAssets) {
        uint256 _totalAssets = totalAssets();
        lastRewardAmount -= _totalAssets - _totalReleasedAssets;
        lastSync = block.timestamp;
        totalReleasedAssets = _totalAssets - amount;
    } else {
        totalReleasedAssets = _totalReleasedAssets - amount;
    }
  }

```
[Link to code](https://github.com/code-423n4/2022-12-gogopool/blob/main/contracts/contract/tokens/TokenggAVAX.sol#L241-L246)

### [M-2] REWARDS DELAY RELEASE COULD CAUSE YIELDS STEAL AND LOSS

In the current rewards accounting, vault shares in `deposit()` and `redeem()` can not correctly record the spot yields generated by the staked asset. Yields are released over the next rewards cycle. As a result, malicious users can steal yields from innocent users by picking special timing to deposit() and redeem().

If withdraw just after the `rewardsCycleEnd` timestamp, a user can not get the yields from last rewards cycle. Since the totalAssets() only contain `totalReleasedAssets` but not the yields part. It takes 1 rewards cycle to linearly add to the `totalReleasedAssets`.

Assume per 10,000 asset staking generate yields of 70 for 7 days, and the reward cycle is 1 day. A malicious user Alice can do the following:

* Watch the mempool for withdraw(10,000) from account Bob, front run it with syncRewards(), so that the most recent yields of amount 70 from Bob will stay in the vault.
* Alice will also deposit a 10,000 to take as much shares as possible.
* After 1 rewards cycle of 1 day, redeem() to take the yields of 70.

Effectively steal the yields from Bob. The profit for Alice is not 70, because after 1 day, her own deposit also generates some yield, in this example this portion is 1. At the end, Alice steal yield of amount 60.

## Recommended Mitigation Steps

* For the `lastRewardsAmt` not released, allow the users to redeem as it is linearly released later.

### [H-1] FIRST TokenggAVAX DEPOSIT EXPLOIT CAN BREAK SHARE CALCULATION

`convertToShares` function follow the formula: 
      
      return supply == 0 ? assets : assets.mulDivDown(supply, totalAssets());

The share price always return 1:1 with asset token. If everything work normally, share price will slowly increase with time to 1:2 or 1:10 as more rewards coming in.

## Proof of Concept

Right after TokenggAVAX contract creation, during first cycle, any user can deposit 1 share set totalSupply = 1. And transfer token to vault to inflate totalAssets() before rewards kick in. (Basically, pretend rewards themselves before anyone can deposit in to get much better share price.)

This can inflate base share price as high as 1:1e18 early on, which force all subsequence deposit to use this share price as base.

## Impact

New TokenggAVAX vault share price can be manipulated right after creation. Which give early depositor greater share portion of the vault during the first cycle.

## Recommended Mitigation Steps

This exploit is unique to contract similar to ERC4626. It only works if starting supply equals very small number and rewards cycle is very short. Or everyone withdraws, total share supply become close to 0.

This can be easily fix by making sure someone always deposited first so totalSupply become high enough that this exploit become irrelevant. Unless in unlikely case someone made arbitrage bot watching vault factory contract.
Just force deposit early token during vault construction as last resort.

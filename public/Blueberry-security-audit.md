# Blueberry

## Medium and High Risk Issues
| Count | Explanation |
|:--:|:-------|
| [M-1] | User can Loss Funds because of `burn` logic of `WERC20` if ERC20 Token used is `Rabase Token` |
| [M-2] | Used Depreciated `safeApprove` and For some tokens that don't support approve `type(uint256).max` amount it will not work. |
| [M-3] | The contract should use approve(0) before approve |
| [M-4] | All `initialize` methods can be Frontrun because of lack of access control |
| [M-5] | `doCutDepositFee` and `doCutWithdrawFee` doesn't take `FEES ON TRANSFER TOKENS` in account |
| [H-1] | "First Deposit Bug" in cToken (CompoundV2 Fork) can lead to Stealing of funds for first depositor in `SoftVault` |

| Issue Type | Count |
|:--:|:--:|
| Total Medium Risk Issues | 5 |
| Total High Risk Issues | 1 |

### [M-1] User can Loss Funds because of `burn` logic of `WERC20` if ERC20 Token used is `Rabase Token`

## Summary

Use of Rebase Tokens can lead to fund loss to user and contract will lock that excess fund forever.

## Vulnerability Detail

Rebasing tokens are tokens that have each holder’s balanceof() increase over time. Aave aTokens are an example of such tokens. If such tokens are used during mint then the amount of wrapped tokens will be `x` at the time of minting. But over time, it can increase from `x` to `x + y` but during burn, user will only be able to claim `x` tokens leaving `y` forever in the contract.

## Impact

If rebasing tokens are used as the underlying token, rewards accrue to the contract and cannot be withdrawn by either the contract or the user, and remain locked forever.

## Code Snippet

```solidity
File: WERC20.sol

    function mint(address token, uint256 amount)
        external
        override
        nonReentrant
    {
        uint256 balanceBefore = IERC20Upgradeable(token).balanceOf(
            address(this)
        );
        IERC20Upgradeable(token).safeTransferFrom(
            msg.sender,
            address(this),
            amount
        );
        uint256 balanceAfter = IERC20Upgradeable(token).balanceOf(
            address(this)
        );
        _mint(
            msg.sender,
            uint256(uint160(token)),
            balanceAfter - balanceBefore,
            ""
        );
    }

    function burn(address token, uint256 amount)
        external
        override
        nonReentrant
    {
        _burn(msg.sender, uint256(uint160(token)), amount);
        IERC20Upgradeable(token).safeTransfer(msg.sender, amount);
    }

```
[Link to Code](https://github.com/sherlock-audit/2023-02-blueberry-Breeje16/blob/main/contracts/wrapper/WERC20.sol#L47-L82)

## Tool used

Manual Review

## Recommendation

Track total amounts currently deposited and allow users to withdraw excess on a pro-rata basis.

### [M-2] Used Depreciated `safeApprove` + For some tokens that don't support approve `type(uint256).max` amount it will not work.

## Summary

Use of `safeApprove` has been depreciated and not recommeded to use. For some tokens that don't support approve `type(uint256).max` amount it will not work.

## Vulnerability Detail

There are tokens that doesn't support approve spender type(uint256).max amount. So the `safeApprove` will not work for some tokens like `UNI` or `COMP` who will revert when approve type(uint256).max amount.

## Impact

Tokens that don't support approve type(uint256).max amount could not be transferred.

## Code Snippet

```solidity
File: SoftVault.sol

55:     _uToken.safeApprove(address(_cToken), type(uint256).max);  

```
[Link to Code](https://github.com/sherlock-audit/2023-02-blueberry-Breeje16/blob/main/contracts/vault/SoftVault.sol#L55)

```solidity
File: BasicSpell.sol

49:     IERC20Upgradeable(token).safeApprove(spender, type(uint256).max); 

```
[Link to Code](https://github.com/sherlock-audit/2023-02-blueberry-Breeje16/blob/main/contracts/spell/BasicSpell.sol#L49)

```solidity
File: WIchiFarm.sol

100:     IERC20Upgradeable(lpToken).safeApprove(

133:     ICHIv1.safeApprove(address(ICHI), ichiRewards);

```
[Link to Code](https://github.com/sherlock-audit/2023-02-blueberry-Breeje16/blob/main/contracts/wrapper/WIchiFarm.sol)

## Tool used

Manual Review

## Recommendation

Use `safeIncreaseAllowance()` and `safeDecreaseAllowance()` instead


### [M-3] The contract should use approve(0) before approve

## Summary

`BlueBerryBank` uses `SafeERC20Upgradeable` for `IERC20Upgradeable` which has an issue in approve.

## Vulnerability Detail

Some tokens do not work when changing the allowance from an existing non-zero allowance value. They must first be approved by zero and then the actual allowance must be approved.

## Impact

Silent Failure of approve.

## Code Snippet

```solidity
File: BlueBerryBank.sol

647:    IERC20Upgradeable(token).approve(bank.softVault, amount);

652:    IERC20Upgradeable(token).approve(bank.hardVault, amount);

882:    IERC20Upgradeable(token).approve(bank.cToken, amountCall);

```
[Link to code](https://github.com/sherlock-audit/2023-02-blueberry-Breeje16/blob/main/contracts/BlueBerryBank.sol)

## Tool used

Manual Review

## Recommendation

Approve with a zero amount first before setting the actual amount.

---


### [M-4] All `initialize` methods can be Frontrun because of lack of access control

## Summary

There is no Access control in `initialize()` method. So anyone can frontrun the transaction and call that on deployer's behalf to gain access.

## Vulnerability Detail

If the `initializer` is not executed in the same transaction as the constructor, a malicious user can front-run the `initialize()` call, forcing the contract to be redeployed.

## Impact

Contract will have to be redeployed.

## Code Snippet

```solidity
File: BlueBerryBank.sol

94:   function initialize(IOracle _oracle, IProtocolConfig _config)

```
[Link to Code](https://github.com/sherlock-audit/2023-02-blueberry-Breeje16/blob/main/contracts/BlueBerryBank.sol#L94)

```solidity
File: ProtocolConfig.sol

28:   function initialize(address treasury_) external initializer {

```
[Link to Code](https://github.com/sherlock-audit/2023-02-blueberry-Breeje16/blob/main/contracts/ProtocolConfig.sol#L28)

```solidity
File: CoreOracle.sol

31:   function initialize() external initializer {

```
[Link to Code](https://github.com/sherlock-audit/2023-02-blueberry-Breeje16/blob/main/contracts/oracle/CoreOracle.sol#L31)

```solidity
File: IchiVaultSpell.sol

59:   function initialize(

```
[Link to Code](https://github.com/sherlock-audit/2023-02-blueberry-Breeje16/blob/main/contracts/spell/IchiVaultSpell.sol#L59)

```solidity
File: HardVault.sol

36:   function initialize(IProtocolConfig _config) external initializer {

```
[Link to Code](https://github.com/sherlock-audit/2023-02-blueberry-Breeje16/blob/main/contracts/vault/HardVault.sol#L36)

```solidity
File: SoftVault.sol

41:   function initialize(

```
[Link to Code](https://github.com/sherlock-audit/2023-02-blueberry-Breeje16/blob/main/contracts/vault/SoftVault.sol#L41)

```solidity
File: WERC20.sol

15:   function initialize() external initializer {

```
[Link to Code](https://github.com/sherlock-audit/2023-02-blueberry-Breeje16/blob/main/contracts/wrapper/WERC20.sol#L15)

```solidity
File: WIchiFarm.sol

30:   function initialize(

```
[Link to Code](https://github.com/sherlock-audit/2023-02-blueberry-Breeje16/blob/main/contracts/wrapper/WIchiFarm.sol#L30)

## Tool used

Manual Review

## Recommendation

Add a control access modifier such that only the owner can call `initialize()` method.

### [M-5] `doCutDepositFee` and `doCutWithdrawFee` doesn't take `FEES ON TRANSFER TOKENS` in account

## Summary

Fees on transfer case not considered in couple of methods.

## Vulnerability Detail

In `BlueBerryBank`, `doCutDepositFee` and `doCutWithdrawFee` methods transfers the fees but doesn't handle fees on transfer condition.

## Impact

The passes value of deducting the fees can be incorrect.

## Code Snippet

```solidity
File: BlueBerryBank.sol

    function doCutDepositFee(address token, uint256 amount)
        internal
        returns (uint256)
    {
        if (config.treasury() == address(0)) revert NO_TREASURY_SET();
        uint256 fee = (amount * config.depositFee()) / DENOMINATOR;
        IERC20Upgradeable(token).safeTransfer(config.treasury(), fee);
        return amount - fee;
    }

    function doCutWithdrawFee(address token, uint256 amount)
        internal
        returns (uint256)
    {
        if (config.treasury() == address(0)) revert NO_TREASURY_SET();
        uint256 fee = (amount * config.withdrawFee()) / DENOMINATOR;
        IERC20Upgradeable(token).safeTransfer(config.treasury(), fee);
        return amount - fee;
    }

```
[Link to code](https://github.com/sherlock-audit/2023-02-blueberry-Breeje16/blob/main/contracts/BlueBerryBank.sol#L892-L910)

## Tool used

Manual Review

## Recommendation

Recommend to check balance before and after transfer and return the value accordingly

### [H-1] "First Deposit Bug" in cToken (CompoundV2 Fork) can lead to Stealing of funds for first depositor in `SoftVault`

## Summary

cToken of CompoundV2 contains the "First Deposit Bug" (Credit to Akshay Srivastav for finding this Bug) which is there in all corresponding V2 Forks of it. It can lead to an attack where funds of First Depositor can be easily stolen. This attacker can be repeated. Below contains Complete breakdown of the Bug published by Akshay.

## Vulnerability Detail

As per the implementation of CToken contract, there exist two cases for CToken amount calculation:

1. First deposit - when CToken.totalSupply() is 0.
2. All subsequent deposits.

Here is the actual CToken code (extra code and comments clipped for better reading):
```solidity
  function exchangeRateStoredInternal() virtual internal view returns (uint) {
      uint _totalSupply = totalSupply;
      if (_totalSupply == 0) {
          return initialExchangeRateMantissa;
      } else {
          uint totalCash = getCashPrior();
          uint cashPlusBorrowsMinusReserves = totalCash + totalBorrows - totalReserves;
          uint exchangeRate = cashPlusBorrowsMinusReserves * expScale / _totalSupply;
          return exchangeRate;
      }
  }

  function mintFresh(address minter, uint mintAmount) internal {
      // ...
      Exp memory exchangeRate = Exp({mantissa: exchangeRateStoredInternal()});

      uint actualMintAmount = doTransferIn(minter, mintAmount);

      uint mintTokens = div_(actualMintAmount, exchangeRate);

      totalSupply = totalSupply + mintTokens;
      accountTokens[minter] = accountTokens[minter] + mintTokens;
      // ...
  }
```

The above implementation contains a critical bug which can be exploited to steal funds of initial depositors of a freshly deployed CToken contract.

As the exchange rate is dependent upon the ratio of CToken's totalSupply and underlying token balance of CToken contract, the attacker can craft transactions to manipulate the exchange rate.

Steps to attack:

1. Once the CToken has been deployed and added to the lending protocol, the attacker mints the smallest possible amount of CTokens.
2. Then the attacker does a plain `underlying` token transfer to the CToken contract, artificially inflating the `underlying.balanceOf(CToken)` value.
Due to the above steps, during the next legitimate user deposit, the `mintTokens` value for the user will become less than 1 and essentially be rounded down to 0 by Solidity. Hence the user gets 0 CTokens against his deposit and the CToken's entire supply is held by the Attacker.
3. The Attacker can then simply `reedem` his CToken balance for the entire `underlying` token balance of the CToken contract.

The same steps can be performed again to steal the next user's deposit.

It should be noted that the attack can happen in two ways:

* The attacker can simply execute the Step 1 and 2 as soon as the CToken gets added to the lending protocol.
* The attacker watches the pending transactions of the network and frontruns the user's deposit transaction by executing Step 1 and 2 and then backruns it with Step 3.

So whenever user deposits in `SoftVault`, the asset is deposited on compound which can be stolen. So when the user tries to withdraw, he/she will fail to withdraw any asset as `redeem` will return 0 and transaction will revert for user with `REDEEM_FAILED` Error.

## Impact

A sophisticated attack can impact all user deposits until the lending protocols owners and users are notified and contracts are paused. Since this attack is a replicable attack it can be performed continuously to steal the deposits of all depositors that try to deposit into the CToken contract through `SoftVault`.

The loss amount will be the sum of all deposits done by users into the CToken multiplied by the underlying token's price.

Suppose there are `10` users and each of them tries to deposit `1,000,000` underlying tokens into the CToken contract. Price of underlying token is `$1`.

`Total loss (in $) = $10,000,000`

## Code Snippet

```solidity
File: SoftVault.sol

    function deposit(uint256 amount)
        external
        override
        nonReentrant
        returns (uint256 shareAmount)
    {
        if (amount == 0) revert ZERO_AMOUNT();
        uint256 uBalanceBefore = uToken.balanceOf(address(this));
        uToken.safeTransferFrom(msg.sender, address(this), amount);
        uint256 uBalanceAfter = uToken.balanceOf(address(this));

        uint256 cBalanceBefore = cToken.balanceOf(address(this));
        if (cToken.mint(uBalanceAfter - uBalanceBefore) != 0)
            revert LEND_FAILED(amount);
        uint256 cBalanceAfter = cToken.balanceOf(address(this));

        shareAmount = cBalanceAfter - cBalanceBefore;
        _mint(msg.sender, shareAmount);

        emit Deposited(msg.sender, amount, shareAmount);
    }

```
[Link to Code](https://github.com/sherlock-audit/2023-02-blueberry-Breeje16/blob/main/contracts/vault/SoftVault.sol#L67-L87)

## Tool used

Manual Review

## Recommendation

The fix to prevent this issue would be to enforce a minimum deposit that cannot be withdrawn. This can be done by minting small amount of CToken units to `0x00` address on the first deposit.

```solidity

  function mintFresh(address minter, uint mintAmount) internal {
      // ...
      Exp memory exchangeRate = Exp({mantissa: exchangeRateStoredInternal()});

      uint actualMintAmount = doTransferIn(minter, mintAmount);

      uint mintTokens = div_(actualMintAmount, exchangeRate);

      /// THE FIX
      if (totalSupply == 0) {
          totalSupply = 1000;
          accountTokens[address(0)] = 1000;
          mintTokens -= 1000;
      }

      totalSupply = totalSupply + mintTokens;
      accountTokens[minter] = accountTokens[minter] + mintTokens;
      // ...
  }

```

Instead of a fixed `1000` value an admin controlled parameterized value can also be used to control the burn amount on a per CToken basis.

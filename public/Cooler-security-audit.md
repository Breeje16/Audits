# Cooler

## Medium and High Risk Issues
| Count | Explanation |
|:--:|:-------|
| [M-1] | POOR CONTRACT ARCHITECTURE LEADING TO REDUNDENT INSTANCES |
| [M-2] | EDGE CASE NOT HANDLED WHERE BORROWER CAN USE INTEREST FREE LOAN TILL LENDER CALLS DEFAULTED METHOD |
| [M-3] | SOME AMOUNT OF COLLATERAL CAN BE STUCK IN ESCROW |
| [H-1] | UNCHECKED RETURN VALUES FROM `transfer` COULD LEAD TO STEALING OF FUNDS. |
| [H-2] | BORROWER CAN EXPLOIT `roll` METHOD WITH FRONTRUNNING |

| Issue Type | Count |
|:--:|:--:|
| Total Medium Risk Issues | 3 |
| Total High Risk Issues | 2 |

### [M-1] Poor Contract Architecture leading to redundent Instances

## Summary

In `Cooler` contract, `CoolerFactory` contract has been imported and its instance has been created and used. Likewise In `CoolerFactory` contract too, `Cooler` contract has been imported and its instance has been used.

## Vulnerability Detail

The issue with this design is that both the contract will have more than 1 instance of the same code.

## Impact

This multiple contracts with same name leads to Collision in Code which can lead to unexpected behavior in future.

## Code Snippet

```solidity
File: src/Cooler.sol

5:      import "./Factory.sol";

```
[Link to Code](https://github.com/sherlock-audit/2023-01-cooler-Breeje16/blob/main/src/Cooler.sol#L5)

```solidity
File: src/Factory.sol

4:      import "./Cooler.sol";

```
[Link to Code](https://github.com/sherlock-audit/2023-01-cooler-Breeje16/blob/main/src/Factory.sol#L4)


## Tool used

Surya Graph + Manual Review

Surya Graph clearly showed redundent dependencies.

## Recommendation

It is recommended to linearized the dependency graph so that it can avoid collision and contracts get imported exactly once.

Best way to achieve this will be:

1. As in `Cooler` contract, `CoolerFactory` is used only for emitting events, we can remove `CoolerFactory` import and instance of it.
2. Instead shift the events [`Request`, `Rescind` and `Clear`] from `CoolerFactory` to `Cooler` contract and directly call them in place of calling them through `CoolerFactory`.

This can linearize the dependency graph and prevent code collisions.

### [M-2] EDGE CASE NOT HANDLED WHERE BORROWER CAN USE INTEREST FREE LOAN TILL LENDER CALLS DEFAULTED METHOD

## Summary

The Code by design allows the lender to call the `defaulted` method if case of borrower not paying back the loan in time.

## Vulnerability Detail

Borrower can decide not to pay the loan untill the Lender send the transaction of `defaulted` in the mempool. Borrower can then frontrun that transaction to pay the loan back before `defaulted` transaction goes through.

## Impact

Borrower can have interest free loan from the expiry time till the lender calls `defaulted` method.

## Code Snippet

```solidity
File: Cooler.sol

2:      pragma solidity ^0.8.0;

```
[Link to Code](https://github.com/sherlock-audit/2023-01-cooler-Breeje16/blob/main/src/Cooler.sol#L198-L207)

## Tool used

Manual Review

## Recommendation

Add a condition in `repay` to handle this edge case where Borrower will have to pay time adjusted amount in case of paying after expiry time.

### [M-3] SOME AMOUNT OF COLLATERAL CAN BE STUCK IN ESCROW

## Summary

In `repay` method, the collateral to be released is calculated by a mathematical calculation with include division.

## Vulnerability Detail

In EVM, as there are no Floating points, the `decollateralized` value will always take the floor value of the calculation ignore the latter part. If borrower is paying back in multiple transactions, he/she will loss some amount of collateral because of this.

## Impact

Loss of Collateral for Borrower despite paying back the Loan Amount.

## Code Snippet

```solidity
File: Cooler.sol

    function repay (uint256 loanID, uint256 repaid) external {
        Loan storage loan = loans[loanID];

        if (block.timestamp > loan.expiry) 
            revert Default();
        
        uint256 decollateralized = loan.collateral * repaid / loan.amount;

        if (repaid == loan.amount) delete loans[loanID];
        else {
            loan.amount -= repaid;
            loan.collateral -= decollateralized;
        }

        debt.transferFrom(msg.sender, loan.lender, repaid);
        collateral.transfer(owner, decollateralized);
    }

```
[Link to Code](https://github.com/sherlock-audit/2023-01-cooler-Breeje16/blob/main/src/Cooler.sol#L108-L124)

## Tool used

Manual Review

## Recommendation

When the repaid amount is equal to loan amount, transfer all the remaining collateral in the escrow to the Borrower. 

Can use a new state variable to keep a track on this which will make sure that no amount of collateral is stuck inside escrow.

### [H-1] Unchecked return values from `transfer` could lead to stealing of funds.

## Summary

In the contracts `Cooler`, the return values of ERC20 `transfer` is not checked to be true, which could be false if the transferred tokens are not ERC20-compliant (e.g., USDT).

## Vulnerability Detail

In such case, the transfer fails without being noticed by the calling contract.

## Impact

This can lead to Stealing of Funds of Lender from the Attacker.

## Proof of Concept

Following Attack vector can be leveraged by the Attacker:

1. Call `generate` with USDT as collateral and debt as any other commonly used token.
2. Call `request` method with all the details.
3. Here the `transferFrom` will fails with false as return value indicating the failure but that return value is not validated in `Cooler`.
4. Lender will clear thinking in case of default he/she will get the collateral.
5. This way attacker will get the debt tokens without giving any collateral.

## Code Snippet

```solidity
File: Cooler.sol

85:      collateral.transferFrom(msg.sender, address(this), collateralFor(amount, loanToCollateral));

```
[Link to Code](https://github.com/sherlock-audit/2023-01-cooler-Breeje16/blob/main/src/Cooler.sol#85)

## Tool used

Manual Review

## Recommendation

Recommend using the `SafeERC20` library implementation from Openzeppelin and call `safeTransfer` or `safeTransferFrom` when transferring ERC20 tokens.

### [H-2] BORROWER CAN EXPLOIT `roll` METHOD WITH FRONTRUNNING

## Summary

`clear` method uses default value `rollable` as true and `roll` function uses the `rollable` value as validation to extend the loan whose duration can be anything.

## Vulnerability Detail

By Frontrunning the `roll` method, Borrower can extend the `expiry` by whatever `duration` he/she wants.

## Impact

This situation can lead to Borrower getting extention for any duration which he/she sets without consent of the lender.

## Proof of Concept

Consider the following situation:

1. Alice after using `generate` method, `request` a loan.
2. Bob as a lender wants to give out loan. He does that using `clear` method.
3. Now the loan has been set with `rollable` value as `true` by default.
4. Bob wants to change it by using `toggleRoll`. So he has to do it in different transaction.
5. Alice can Frontrun this transaction and call `roll` method and can extend `expiry` to whatever time he wants and Bob can no control over it.

## Code Snippet

```solidity
File: Cooler.sol

129:  function roll (uint256 loanID) external {
          Loan storage loan = loans[loanID];
          Request memory req = loan.request;

          if (block.timestamp > loan.expiry) 
              revert Default();

          if (!loan.rollable)
              revert NotRollable();

          uint256 newCollateral = collateralFor(loan.amount, req.loanToCollateral) - loan.collateral;
          uint256 newDebt = interestFor(loan.amount, req.interest, req.duration);

          loan.amount += newDebt;
          loan.expiry += req.duration;
          loan.collateral += newCollateral;
          
          collateral.transferFrom(msg.sender, address(this), newCollateral);
147:  }

162:  function clear (uint256 reqID) external returns (uint256 loanID) {
          Request storage req = requests[reqID];

          factory.newEvent(reqID, CoolerFactory.Events.Clear);

          if (!req.active) 
              revert Deactivated();
          else req.active = false;

          uint256 interest = interestFor(req.amount, req.interest, req.duration);
          uint256 collat = collateralFor(req.amount, req.loanToCollateral);
          uint256 expiration = block.timestamp + req.duration;

          loanID = loans.length;
          loans.push(
              Loan(req, req.amount + interest, collat, expiration, true, msg.sender)
          );
          debt.transferFrom(msg.sender, owner, req.amount);
180:  }

```
[Link to Code](https://github.com/sherlock-audit/2023-01-cooler-Breeje16/blob/main/src/Cooler.sol#L129-L180)

## Tool used

Manual Review

## Recommendation

Mitigating this Vulnerability can be done in 2 ways:

1. Add an extra parameter where lender can set `rollable` value in `clear` method. With `rollable` value directly set as `false` in same transaction of releasing debt token, Borrower can no longer use frontrunning attack vector.
2. Add a delay in executing `roll` from the time it gets called first. This can give lender enough time to agree or not with loan getting rolled.


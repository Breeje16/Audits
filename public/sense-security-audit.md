# Sense

## Medium and High Risk Issues

| Count | Explanation |
|:--:|:-------|
| [M-1] | Missing safeApprove(0) before using safeApprove for any approval which can lead to DOS |
| [M-2] | Missing deadline checks allow pending transactions to be maliciously executed |
| [H-1] | Use of payable.transfer can lead to DOS and Freezing of funds |

| Issue Type | Count |
|:--:|:--:|
| Total Medium Risk Issues | 2 |
| Total High Risk Issues | 1 |

### [M-1] Missing safeApprove(0) before using safeApprove for any approval which can lead to DOS

## Summary

safeApprove not used correctly leading to DOS.

## Vulnerability Detail

OpenZeppelin’s `safeApprove()` will revert if the account already is approved and the new `safeApprove()` is done with a non-zero value.

```solidity

    function safeApprove(
        IERC20 token,
        address spender,
        uint256 value
    ) internal {
        // safeApprove should only be called when setting an initial allowance,
        // or when resetting it to zero. To increase and decrease it, use
        // 'safeIncreaseAllowance' and 'safeDecreaseAllowance'
        require(
            (value == 0) || (token.allowance(address(this), spender) == 0),
            "SafeERC20: approve from non-zero to non-zero allowance"
        );
        _callOptionalReturn(token, abi.encodeWithSelector(token.approve.selector, spender, value));
    }

```

## Impact

DOS

## Code Snippet

```solidity
File: Periphery.sol

128:    ERC20(stake).safeApprove(address(divider), stakeSize);

509:    ERC20(assetIn).safeApprove(address(balancerVault), amountIn);

861:    liq.tokens[i].safeApprove(address(balancerVault), liq.amounts[i]);

```
[Link to Code](https://github.com/sherlock-audit/2023-03-sense-Breeje16/blob/main/sense-v1/pkg/core/src/Periphery.sol#L128)

## Tool used

Manual Review

## Recommendation

Use safeApprove(0) first before using safeApprove for any value.

### [M-2] Missing deadline checks allow pending transactions to be maliciously executed

## Summary

The `Periphery` contract does not allow users to submit a deadline for their action. This missing feature enables pending transactions to be maliciously executed at a later point.

## Vulnerability Detail

Here's how MEV Bot can exploit it without deadline check:

1. The swap transaction is still pending in the mempool. The price of 1 token has gone up significantly since the transaction was signed, meaning Alice would receive a lot more second token when the swap is executed. But that also means that her minimum Output Amount value is outdated and would allow for significant slippage.
2. A MEV bot detects the pending transaction. Since the outdated minimum Output Amount now allows for high slippage, the bot sandwiches Alice, resulting in significant profit for the bot and significant loss for Alice.

This is the reason any critical functionality where user is going to interact with the pool should be passed with a deadline of execution. As pool balance and result out of interact with pool is time dependent, there should not be any attack vector left for attackers to manipulate. By setting deadline such attack vector will be mitigated.

## Impact

Loss of funds.

## Code Snippet

Functions which can be affected:

```solidity
File: Periphery.sol

178:    function swapForPTs(

204:    function swapForYTs(

240:    function swapPTs(

263:    function swapYTs(

275:    function swapYTsForTarget(

325:    function addLiquidity(

371:    function removeLiquidity(

```
[Link to Code](https://github.com/sherlock-audit/2023-03-sense-Breeje16/blob/main/sense-v1/pkg/core/src/Periphery.sol#L178)

## Tool used

Manual Review

## Recommendation

Introduce a deadline parameter to the mentioned functions.

### [H-1] Use of payable.transfer can lead to DOS and Freezing of funds

## Summary

Unsafe Transfer are used in the following methods:

* `issue()`
* `_removeLiquidity()`
* `_fillQuote()`
* `_transfer()`

which can lead to DOS in future.

## Vulnerability Detail

Transfer has hard coded gas budget and can fail when the user is a smart contract. Whenever the user either fails to implement the payable fallback function or cumulative gas cost of the function sequence invoked on a native token transfer exceeds 2300 gas consumption limit the native tokens sent end up undelivered and the corresponding user funds return functionality will fail each time.

This is the reason in future if any transfer exceeds 2300 gas consumption limit, the function will always revert.

As `transfer` call is used in critical function like `removeLiquidity`, it can lead to permanent freezing of funds as the contract will not be able to remove any liquidity. These freezing of funds makes it a High Severity Issue.

## Impact

DOS of critical functionalities like `removeLiquidity` leading to freezing of funds.

## Code Snippet

```solidity
File: Periphery.sol

419:    ERC20(divider.pt(adapter, maturity)).transfer(receiver, uBal); // Send PTs to the receiver
420:    ERC20(divider.yt(adapter, maturity)).transfer(receiver, uBal); // Send YT to the receiver

744:    if (ptBal > 0) ERC20(pt).transfer(receiver, ptBal);

930:    payable(msg.sender).transfer(refundAmt);

1013:   address(token) == ETH ? payable(receiver).transfer(amt) : token.safeTransfer(receiver, amt);

```
[Link to Code](https://github.com/sherlock-audit/2023-03-sense-Breeje16/blob/main/sense-v1/pkg/core/src/Periphery.sol#L744)

## Tool used

Manual Review

## Recommendation

Using low-level call.value(amount) with the corresponding result check or using the OpenZeppelin’s Address.sendValue is advised.


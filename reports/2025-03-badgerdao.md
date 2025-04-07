# (HIGH) Lack of decimal adjustment in `EbtcBSM` causes incorrect exchange rates between `eBTC` and `BTC` assets with different decimals

## Summary

As stated in the protocol's `README.md` file, the `EbtcBSM` contract facilitates bi-directional exchanges between `eBTC` and other `BTC`-denominated assets with no slippage. The `eBTC` token has 18 decimals; however, some `BTC`-denominated assets use different decimal places, such as:

- [wBTC](https://etherscan.io/address/0x2260fac5e5542a773aa44fbcfedf7c193bc2c599#readContract) - 8 decimals
- [cbBTC](https://etherscan.io/address/0xcbb7c0000ab88b473b1f5afd9ef808440eed33bf#readProxyContract) - 8 decimals

As a result, users will incur significant losses when selling these types of `BTC`-denominated assets for `eBTC`. For example, if a user sells `1 cbBTC` (which is `1e8`), they will not receive `1 eBTC` (which is `1e18`) in return but rather `0.0000000001 eBTC`. Conversely, users will gain an unfair advantage when buying these assets with `eBTC`.

## Context

- [EbtcBSM.sol#L172-L203](https://github.com/ebtc-protocol/ebtc-bsm/blob/3a47029271b1b00635b0a4c1bbd0eef91115a300/src/EbtcBSM.sol#L172-L203)
- [EbtcBSM.sol#L206-L242](https://github.com/ebtc-protocol/ebtc-bsm/blob/3a47029271b1b00635b0a4c1bbd0eef91115a300/src/EbtcBSM.sol#L206-L242)

## Impact

High. When exchanging `BTC`-denominated assets with different decimal places, users will experience substantial losses when selling and unfair profits when buying, due to the lack of decimal adjustment.

## Likelihood

High. Several `BTC`-denominated assets have different decimal places than `eBTC`, making this issue highly probable.

## Proof of Concept

```solidity
function test_AssetWithDifferentDecimals() public {
    // Initial state
    uint256 bobTBtcBalanceBefore = tBtc.balanceOf(bob);
    uint256 bobEBtcBalanceBefore = eBtc.balanceOf(bob);

    // Sell asset
    uint256 tBtcAmountIn = 1e8;
    vm.prank(bob);
    bsm.sellAsset(tBtcAmountIn, bob, tBtcAmountIn);

    // Asserting that Bob has the correct asset balance after selling
    assertEq(tBtc.balanceOf(bob), bobTBtcBalanceBefore - tBtcAmountIn);
    // Asserting that Bob has received 1e8 eBTC, instead of 1e18
    assertEq(eBtc.balanceOf(bob), bobEBtcBalanceBefore + tBtcAmountIn);

    console.log("tBTC BALANCE BEFORE", bobTBtcBalanceBefore);
    console.log("tBTC BALANCE AFTER", tBtc.balanceOf(bob));
    console.log("eBTC BALANCE BEFORE", bobEBtcBalanceBefore);
    console.log("eBTC BALANCE AFTER", eBtc.balanceOf(bob));
}
```

**Logs:**

```bash
[PASS] test_AssetWithDifferentDecimals() (gas: 175294)
Logs:
  tBTC BALANCE BEFORE    : 2000000000
  tBTC BALANCE AFTER     : 1900000000
  eBTC BALANCE BEFORE    : 10000000000000000000
  eBTC BALANCE AFTER     : 10000000000100000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.75ms (472.92µs CPU time)
```

### Instructions

1. Under `test/`, create an `audit/` folder.
2. Inside `audit/`, add the [`BaseTest.t.sol`](https://gist.github.com/zeGarcao/3707f1904c1748711f7d7ad3e6716c46#file-basetest-t-sol) and [`Main.t.sol`](https://gist.github.com/zeGarcao/3707f1904c1748711f7d7ad3e6716c46#file-main-t-sol) files.
3. Inside `audit/`, create a `mocks/` folder.
4. Inside `mocks/`, add the [`ERC20Mock.sol`](https://gist.github.com/zeGarcao/3707f1904c1748711f7d7ad3e6716c46#file-erc20mock-sol) file.
5. Run the following command in your terminal:

```bash
forge test --mt test_AssetWithDifferentDecimals -vvv
```

## Recommendation

Implement proper decimal adjustments to ensure accurate conversions between `eBTC` and `BTC`-denominated assets with different decimal places.

# (LOW) Reentrancy in the `sellAsset` function of `EbtcBSM` allows users to bypass mint rate limiting constraint

## Summary

As stated in the protocol's `README.md` file, the `EbtcBSM` contract facilitates bi-directional exchanges between `eBTC` and other `BTC`-denominated assets with no slippage. However, some of these assets may not fully comply with the ERC20 standard. For example, [`imBTC`](https://etherscan.io/token/0x3212b29e33587a00fb1c83346f5dbfa69a458923/), which follows the ERC777 standard, introduces reentrancy risks. Integrating such assets makes `EbtcBSM` vulnerable to reentrancy attacks, allowing malicious actors to bypass the mint rate limiting constraint.

### Exceptions to the severity matrix - Weird ERC20 Tokens

I acknowledge [Cantina's exceptions regarding weird ERC20 tokens](https://docs.cantina.xyz/cantina-docs/cantina-competitions/judging-process/finding-severity-criteria#exceptions-to-the-severity-matrix), which state:

> Unless explicitly mentioned in the contest details, or **otherwise critical for the protocol**, the classes of issues listed below will have their severity capped to the respective severity

However, I believe this exception does not apply in this context, particularly because:

- The integration of `BTC`-denominated assets is critical for the protocol.
- The protocol does not explicitly state that non-standard ERC20 tokens are excluded, making their integration as likely as standard tokens.
- We must assume that the protocol can and probably will integrate such assets.
- There are actual `BTC`-denominated assets that follow the ERC777 standard, making this scenario realistic rather than speculative.
- Integrating such assets breaks the protocol's core functionality.

### Example

Before going through the example, let's define the initial state:

- Total `eBTC` supply: 100 `eBTC`
- Total `eBTC` minted by `EbtcBSM`: 9 `eBTC`
- Mint rate limiting constraint: 10% of total supply (10 `eBTC`)
- Integrated asset: `imBTC`
- Configured fees: None
  
With this initial state:

1. Bob calls the `sellAsset` function to sell 1 `imBTC`.
2. The rate limiting constraint is checked and passed since the mint cap (10 `eBTC`) is not exceeded.
3. `EbtcBSM` triggers the `transferFrom` function of `imBTC`, which in turn invokes the `tokensToSend` callback on Bob's malicious contract before transferring the tokens.
4. Bob's malicious contract calls `sellAsset` again to sell 1 more `imBTC`.
5. The rate limiting constraint is checked and passed again, as no `eBTC` has been minted yet.
6. Bob successfully manipulates `EbtcBSM` into minting more tokens than expected.

## Context

- [EbtcBSM.sol#L172-L203](https://github.com/ebtc-protocol/ebtc-bsm/blob/3a47029271b1b00635b0a4c1bbd0eef91115a300/src/EbtcBSM.sol#L172-L203)

## Impact

High. This issue breaks core protocol functionality by allowing users to bypass `EbtcBSM`'s mint rate cap for `eBTC`, causing a failure in fundamental protocol operations.

## Likelihood

Medium. While any user can exploit this issue, it only affects tokens following the ERC777 standard.

## Proof of Concept

```solidity
function test_BypassRateConstraintWithERC777() public {
    vm.startPrank(alice);
    bsm.sellAsset(imBtc.balanceOf(alice), alice, 0);
    vm.stopPrank();

    vm.startPrank(authorizedUser);
    bsm.sellAsset(imBtc.balanceOf(authorizedUser), authorizedUser, 0);
    vm.stopPrank();

    uint256 totalEbtcSupply = observer.observe();
    uint256 maxMint = (totalEbtcSupply * 1_000) / BPS_BASE;

    console.log("BSM MAX MINT       :", maxMint);

    vm.startPrank(bob);
    BypassRateConstraint bypassRateConstraint =
        new BypassRateConstraint(address(bsm), address(imBtc), address(escrow));
    imBtc.transfer(address(bypassRateConstraint), imBtc.balanceOf(bob));
    bypassRateConstraint.attack();
    vm.stopPrank();

    console.log("BSM eBTC MINTED    :", bsm.totalMinted());
}
```

> **Note:** For demonstration purposes, `eBTC` was deployed with 8 decimal places to match `imBTC`, as `eBTC` currently lacks proper decimal adjustments.

**Logs:**

```bash
Ran 1 test for test/audit/ForkMain.t.sol:ForkMain
[PASS] test_BypassRateConstraintWithERC777() (gas: 959409)
Logs:
  BSM MAX MINT       : 1100000000
  BSM eBTC MINTED    : 1500000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 476.25ms (12.22ms CPU time)
```

### Instructions

1. Under the project's root folder, add a `.env` file containing:

```
MAINNET_RPC_URL="YOUR RPC URL"
```

2. Add the following configuration in the `foundry.toml` file:

```toml
[rpc_endpoints]
mainnet = "${MAINNET_RPC_URL}"
```

3. Under `test/`, create an `audit/` folder.
4. Inside `audit/`, add the [`ForkBaseTest.t.sol`](https://gist.github.com/zeGarcao/3fc8e1a1c536133b21de2224851fb141#file-forkbasetest-t-sol), [`ForkMain.t.sol`](https://gist.github.com/zeGarcao/3fc8e1a1c536133b21de2224851fb141#file-forkmain-t-sol), and [`BypassRateConstraint.sol`](https://gist.github.com/zeGarcao/3fc8e1a1c536133b21de2224851fb141#file-bypassrateconstraint-sol) files.
5. Inside `audit/`, create a `mocks/` folder.
6. Inside `mocks/`, add the [`ERC20Mock.sol`](https://gist.github.com/zeGarcao/3fc8e1a1c536133b21de2224851fb141#file-erc20mock-sol) file.
7. Run the following command in your terminal:

```bash
forge test --mt test_BypassRateConstraintWithERC777 -vvv
```

## Recommendation

Modify the `sellAsset` function in the `EbtcBSM` contract to check the rate limiting constraint after transferring the assets to the contract. This will prevent users from bypassing the constraint through reentrancy.

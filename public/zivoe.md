# Zivoe-Findings

Zivoe is a real-world asset credit protocol aiming to disrupt predatory high-interest consumer lending. Leveraging a B2B2C model, Zivoe offers on-chain loans to regulated consumer lending entities, who then use that capital to fund consumer credit products off-chain. These entities then utilize the yield from such products to fulfill their on-chain obligations to Zivoe.



# [H-01] Double counting of the vote is possible in `revokeVestingSchedule` breaking the protocol core invariant.

## Code Line

https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewardsVesting.sol#L453

## Summary

When the function `revokeVestingSchedule` is called, the vested amount until that point is subtracted from the delegated votes instead of the total vested amount. The unvested amount is left in the reward contract and can be used to create a delegation for another user with the same amount. However, this violates the protocol invariant making the `sum of all user's votes greater than the total votes`.

## Details

`zivoeRewardVesting:createVestingSchedule` is used to create a vesting schedule for a user. The function also delegates the voting rights equal to the amount vested to the user for whom the vesting is created.

The `zivoeRewardVesting:revokeVestingSchedule` can be used to end the vesting schedule for a user before the vesting period ends. 

https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewardsVesting.sol#L429C5-L468C1

```solidity
function revokeVestingSchedule(address account) external updateReward(account) onlyZVLOrITO nonReentrant {
        
 439->  uint256 amount = amountWithdrawable(account);
        uint256 vestingAmount = vestingScheduleOf[account].totalVesting;

        vestingTokenAllocated -= amount;

        vestingScheduleOf[account].totalWithdrawn += amount;
        vestingScheduleOf[account].totalVesting = vestingScheduleOf[account].totalWithdrawn;
        vestingScheduleOf[account].cliff = block.timestamp - 1;
        vestingScheduleOf[account].end = block.timestamp;

        vestingTokenAllocated -= (vestingAmount - vestingScheduleOf[account].totalWithdrawn);

        _totalSupply = _totalSupply.sub(vestingAmount);
        _writeCheckpoint(_totalSupplyCheckpoints, _subtract, vestingAmount);
  453-> _writeCheckpoint(_checkpoints[account], _subtract, amount);
        _balances[account] = 0;
        stakingToken.safeTransfer(account, amount);

        vestingScheduleOf[account].revokable = false;

        emit VestingScheduleRevoked(
            account, 
            vestingAmount - vestingScheduleOf[account].totalWithdrawn, 
            vestingScheduleOf[account].cliff, 
            vestingScheduleOf[account].end, 
            vestingScheduleOf[account].totalVesting, 
            false
        );
    }

```

In `L439`, the amount that can be withdrawn is calculated. This amount will be less than the total vested amount if it is revoked before the vesting period ends. Only the withdrawable amount is deducted from the user's voting power in `L453`, instead of deducting all the vested amount. The unvested amount still represents the user's voting power.

The remaining unvested amount is kept in the contract and can be used to create a vesting schedule for another user. This results in a double voting delegation for the same amount because the first user still holds the voting power for the unvested amount.

### Flow

1. The vesting schedule for Alice is created for 100 ZVE for 1 year
2. The voting power of Alice is 100 
3. For some reason team decided to revoke the vesting schedule for Alice after 6 months
4. The `amountWithdrawable` at this point will be 50 ZVE 
5. 50 ZVE is transferred to Alice and  is subtracted from her voting power, making her voting power 100-50 = 50 ZVE
6. Now the reward contract has 100 - 50 = 50 ZVE remaining token balance
7. The team creates another vesting schedule for Bob using this remaining 50 ZVE which is otherwise unused. 
8. Bob's voting power is now 50. 
9. As you can see using the remaining 50 to create a vesting schedule results in the double counting of voting power. 

Here total votes are 100 but the sum of voting power the user holds is 150 which breaks the core invariant which says. `total votes should be always == to the sum of all user votes` 

## POC

```solidity
 function testDoubleCountingVotes() public {
        uint256 amount = uint256(20e18);
        uint256 totalBal = ZVE.balanceOf(address(vestZVE));
        // emitted events in createVestingSchedule() already tested above.
        assert(
            zvl.try_createVestingSchedule(
                address(vestZVE),
                address(moe),
                (amount % 360) + 1,
                ((amount % 360) * 5 + 1),
                amount,
                true
            )
        );

        // Pre-state.
        (
            uint256 start,
            uint256 cliff,
            uint256 end,
            uint256 totalVesting,
            uint256 totalWithdrawn,
            uint256 vestingPerSecond,

        ) = vestZVE.viewSchedule(address(moe));

        // warp some random amount of time from now to end.
        hevm.warp(block.timestamp + (amount % (end - start)));

        assert(zvl.try_revokeVestingSchedule(address(vestZVE), address(moe)));

        // Post-state.
        bool revokable;
        (
            ,
            cliff,
            end,
            totalVesting,
            totalWithdrawn,
            vestingPerSecond,
            revokable
        ) = vestZVE.viewSchedule(address(moe));

        assert(!revokable);
        // emitted events in createVestingSchedule() already tested above.
        amount = ZVE.balanceOf(address(vestZVE));
        assert(
            zvl.try_createVestingSchedule(
                address(vestZVE),
                address(tia),
                (amount % 360) + 1,
                ((amount % 360) * 5 + 1),
                amount,
                true
            )
        );

        vm.roll(block.number + 10);
        vm.warp(block.timestamp + 120);

        uint256 totalVotes = vestZVE.getPastTotalSupply(block.number - 1);
        uint256 moeVotes = vestZVE.getPastVotes(address(moe), block.number - 1);
        uint256 tiaVotes = vestZVE.getPastVotes(address(tia), block.number - 1);
        if (totalVotes < moeVotes + tiaVotes) {
            console2.log("votes diff: %e", (moeVotes + tiaVotes) - totalVotes);
        }
        console2.log("totalVotes: %e", totalVotes);
        console2.log("Sum of all votes: %e", moeVotes + tiaVotes);
    }
```

 

If we observe the test results then we can see that the total votes < sum of all user votes. when there are two user moe and tia.

```bash
Ran 1 test for src/TESTS_Core/Test_ZivoeRewardsVesting.sol:Test_ZivoeRewardsVesting
[PASS] testDoubleCountingVotes() (gas: 701509)
Logs:
  votes diff: 1.5374995375e19
  totalVotes: 1.4999995374995375e25
  Sum of all votes: 1.500001074999075e25

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 52.65s (372.07ms CPU time)

Ran 1 test suite in 55.00s (52.65s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)

```

## Impact

Double counting of the votes can have numerous consequences like users can user their voting power for granted to destroy the protocol governance and make the decision in favor of them. In the long run, this will destroy the protocol.  Because of this, i feel high severity is justified. 

## Recommendation

Instead of only subtracting the amount that has been completely vested from the user’s voting power subtract all the vested amount.


```diff
function revokeVestingSchedule(
        address account
    ) external updateReward(account) onlyZVLOrITO nonReentrant {
        require(
            vestingScheduleSet[account],
            "ZivoeRewardsVesting::revokeVestingSchedule() !vestingScheduleSet[account]"
        );
        require(
            vestingScheduleOf[account].revokable,
            "ZivoeRewardsVesting::revokeVestingSchedule() !vestingScheduleOf[account].revokable"
        );

        uint256 amount = amountWithdrawable(account);
        uint256 vestingAmount = vestingScheduleOf[account].totalVesting;
+       uint256 votesToDecrease = vestingAmount -
            vestingScheduleOf[account].totalWithdrawn;

        vestingTokenAllocated -= amount;
        vestingScheduleOf[account].totalWithdrawn += amount;
        vestingScheduleOf[account].totalVesting = vestingScheduleOf[account]
            .totalWithdrawn;
        vestingScheduleOf[account].cliff = block.timestamp - 1;
        vestingScheduleOf[account].end = block.timestamp;

        vestingTokenAllocated -= (vestingAmount -
            vestingScheduleOf[account].totalWithdrawn);

        _totalSupply = _totalSupply.sub(vestingAmount);
        _writeCheckpoint(_totalSupplyCheckpoints, _subtract, vestingAmount);
-       _writeCheckpoint(_checkpoints[account], _subtract, amount);
+       _writeCheckpoint(_checkpoints[account], _subtract, votesToDecrease);
        _balances[account] = 0;
        stakingToken.safeTransfer(account, amount);

        vestingScheduleOf[account].revokable = false;

        emit VestingScheduleRevoked(
            account,
            vestingAmount - vestingScheduleOf[account].totalWithdrawn,
            vestingScheduleOf[account].cliff,
            vestingScheduleOf[account].end,
            vestingScheduleOf[account].totalVesting,
            false
        );
    }
```

# [H-02] Incorrect Accounting of `_totalSupply` and `_totalSupplyCheckpoints` in `zivoeStakingRewards:revokeVestingSchedule` can cause DOS because of the underflow.

## Code Line

https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewardsVesting.sol#L452

## Summary

If a vested user calls `withdraw` before the `revokeVestingSchedule` It will break the accounting of the `_totalSupply` and  `_totalSupplyCheckpoints` resulting in the DOS of the system.

## Details

When a vested user calls the `withdraw` function to claim his vested amount. The `_totalSupply` and `_totalSupplyCheckpoints` is decreased by the amount the user is transferred. 

https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewardsVesting.sol#L509

```solidity
 /// @notice Withdraws the available amount of stakingToken from this contract.
    function withdraw() public nonReentrant updateReward(_msgSender()) {
        uint256 amount = amountWithdrawable(_msgSender());
        require(amount > 0, "ZivoeRewardsVesting::withdraw() amountWithdrawable(_msgSender()) == 0");
        
        vestingScheduleOf[_msgSender()].totalWithdrawn += amount;
        vestingTokenAllocated -= amount;

->>>    _totalSupply = _totalSupply.sub(amount);
->>>    _writeCheckpoint(_totalSupplyCheckpoints, _subtract, amount);
        _writeCheckpoint(_checkpoints[_msgSender()], _subtract, amount);
        _balances[_msgSender()] = _balances[_msgSender()].sub(amount);
        stakingToken.safeTransfer(_msgSender(), amount);

        emit Withdrawn(_msgSender(), amount);
    }
```

When the `revokeVestingSchedule` function is called for a user by the multisig/DAO, the  `_totalSupply`, and `_totalSupplyCheckpoints` will attempt to decrease by the user’s total vested amount, regardless of what has already been withdrawn. This may lead to an arithmetic underflow when a large amount is subtracted from a smaller amount. 

It is also possible for this issue to remain dormant for a long without reverting if `_totalSupply` is greater than the user vesting amount but it will cause a loss to the system in the long run because of bad accounting.

https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewardsVesting.sol#L452

```solidity
function revokeVestingSchedule(address account) external updateReward(account) onlyZVLOrITO nonReentrant {
      ...
        
        uint256 amount = amountWithdrawable(account);
        uint256 vestingAmount = vestingScheduleOf[account].totalVesting;

        vestingTokenAllocated -= amount;

        vestingScheduleOf[account].totalWithdrawn += amount;
        vestingScheduleOf[account].totalVesting = vestingScheduleOf[account].totalWithdrawn;
        vestingScheduleOf[account].cliff = block.timestamp - 1;
        vestingScheduleOf[account].end = block.timestamp;

        vestingTokenAllocated -= (vestingAmount - vestingScheduleOf[account].totalWithdrawn);

->>>    _totalSupply = _totalSupply.sub(vestingAmount);
->>>    _writeCheckpoint(_totalSupplyCheckpoints, _subtract, vestingAmount);
        _writeCheckpoint(_checkpoints[account], _subtract, amount);
       ...
    }
```

### POC

```solidity
function testRevokeVestingUnderflow() public {
        uint256 amount = uint256(20e18);
        uint totalBal = ZVE.balanceOf(address(vestZVE));
        // emitted events in createVestingSchedule() already tested above.
        assert(
            zvl.try_createVestingSchedule(
                address(vestZVE),
                address(moe),
                0,
                360 days,
                amount,
                true
            )
        );

        hevm.warp(block.timestamp + 100 days);

        vm.startPrank(address(moe));
        vestZVE.withdraw();
        vm.stopPrank();

        hevm.warp(block.timestamp + 2 days);

        zvl.try_revokeVestingSchedule(address(vestZVE), address(moe));
    }
```

Paste the test in `Test_ZivoeRewardsVesting.sol` and run, This test will revert  as you can see the moe has withdrawn some amount before the revokeVestingSchedule is called

## Impact

The `revokeVestingSchedule` can revert making this functionality useless and it can lead to an unforeseen loss to the protocol.  

 

## Recommendation

Instead of decreasing totalVesting amount only decrease the amount that has not been withdrawn

```diff
function revokeVestingSchedule(
        address account
    ) external updateReward(account) onlyZVLOrITO nonReentrant {
        require(
            vestingScheduleSet[account],
            "ZivoeRewardsVesting::revokeVestingSchedule() !vestingScheduleSet[account]"
        );
        require(
            vestingScheduleOf[account].revokable,
            "ZivoeRewardsVesting::revokeVestingSchedule() !vestingScheduleOf[account].revokable"
        );

        uint256 amount = amountWithdrawable(account);
        uint256 vestingAmount = vestingScheduleOf[account].totalVesting;
+        uint256 totalVotesToDecrease = vestingAmount-vestingScheduleOf[account].totalWithdrawn;

        vestingTokenAllocated -= amount;

        vestingScheduleOf[account].totalWithdrawn += amount;
        vestingScheduleOf[account].totalVesting = vestingScheduleOf[account]
            .totalWithdrawn;
        vestingScheduleOf[account].cliff = block.timestamp - 1;
        vestingScheduleOf[account].end = block.timestamp;

        vestingTokenAllocated -= (vestingAmount -
            vestingScheduleOf[account].totalWithdrawn);

-        _totalSupply = _totalSupply.sub(vestingAmount);
-        _writeCheckpoint(_totalSupplyCheckpoints, _subtract, vestingAmount);
-        _totalSupply = _totalSupply.sub(vestingAmount);
+        _writeCheckpoint(_totalSupplyCheckpoints, _subtract, totalVotesToDecrease);
        _writeCheckpoint(_checkpoints[account], _subtract, amount);
        _balances[account] = 0;
        stakingToken.safeTransfer(account, amount);

        vestingScheduleOf[account].revokable = false;

        emit VestingScheduleRevoked(
            account,
            vestingAmount - vestingScheduleOf[account].totalWithdrawn,
            vestingScheduleOf[account].cliff,
            vestingScheduleOf[account].end,
            vestingScheduleOf[account].totalVesting,
            false
        );
    }
```

# [M-03] Malicious actor can call `ZivoeRewards:depositReward` with 1 wei many times increase the reward’s `periodFinish` system DOS.

## Code Line

https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewards.sol#L228C3-L243C6

## Summary

Anyone can call `ZivoeRewards:depositReward` with 1 wei and increase the distribution time for rewards infinitely, which will cause a delay in rewards distribution for a prolonged period. 

## Details

The `ZivoeRewards:depositReward` is a public function that can be called by anyone to deposit the reward.  

https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeRewards.sol#L228C3-L243C6

```solidity
 function depositReward(address _rewardsToken, uint256 reward) external updateReward(address(0)) nonReentrant {
        IERC20(_rewardsToken).safeTransferFrom(_msgSender(), address(this), reward);

        // Update vesting accounting for reward (if existing rewards being distributed, increase proportionally).
        if (block.timestamp >= rewardData[_rewardsToken].periodFinish) {
            rewardData[_rewardsToken].rewardRate = reward.div(rewardData[_rewardsToken].rewardsDuration);
        } else {
            uint256 remaining = rewardData[_rewardsToken].periodFinish.sub(block.timestamp);
            uint256 leftover = remaining.mul(rewardData[_rewardsToken].rewardRate);
            rewardData[_rewardsToken].rewardRate = reward.add(leftover).div(rewardData[_rewardsToken].rewardsDuration);
        }

        rewardData[_rewardsToken].lastUpdateTime = block.timestamp;
241-    rewardData[_rewardsToken].periodFinish = block.timestamp.add(rewardData[_rewardsToken].rewardsDuration);
        emit RewardDeposited(_rewardsToken, reward, _msgSender());
    }
```

If a malicious actor deposits a very small amount of reward (1 wei) by calling a particular function in `L241`, the contract's periodFinish variable is incremented by a duration of 30 days. This periodFinish variable indicates the duration for which the deposited rewards will be distributed among all the stakers in the contract. So, if 100 rewards are deposited, then they should be distributed over the next 30 days to all the stakers.

However, if someone keeps calling this function with 1 wei, then the periodFinish variable keeps getting incremented by 30 days each time, effectively extending the reward distribution period. This can be done multiple times, making the rewards unclaimable for a very long time.

## Impact

If the function is called 100 times the periodFinish will be 100 * 30 = 3000 days. This means the reward distribution will take place over 3000 days which is practically not feasible and will cause the shutdown of the protocol reward system. I feel because of these reasons the High severity of this issue is justified. 

## Recommendation

Introduce access control for this function in `ZivoeRewards.sol` and `ZivoeRewardsVesting.sol`  

# [M-03] Voting delay in GovernorV2 is extremely short.

## Code Line

https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/ZivoeGovernorV2.sol#L63

## Summary

Voting delay in the `governorV2` is set to 1 block which is < 15 sec in ETH Mainnet. 15 seconds is very little time for the token holders to delegate their votes and prepare for voting.

## Details

In  `GovernorV2` initial voting configuration is done. 

```solidity
  constructor(IVotes _token, ZivoeTLC _timelock, address _GBL)
        Governor("ZivoeGovernorV2") GovernorSettings(1, 3600, 100000 ether)
        GovernorVotes(_token) GovernorVotesQuorumFraction(10) ZivoeGTC(_timelock) { GBL = _GBL; }

```

Here in initialization of `GovernorSetting` the first parameter is `votingDelay` which is set to 1 block(<15sec).  The voting delay is used as a time when the voting token holders can delegate token votes and prepare for the upcoming voting. 

It is practically impossible for enough users to delegate and prepare within a voting delay of less than 15 seconds from the proposal time. 

## Recommendation

Increase the voting delay to at least 1 day. I also feel the voting period of 3600 blocks (< 12 hours) is short and it should be at least 1 week





# [M-01] `OCL_ZVE:pushToLockerMulti` will likely revert because of allowance check.

## Code Line

https://github.com/sherlock-audit/2024-03-zivoe/blob/d4111645b19a1ad3ccc899bea073b6f19be04ccd/zivoe-core-foundry/src/lockers/OCL/OCL_ZVE.sol#L208

## Summary

The function `OCL_ZVE:pushToLockerMulti` checks the allowance for the router after `add_liquidity` in Uniswap/Sushiswap to be 0. However, it is highly unlikely that the router will use 100% of the approved tokens, resulting in the allowance check being reverted.

## Details

`OCL_ZVE:pushToLockerMulti` is used to add liquidity to uniswap or sushiswap. The token balances of the contract are used as an input amount. 

```solidity
  function pushToLockerMulti(
        address[] calldata assets,
        uint256[] calldata amounts,
        bytes[] calldata data
    ) external override onlyOwner nonReentrant {
       ....
        // Router addLiquidity() endpoint.
        uint balPairAsset = IERC20(pairAsset).balanceOf(address(this));
        uint balZVE = IERC20(ZVE).balanceOf(address(this));
        IERC20(pairAsset).safeIncreaseAllowance(router, balPairAsset);
        IERC20(ZVE).safeIncreaseAllowance(router, balZVE);

        // Prevent volatility of greater than 10% in pool relative to amounts present.
        (
            uint256 depositedPairAsset,
            uint256 depositedZVE,
            uint256 minted
        ) = IRouter_OCL_ZVE(router).addLiquidity(
                pairAsset,
                ZVE,
                balPairAsset,
                balZVE,
                (balPairAsset * 9) / 10,
                (balZVE * 9) / 10,
                address(this),
                block.timestamp + 14 days
            );
        emit LiquidityTokensMinted(minted, depositedZVE, depositedPairAsset);
        assert(IERC20(pairAsset).allowance(address(this), router) == 0);
        assert(IERC20(ZVE).allowance(address(this), router) == 0);
        
        ....
        }
```

If we closely look at this function then this is approving all the balances to the router. deadline is set to 14 days and `amountAMin` and `amountBMin` are allowed to be 90% of the total approved amount. meaning it is possible for a router to only use 90% of the approved amount. 

The assertion check, at last, required the router allowance to be 0 after the addLiquidity happens. the function assumes that the liquidity pool is always balanced and will utilize 100% of the approved amount. Which is incorrect.

The optimal  amount calculation in uniswap v2 confirms that 100% of approved amount is not always used 

```solidity
 function _addLiquidity(
        address tokenA,
        address tokenB,
        uint amountADesired,
        uint amountBDesired,
        uint amountAMin,
        uint amountBMin
    ) internal virtual returns (uint amountA, uint amountB) {
        ....
        } else {
            uint amountBOptimal = UniswapV2Library.quote(amountADesired, reserveA, reserveB);
            if (amountBOptimal <= amountBDesired) {
                require(amountBOptimal >= amountBMin, 'UniswapV2Router: INSUFFICIENT_B_AMOUNT');
                (amountA, amountB) = (amountADesired, amountBOptimal);
            } else {
                uint amountAOptimal = UniswapV2Library.quote(amountBDesired, reserveB, reserveA);
                assert(amountAOptimal <= amountADesired);
                require(amountAOptimal >= amountAMin, 'UniswapV2Router: INSUFFICIENT_A_AMOUNT');
                (amountA, amountB) = (amountAOptimal, amountBDesired);
            }
        }
    } 
```

If 100% of the approved amount is not used then the allowance check will fail reverting the function.

It may be pointed out that the team can supply only the required amount of collateral after calculating it offchain or using the `quote` function from uniswap. Even if the team does so the other points of failure are the following:

1. Anyone can transfer 1 wei to the contract frontrunning the team deposit.
2. The 14-day deadline with 90% utilization instead of 100% is set. so the add liquidity can be executed at any time in the future when the pool is not balanced as desired. 

### POC

```solidity
 function test_OCL_ZVE_UNIV2_pushToLockerMulti_state_initial(
        uint96 randomA,
        uint96 randomB
    ) public {
        uint256 amountA = (uint256(randomA) % (10_000_000 * USD)) + 10 * USD;
        uint256 amountB = (uint256(randomB) % (10_000_000 * USD)) + 10 * USD;
        uint256 modularity = 0; // EDIT

        address[] memory assets = new address[](2);
        uint256[] memory amounts = new uint256[](2);

        assets[1] = address(ZVE);
        amounts[0] = amountA;
        amounts[1] = amountB;

        if (modularity == 0) {
            assets[0] = DAI;

            // Pre-state.
            assertEq(OCL_ZVE_UNIV2_DAI.basis(), 0);
            assertEq(OCL_ZVE_UNIV2_DAI.nextYieldDistribution(), 0);

            assert(
                god.try_pushMulti(
                    address(DAO),
                    address(OCL_ZVE_UNIV2_DAI),
                    assets,
                    amounts,
                    new bytes[](2)
                )
            );

            // Post-state.
            (uint256 basis, uint256 lpTokens) = OCL_ZVE_UNIV2_DAI.fetchBasis();
            assertGt(basis, 0);
            assertGt(lpTokens, 0);
            assertEq(IERC20(DAI).balanceOf(address(OCL_ZVE_UNIV2_DAI)), 0);
            assertEq(
                IERC20(address(ZVE)).balanceOf(address(OCL_ZVE_UNIV2_DAI)),
                0
            );
            assertEq(OCL_ZVE_UNIV2_DAI.basis(), basis);
            assertEq(
                OCL_ZVE_UNIV2_DAI.nextYieldDistribution(),
                block.timestamp + 30 days
            );

            // try adding liquidity again
            //@audit Frontrun and deposit
            deal(DAI, address(OCL_ZVE_UNIV2_DAI), 1);

            assert(
                god.try_pushMulti(
                    address(DAO),
                    address(OCL_ZVE_UNIV2_DAI),
                    assets,
                    amounts,
                    new bytes[](2)
                )
            );
        }
        ....
        }
```

 add this code to the test with the same name and the test will revert. 

## Impact

The `OCL_ZVE:pushToLockerMulti` has a high chance of reverting with the following allowance assumptions. 

## Recommendation

We required the allowance to be 0 after the function for tokens like USDT.  So check if the allowance is 0 first, if not, then set it to 0.

```solidity
if(IERC20(pairAsset).allowance(address(this), router) != 0)) {
 IERC20(pairAsset).approve(router, 0)
}
```
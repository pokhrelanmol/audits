# Beedle - Oracle free perpetual lending - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. ERC20 tokens that have a fee-on-transfer mechanism require special handling](#H-01)
    - ### [H-02. New lenders can cause disruption to the system by modifying the original tokens when purchasing the loan.](#H-02)
    - ### [H-03. Reward Token(WETH) deposit transaction can be front-run with a deposit of a large quantity of staking token(TKN) making the frontrunner eligible for a large share of claimable reward.](#H-03)
    - ### [H-04.  Does not update rewards state correctly on ```Staking.deposit()``` resulting in significant loss of rewards of the existing stakers. ](#H-04)
    - ### [H-05. No slippage Parameter where the amountOutMinimum is set to zero swaps are prone to sandwich attacks](#H-05)
- ## Medium Risk Findings
    - ### [M-01. The lender can front-run the borrowing transaction and set the interest rate to maximum](#M-01)

- ## Gas Optimizations / Informationals
    - ### [G-01. Typo in the ```LoanSeized()``` event ](#G-01)
    - ### [G-02. Floating pragma is set](#G-02)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: BeedleFi

### Dates: Jul 24th, 2023 - Aug 7th, 2023

[See more contest details here](https://www.codehawks.com/contests/clkbo1fa20009jr08nyyf9wbx)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 5
   - Medium: 1
   - Low: 0
  - Gas/Info: 2

# High Risk Findings

## <a id='H-01'></a>H-01. ERC20 tokens that have a fee-on-transfer mechanism require special handling            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L246

## Summary
Some tokens take a transfer fee (STA, PAXG) and there are some that currently do not but might do so in the future (USDT, USDC). This can create an accounting problem in the contract. 


## Vulnerability Details
As per the information provided by the developer, the contract can handle any ERC20 tokens but there may arise some issues with tokens with fees on transfer like STA, PAXG, etc. For example, when taking the loan the borrower will specify the debt amount, when the debt amount is actually transferred to the borrower it will be less than expected but all the calculations like `loanRation` and `Loan` struct update will happen based on the debt amount provided by the user.

Let's consider the scenario where the transfer fee on tokens is 10%. The borrower will specify `debt` to 100 tokens but he will only receive 90 tokens all the calculations inside the contract will happen on the actual 100 tokens resulting in the incorrect calculation which will eventually result in the borrower's loss in this case.

Note that this is only one example of an issue related to tokens with transfer fees but there may the multiple instances of issues alike so it is important to handle all similar edge cases related to this type of token.

## Impact
The allowance of the use of tokens with fees can create a lot of accounting problems in contracts which may cause the loss of user funds or system DOS.
## Tools Used
manual review
## Recommendations
If possible do not allow any weird tokens to be traded in the protocol.

Another option would be to add an extra calculation to check and handle all use cases for this type of fees bearing token. For example, in the case of the borrowing scenario, we can calculate the balance before the transfer and the balance after the transfer for the borrower account and only perform calculations on the difference.

```js

uint256 balBefore = IERC20(loanToken).balanceof(msg.sender);
//perform transfer
uint256 balAfter = IERC20(loanToken).balanceof(msg.sender);

uint256 actualDebt = balAfter - balBefore;
```
## <a id='H-02'></a>H-02. New lenders can cause disruption to the system by modifying the original tokens when purchasing the loan.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L314

## Summary
During the ```Lender.buyLoan()``` function, the buyer has the ability to purchase the loan in a pool that has been created with fake tokens (other than loan and collateral tokens). This vulnerability could potentially allow malicious actors to manipulate the system and cause harm to the lending platform, resulting in a DOS attack.
## Vulnerability Details
During a Dutch auction, a new lender can purchase a loan from an old lender by using the ```Lender.buyLoan()``` function. This adds the loan to their own pool with a new interest rate that is less than the currentAuctionRate. However, the new pool created may contain tokens other than the ```loan.loanToken``` and ```loan.collateralToken```. This presents a risk, as a new lender may buy the loan with fake tokens that are not actually deposited into the poolId, causing an imbalance in the tokens. This can lead to issues later on when the borrower wants to repay the loan. Inside the ```repay()``` function, the pool is calculated using the new lender address, old loan, and collateral token address. If there is no such combination of poolId, the system will face a DOS attack resulting in the loss of borrower collateral as he will never be able to get back his collateral using ```repay()```.

There is no direct benefit for attacked doing this attack so likelihood of this happening is less therefore this is marked as a medium issue.

## POC
```js
function test_unfairProfitLender() public {
        vm.startPrank(lender1);
        //    Lender set pool
        Pool memory p = Pool({
            lender: lender1,
            loanToken: address(loanToken),
            collateralToken: address(collateralToken),
            minLoanSize: 100 * 10 ** 18,
            poolBalance: 1000 * 10 ** 18,
            maxLoanRatio: 2 * 10 ** 18,
            auctionLength: 1 days,
            interestRate: 1000,
            outstandingLoans: 0
        });
        bytes32 poolId = lender.setPool(p);

        vm.stopPrank();
        // Borrower borrows
        vm.startPrank(borrower);
        Borrow memory b = Borrow({poolId: poolId, debt: 100 * 10 ** 18, collateral: 100 * 10 ** 18});

        Borrow[] memory borrows = new Borrow[](1);
        borrows[0] = b;
        lender.borrow(borrows);

        vm.stopPrank();

        vm.startPrank(lender1);
        loanToken.transfer(address(borrower), 100 * 10 ** 18);
        // start the auction
        uint256[] memory loanIds = new uint256[](1);
        loanIds[0] = 0;
        lender.startAuction(loanIds);
        vm.stopPrank();

        // attacker create new fake pool and buy the loan
        vm.startPrank(lender2);
        loanToken2.approve(address(lender), 10000 * 10 ** 18);

        Pool memory p2 = Pool({
            lender: lender2,
            loanToken: address(loanToken2),
            collateralToken: address(collateralToken2),
            minLoanSize: 100 * 10 ** 18,
            poolBalance: 1000 * 10 ** 18,
            maxLoanRatio: 2 * 10 ** 18,
            auctionLength: 1 days,
            interestRate: 1000,
            outstandingLoans: 0
        });
        vm.startPrank(lender2);
        vm.warp(block.timestamp + 10 hours);
        lender.zapBuyLoan(p2, 0);
        vm.stopPrank();

        // repay the loan
        vm.startPrank(borrower);
        loanToken.approve(address(lender), 102 * 10 ** 18);
        
        lender.repay(loanIds); //DOS

        vm.stopPrank();
    }
```
## Impact

If a new lender creates a pool with fake tokens and purchases an original loan using those tokens, it can cause a Denial of Service (DOS) attack, which can result in the borrower losing all of their collateral.

## Tools Used
manual review
## Recommendations
To mitigate this issue we can add two missing checks in the buyLoan function

```js
    // validate the new pool
            if (pool.loanToken != loan.loanToken) revert TokenMismatch();
            if (pool.collateralToken != loan.collateralToken) {
                revert TokenMismatch();
            }
```
## <a id='H-03'></a>H-03. Reward Token(WETH) deposit transaction can be front-run with a deposit of a large quantity of staking token(TKN) making the frontrunner eligible for a large share of claimable reward.            

### Relevant GitHub Links
	
https://github.com/0xTingle/Beedle/blob/main/src/Staking.sol

## Summary

Reward Token(WETH) deposit transaction can be front-run with a deposit of a large quantity of staking token(TKN) making the frontrunner eligible for a large share of claimable reward.

## Vulnerability Details

The staking contract will be refilled with the reward token(WETH) on a regular basis to maintain the reward cycle. The attacker can front-run this reward deposit transaction and deposit the large amount of TKN using the ```Staking.deposit()```, This will make him eligible to claim a high reward from the contract in an unfair way.

## Impact

The frontrunning bot can see the upcoming reward deposit in the mempool and can front-run the transaction by depositing a large amount of staking tokens, that way the attacker can earn a large portion of the reward making the reward system unfair.

## Tools Used

manual review

## Recommendations

The better way to write the staking contract would be to use Synthetix StakingRewards as an inspiration which does not have the frontrunning issues.

[example](https://solidity-by-example.org/defi/staking-rewards/)
[Inspiration](https://github.com/curvefi/unipool-fork/blob/master/contracts/StakingRewards.sol)

## <a id='H-04'></a>H-04.  Does not update rewards state correctly on ```Staking.deposit()``` resulting in significant loss of rewards of the existing stakers.             

### Relevant GitHub Links
	
https://github.com/0xTingle/Beedle/blob/884be50ea974b264b6c888932ca9d3b039c400e0/src/Staking.sol#L39

## Summary

```Staking.deposit()``` There's a significant issue in how the contract updates its record of cumulative rewards, which results in users not receiving the correct amount of rewards and the remaining portion of funds stuck in the contract forever.

## Vulnerability Details
```Staking.deposit()``` There's a significant issue in how the contract updates its record of cumulative rewards, which results in users not receiving the correct amount of rewards and the remaining portion of funds stuck in the contract forever.

Here's an example to illustrate the issue: Suppose the staking contract has 100 TKN deposited, and the protocol transfers 100 WETH as rewards for those staking their tokens. Until the next action that changes the state of the staking contract, the cumulative rewards and other necessary details aren't updated.

Now, let's say another user deposits 100 TKN into the contract. The **`deposit`** function takes 100 TKN from the user and updates the contract's state. However, instead of calculating the additional cumulative rewards as 100 WETH divided by the original 100 TKN (which would be correct), it calculates the additional cumulative rewards as 100 WETH divided by the new total of 200 TKN. This is because the contract fetches the current deposit amount before it updates the rewards information. As a result, the rewards are distributed over a larger number of tokens than they should be, leading to inaccurately low reward payouts.

## Impact

users will not receive the correct amount of rewards and funds will get stuck in the contract forever.

## POC

```js

// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console2} from "forge-std/Test.sol";
import {ERC20, IERC20} from "openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Staking} from "../src/Staking.sol";

contract Token is ERC20 {
    constructor(address minter) ERC20("TKN", "TOKEN") {
        _mint(minter, 1e23);
    }
}

contract StakingTest is Test {
    Staking public staking;
    IERC20 public weth;
    ERC20 public tkn;
    address wethWhale = 0x8EB8a3b98659Cce290402893d0123abb75E3ab28;
    address user = address(123456);

    function setUp() public {
        vm.createSelectFork("https://mainnet.infura.io/v3/API_KEY");
        tkn = new Token(wethWhale);
        weth = IERC20(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);
        staking = new Staking(address(tkn), address(weth));
        vm.deal(user, 100e18);

        vm.startPrank(wethWhale);
        tkn.transfer(user, 1000e18);
        vm.stopPrank();
    }

    function testStake() public {
        uint256 amount = 1e20;
        vm.startPrank(wethWhale);
        tkn.approve(address(staking), amount);
        staking.deposit(amount);
        console2.log("Initial TKN Deposit by user 1: ", amount / 1e18);
        // deposit 100 weth as rewards
        weth.transfer(address(staking), amount);
        console2.log("Total reward WETH deposited: ", weth.balanceOf(address(staking)) / 1e18);
        vm.stopPrank();

        vm.startPrank(user);
        tkn.approve(address(staking), amount);
        staking.deposit(amount);
        console2.log("Later TKN deposit by user 2: ", amount / 1e18);
        // deposit 100 weth as rewards

        vm.stopPrank();

        uint256 wethBalBefore = weth.balanceOf(wethWhale);

        vm.startPrank(wethWhale);
        console2.log("Updated Index: ", staking.index());
        console2.log("Total Reward in the pool before claim: ", weth.balanceOf(address(staking)) / 1e18);
        staking.claim();

        uint256 wethBalAfter = weth.balanceOf(wethWhale);
        console2.log("Total TKN supply", tkn.balanceOf(address(staking)) / 1e18);
        console2.log("Reward claimed by user expected 100 but getting: ", (wethBalAfter - wethBalBefore) / 1e18);
        console2.log("Reward in the pool after claim: ", weth.balanceOf(address(staking)) / 1e18);
        assertEq(wethBalAfter - wethBalBefore, amount / 2);
        vm.stopPrank();

        wethBalBefore = weth.balanceOf(user);
        vm.startPrank(user);
        staking.claim();
        wethBalAfter = weth.balanceOf(user);
        console2.log("Reward Claimed by user 2: ", (wethBalAfter - wethBalBefore) / 1e18);
        vm.stopPrank();
    }
}
```
## Output
<a href="https://ibb.co/YXgkGWh"><img src="https://i.ibb.co/zFdRw5Z/Screenshot-from-2023-07-30-11-30-28.png" alt="Screenshot-from-2023-07-30-11-30-28" border="0"></a><br /><a target='_blank' href='https://imgbb.com/'>temporary image hosting</a><br />


## Tools Used

manual review

## Recommendations

Update the user’s rewards before transferring tokens from the staker. 

```js
function deposit(uint _amount) external {
          updateFor(msg.sender);
          TKN.transferFrom(msg.sender, address(this), _amount);
          balances[msg.sender] += _amount;
    }
```
## <a id='H-05'></a>H-05. No slippage Parameter where the amountOutMinimum is set to zero swaps are prone to sandwich attacks            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Fees.sol#L38

## Summary

The ```ISwapRouter.ExactInputSingleParams()``` in the Fees.sol contract sets the amountOutMinimum  to zero, which makes it prone to sandwich attacks.

## Vulnerability Details

The “0” here is the value of the amountOutMinimum argument which is used for slippage tolerance. 0 value here essentially means 100% slippage tolerance. This is a very easy target for MEV and bots to do a flash loan sandwich attack on each of the strategy’s swaps, resulting in a very big slippage on each trade.
[Read more from official docs](https://docs.uniswap.org/contracts/v3/guides/swaps/single-swaps#)

[Previously found on](https://github.com/sherlock-audit/2023-05-USSD-judging/issues/12)


## Impact

100% slippage tolerance can be exploited in a way that the fees receive will be much less value than they should have been. This can be done on every trade if the trade transaction goes through a public mempool.

## Tools Used

manual review

## Recommendations

A Good solution would be for the developers to set the slippage to a strict amount where the swaps would revert if such an attack is performed on it.
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. The lender can front-run the borrowing transaction and set the interest rate to maximum            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L221

## Summary

Lenders can potentially front-run a borrowing transaction in the mempool, taking advantage of the updateInterestRate() function to set the interest rate to the maximum. This would allow the lender to potentially profit from the transaction in an unfair manner.

## Vulnerability Details

When a borrower initiates a borrowing transaction, it is typically recorded in a mempool before it is confirmed and included in a block on the blockchain. During this time, the borrowing transaction is visible to other participants, including lenders and arbitrageurs. A lender who closely monitors the mempool can observe pending borrowing transactions and take advantage of the situation.

The critical component in this scenario is the **`updateInterestRate()`** function. This function is responsible for setting the interest rate for the borrower's loan.This is how the flow works

1. Monitor all borrowing transactions: The lender continuously monitors the mempool for all incoming borrowing transactions, regardless of the loan amount or collateral-to-loan ratio.
2. Execute the front-run: Whenever a borrowing transaction is detected, the lender quickly initiates their own transaction providing high gas, ensuring they call the **`updateInterestRate()`** function before the original transaction can do so.
3. Set the interest rate to the maximum: By calling the **`updateInterestRate()`** function first, the lender sets the interest rate to the maximum allowed value, irrespective of the borrower's willingness to take the loan at that high rate.
4.  Benefit from all transactions: By front-running and setting the interest rate to the maximum for every loan, the lender ensures that all borrowers are subjected to significantly higher interest rates than they should be paying based on fair market conditions and risk assessments.
5. Unfair profit from all loans: With the ability to manipulate the interest rates across all borrowing transactions, the lender makes unfair profits from each loan, as borrowers will have to pay inflated interest rates that they did not expect or agree to.

## POC
```js

function test_frontrunBorrowAndIncreaseInterest() public {
        //    Lender set pool
        Pool memory p = Pool({
            lender: lender1,
            loanToken: address(loanToken),
            collateralToken: address(collateralToken),
            minLoanSize: 100 * 10 ** 18,
            poolBalance: 1000 * 10 ** 18,
            maxLoanRatio: 20 * 10 ** 18,
            auctionLength: 1 days,
            interestRate: 1000,
            outstandingLoans: 0
        });
        vm.startPrank(lender1);
        bytes32 poolId = lender.setPool(p);
        vm.stopPrank();
        (,,,,,,, uint256 interestRate,) = lender.pools(poolId);
        console.log("Initial Interest Rate", interestRate / 100, "%");

        console.log("Frontrunning borrow transaction by increasing interest rate...");
        vm.startPrank(lender1);
        lender.updateInterestRate(poolId, 100000);
        vm.stopPrank();
        (,,,,,,, uint256 interestRate1,) = lender.pools(poolId);
        console.log("Interest Rate after frontrunning", interestRate1 / 100, "%");

        // Mint some loan token to borrower
        loanToken.mint(address(borrower), 200 * 10 ** 18);
        // Borrower borrows
        vm.startPrank(borrower);
        Borrow memory b = Borrow({poolId: poolId, debt: 100 * 10 ** 18, collateral: 100 * 10 ** 18});

        Borrow[] memory borrows = new Borrow[](1);
        borrows[0] = b;
        lender.borrow(borrows);

        vm.warp(block.timestamp + 2 days); //increase time since borrow to accumulate interest

        // Borrower repays the loan
        //*5.479452055×10e18 => This what we need to repay with 1000% interest rate after 2 days
        //*5.479452055×10e16 => This what we need to repay with 10% interest rate after 2 days
        loanToken.approve(address(lender), 105 * 10 ** 18);
        uint256[] memory loanIds = new uint256[](1);
        loanIds[0] = 0;
        vm.expectRevert(ERC20.InsufficientAllowance.selector);
        lender.repay(loanIds);
        vm.stopPrank();
    }
```
## Impact

The impact of a lender front-running borrowing transactions and manipulating interest rates in the mempool includes unfair profits for the lender, loss of trust, ecosystem instability, and reputational damage to the decentralized finance (DeFi) ecosystem.

## Tools Used

manual review

## Recommendations

To prevent lender front-running and unfair manipulation of interest rates, a crucial mitigation measure is to design the smart contract in a way that restricts lenders from changing the **`interestRate`** directly or arbitrarily.   

[Read more](https://consensys.github.io/smart-contract-best-practices/attacks/frontrunning/)





# Gas Optimizations / Informationals

## <a id='G/I-01'></a>G/I-01. Typo in the ```LoanSeized()``` event             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-07-beedle/blob/658e046bda8b010a5b82d2d85e824f3823602d27/src/Lender.sol#L50

## Summary

There is a spelling mistake in the event, it should be  ```LoanSeized()``` instead of ```LoanSiezed()```

## Impact

Can easily create confusion when listening to the events resulting in the unexpected behaviour.

## Tools Used

manual review

## Recommendations
change the spelling to ```LoanSeized``` in the place of event declaration and emission.
## <a id='G/I-02'></a>G/I-02. Floating pragma is set            

### Relevant GitHub Links
	
https://github.com/code-423n4/2021-11-streaming-findings/issues/19

## Summary

contracts use the floating pragma ^0.8.19.

## Vulnerability Details

Contracts should be deployed with the same compiler version and flags that they
have been tested with thoroughly. Locking the pragma helps to ensure
that contracts do not accidentally get deployed using another pragma,
for example, either an outdated pragma version that might introduce bugs
that affect the contract system negatively or a recently released pragma
version which has not been extensively tested.

## Impact

Contracts should be deployed using the same compiler version/flags with which they have been tested. Locking the pragma (for e.g. by not using ^ in pragma solidity 0.8.19) ensures that contracts do not accidentally get deployed using an new compiler version with unfixed bugs
## Tools Used

manual review

## Recommendations

Lock the pragma version to the same version as used in the other contracts and also consider known bugs (https://github.com/ethereum/solidity/releases) for the compiler version that is chosen.

Pragma statements can be allowed to float when a contract is intended for consumption by other developers, as in the case with contracts in a library or EthPM package. Otherwise, the developer would need to manually update the pragma in order to compile it locally.

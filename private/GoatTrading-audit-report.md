# Goat Trading Audit Report

## Introduction

An internal security review of the **Goat Trading protocol** alongside protocol development was done by **0xanmol**, with a focus on the security aspects of the application's smart contracts implementation.

## **Disclaimer**

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time, resource, and expertise-bound effort where I try to find as many vulnerabilities as possible. I can not guarantee 100% security after the review or even if the review will find any problems with your smart contracts. Subsequent security reviews, bug bounty programs, and on-chain monitoring are strongly recommended.

## About 0xanmol

Anmol Pokhrel, or **0xanmol**, is an independent smart contract security researcher. Having found numerous security vulnerabilities in various protocols, he does his best to contribute to the blockchain ecosystem and its protocols by putting time and effort into security research & reviews.

[Twitter](https://twitter.com/AnmolPokhrel6)

[Github](https://github.com/pokhrelanmol)

## About Goat Trading

Goat Trading is a phased Automated Market Maker (PAMM) protocol that enables anyone to launch their token pool to accumulate a specific amount of WETH. Initially, this pool is known as a presale AMM and includes a virtual amount in the reserve. As swaps gradually increase the desired WETH amount, the presale pool changes into an x*y AMM. The initial Liquidity provider then owns all the raised WETH. Besides token launch, Goat Trading also features built-in MEV protection and allows LPs to withdraw rewards in WETH without removing liquidity.

## **Severity classification**

| Severity | Impact: High | Impact: Medium | Impact: Low |
| --- | --- | --- | --- |
| Likelihood: High | Critical | High | Medium |
| Likelihood: Medium | High | Medium | Low |
| Likelihood: Low | Medium | Low | Low |

**Impact** - the technical, economic, and reputation damage of a successful attack

**Likelihood** - the chance that a particular vulnerability gets discovered and exploited

**Severity** - the overall criticality of the risk

## Security Assessment Summary

repo: [https://github.com/inedibleX/](https://github.com/inedibleX/)goat-trading

### Scope

- exchange
    - GoatV1pair.sol
    - GoatV1ERC20.sol
    - Goatv1Factory.sol
- periphery
    - GoatV1Router.sol
- library
    - GoatTypes.sol
    - GoatLibrary.sol

## Findings Summary

| ID | Title | Severity | Status |
| --- | --- | --- | --- |
| [C-01] | Initial LP can bypass fractional liquidity checks by passing different to address when removing liquidity. | Critical | Fixed |
| [M-01]  |  Unnecessary state changes occur when calling _updateInitialLpInfo with isBurn set as true within _burnLiquidityAndConvertToAmm | medium | Fixed |
| [M-02] | Anyone holding the token can create a pool with a dust amount and convert that to amm, not giving the actual team a chance to raise WETH | Critical | Acknowledged |
| [M-03] |  An attacker could potentially reduce the fractional liquidity of the initial LP by minting a dust amount. | high | Acknowledged |
| [L-01] | incorrect data is passed when emitting events for mint and burn | medium | Acknowledged |
| [G-01] | Reading the state only when necessary can help save gas in the mint function. | low | Acknowledged |
| [G-02] | updateFeeReward can be skipped if to address is pair in _beforeTokenTransfer | low | Acknowledged |

## [C-01] Initial LP can bypass fractional liquidity checks by passing different `to` address when removing liquidity.

**Impact**: High, as the protocol will compromise its core functionality to prevent token dumping

**Likelihood**: High, as teams can directly dump their all tokens.

**Commit Hash**: 0xxxxxx

### Description

The `to` address given in a burn function is used to determine if that address is the initial liquidity provider (LP). If the `to` address matches the initial LP, then fractional liquidity logic is implemented to prevent the initial LP from dumping all the tokens and walking away with the raised WETH.

However, the initial LP can bypass this check simply by providing a different `to` address than the one recorded as the initial LP.

//code and commit hash

### Recommendation

The protocol should be able to identify if the caller of the burn function is the initial Liquidity Provider (LP). One approach could be to store the address that transfers the LP token to the pair contract before invoking the burn function. before transfer hook can be used to achieve this.

## [M-01] Unnecessary state changes occur when calling `_updateInitialLpInfo` with `isBurn` set as true within `_burnLiquidityAndConvertToAmm`

**Impact**: Medium. The initial liquidity provider can now withdraw their liquidity in 3 weeks instead of 2.

**Likelihood**: High. This will happen every time the pool converts to AMM.

**Commit Hash**: 0xxxxxx

### Description

The `_updateInitialLpInfo` function serves to update the details of the initial liquidity provider (LP). It modifies their fractional balance and remaining withdrawals. If the initial LP is burning their liquidity, the number of remaining withdrawals decreases by one, and the last withdrawal timestamp is updated to the current time. This change is determined by the `isBurn` parameter passed into the function.

This function performs as expected for standard liquidity withdrawals. However, during the conversion of the presale pool to AMM, if the initial LP's liquidity is burned, it unnecessarily alters the remaining withdrawals and the last withdrawal time.

Consequently, the initial LP only needs to wait 3 weeks to withdraw all their liquidity.

### Recommendation

In the `_updateIntialLpInfo` function, add a check to distinguish between internal burn and actual burn.

## [M-02] Anyone holding the token can create a pool with a dust amount and convert that to amm, not giving the actual team a chance to raise WETH.

**Impact**: High, since this could prevent the team from raising any WETH.

**Likelihood**: Medium, because the attacker would need to hold the team token and there's no direct financial incentive for them.

**Commit Hash**: 0xxxxxx

### Description

An attacker with the token could front-run the actual pool creation by creating a pool with a dust amount and converting it directly to AMM. This could be easily accomplished by providing a very small `bootstrapEth` and an equal `initialEth` amount.

Once the pool is created and converted to AMM, the actual team cannot use the `takeOver` function to gain control of the pool.

### Recommendation

Prevent `bootstrapEth` from being a dust amount and establish a minimum requirement for bootstrapEth.

For instance, if the protocol only permits pool creation with 1 WETH, an attacker would need to put 1 WETH to convert it directly to AMM, which wouldn't be beneficial for them.

## [M-03] An attacker could potentially reduce the fractional liquidity of the initial LP by minting a dust amount.

**Impact**: High. The system might not function as expected for the initial LP.

**Likelihood**: Low, as there's no direct benefit for the attacker.

**Commit Hash**: 0xxxxxxxx

### Description

If a malicious user sets a `to` address as the initial Liquidity Provider (LP) in the `mint` function, it alters the fractional liquidity and remaining withdrawal for the initial LP, potentially decreasing the liquidity that the initial LP could withdraw.

For example, suppose the initial LP has a liquidity balance of 75 and a remaining withdrawal of 3, indicating a fractional liquidity of 25.

If a malicious actor mints a liquidity of 1 to this LP, the remaining withdrawal resets to 4, and the fractional liquidity changes to 76 / 4 = 19.

If an attacker repeats this action, the initial LP may never be able to withdraw all of their amounts, even after the vesting duration ends.

```solidity
 if (mintVars.isFirstMint || to == _initialLPInfo.liquidityProvider) {
      _updateInitialLpInfo(liquidity, balanceEth, to, false, false);
    }
```

```solidity
function _updateInitialLpInfo(
    uint256 liquidity,
    uint256 wethAmt,
    address lp,
    bool isBurn,
    bool internalBurn
  ) internal {
    GoatTypes.InitialLPInfo memory info = _initialLPInfo;

    if (internalBurn) {
      // update from from swap when pool converts to an amm
      info.fractionalBalance = uint112(liquidity) / 4;
    } else if (isBurn) {
      if (lp == info.liquidityProvider) {
        info.lastWithdraw = uint32(block.timestamp);
        info.withdrawalLeft -= 1;
      }
    } else {
    //@audit decrease fractional balance of initial LP
      info.fractionalBalance = uint112(((info.fractionalBalance * info.withdrawalLeft) + liquidity) / 4);
      info.withdrawalLeft = 4;
      info.liquidityProvider = lp;
      if (wethAmt != 0) {
        info.initialWethAdded = uint104(wethAmt);
      }
    }

    // Update initial liquidity provider info
    _initialLPInfo = info;
  }
```

### Recommendation

Prohibit the minting of liquidity to the initial LP.

## [L-01] Incorrect data is passed when emitting events for `mint` and `burn`

### Description

In the event data, `msg.sender` is passed as a user address inside the `mint` and `burn` functions. These functions are expected to be called by the router, so in this scenario, `msg.sender` will be a router.

### Recommendation

Use the `to` address as a user in the event, instead of `msg.sender`.

## [G-01] Saving gas in the mint function by reading the state only when necessary

### Description

Reading the state can incur 2100 gas if it's cold storage and 100 gas if it's hot storage.

Within the `mint` function, certain states are required only when the pool is in a presale.

For example:

```solidity
   //@audit this state read can be done inside the if block
    mintVars.virtualEth = _virtualEth;
    mintVars.initialTokenMatch = _initialTokenMatch;
    mintVars.bootstrapEth = _bootstrapEth;

```

Rather than always initializing these local variables by reading storage, it would be more gas-efficient to initialize them only when the pool is in a presale.

## [G-02] `updateFeeRewards` can be skipped if `to` address is `pair` in `_beforeTokenTransfer`

### Description

Inside `_beforeTokenTransfer` the rewards are updated for the sender and receiver of the token.

If the receiver is pair address itself then, there is no need to update the receiver’s rewards. This can save a significant amount of gas for a sender.

```solidity
  // Update fee rewards for both sender and receiver
    _updateFeeRewards(from);
    // @audit do not update fee if to is address(this)
    _updateFeeRewards(to);
```

### Recommendation

Add a condition to check for pair address

```solidity
 // Update fee rewards for both sender and receiver
    _updateFeeRewards(from);
    if(to != address(this){
    _updateFeeRewards(to);
    }
```

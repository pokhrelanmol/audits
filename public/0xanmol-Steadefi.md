# Steadefi - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. `lpTokenValue` will always calculate for withdrawals even if the action is deposits because `isDeposit` is always passed false](#M-01)
    - ### [M-02. The protocol will mint unnecessary fees if the vault is paused and reopened later.](#M-02)
    - ### [M-03. `Compound()` will not work if there is only TokenA/TokenB in the trove.](#M-03)
- ## Low Risk Findings
    - ### [L-01. In some extreme cases, oracles can be taken offline or token prices can fall to zero. Therefore a call to latestRoundData could potentially revert.](#L-01)
    - ### [L-02. `processDeposit()` can cause a DoS if equityAfter is 0 and equityBefore > 0.](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Steadefi

### Dates: Oct 26th, 2023 - Nov 6th, 2023

[See more contest details here](https://www.codehawks.com/contests/clo38mm260001la08daw5cbuf)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 0
   - Medium: 3
   - Low: 2



		
# Medium Risk Findings

## <a id='M-01'></a>M-01. `lpTokenValue` will always calculate for withdrawals even if the action is deposits because `isDeposit` is always passed false            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/oracles/GMXOracle.sol#L234C2-L248C6

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXDeposit.sol#L92

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXDeposit.sol#L136

## Summary
`lpTokenValue` will be always calculated with the factor type `WITHDRAWALS` even if the action is deposit, due to the hardcoded false value passed in the function parameter.

## Vulnerability Details
`lpTokeValue` function defined is `GMXOracle.sol` is responsible for returning the price of the GM token and it calculates this price based on the _pnlFactorType of either withdraw or deposit actions. To determine the type of action, `isDeposit` param is passed to the function. 

```js

function getLpTokenValue(
    address marketToken,
    address indexToken,
    address longToken,
    address shortToken,
    bool isDeposit,//@audit this should be true or false based on action
    bool maximize
  ) public view returns (uint256) {
    bytes32 _pnlFactorType;

```

But wherever this function is called the `isDeposit` is passed always as false which will lead to the incorrect `lpTokenValue`. 

## Proof of concept
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXDeposit.sol#L92

```js
if (dp.token == address(self.lpToken)) {
            // If LP token deposited
            _dc.depositValue = self.gmxOracle.getLpTokenValue(
                address(self.lpToken), 
                address(self.tokenA),
                address(self.tokenA),
                address(self.tokenB), 
                false,//@audit this should be true
                false
            ) * dp.amt / SAFE_MULTIPLIER;
```

When user deposit `lpToken` to GMXVault the `isDeposit` is passed as false which will always lead to the incorrect price calculations.

Even in the `calMinMarketSlippage` this value is always hardcoded to false. This can result in the loss of user funds if the slippage is calculated on the wrong `lpTokenPrice`.

```js
function calcMinMarketSlippageAmt(GMXTypes.Store storage self, uint256 depositValue, uint256 slippage)
external
view
returns (uint256)
{
uint256 _lpTokenValue = self.gmxOracle.getLpTokenValue(
address(self.lpToken),
address(self.tokenA),
address(self.tokenA),
address(self.tokenB),
false, //@audit-issue when depositing this should be true otherwise it will calculte for withdraw
false
);


    return depositValue * SAFE_MULTIPLIER / _lpTokenValue * (10000 - slippage) / 10000;
}

```

## Impact
The function is used to calculate the value of user GM Deposit or slippage, so if the price based on `MAX_PNL_FACTOR_FOR_WITHDRAWALS` is low this could lead to the loss of user deposits.  For example if user deposit GM token and value of it is calcualted low for withdraw but it is actually high for deposits, the user will end up with the less `gvTokens(vault token)`.

## Tools Used
manual review
## Recommendations
Always passed appropriate boolean when calculating the `lpTokenValue`, i.e true for deposit and false for withdraw.
## <a id='M-02'></a>M-02. The protocol will mint unnecessary fees if the vault is paused and reopened later.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXReader.sol#L38

## Summary
Unnecessary fees will be minted to the treasury if the vault is paused and reopened later. 
## Vulnerability Details
Based on the test results, the protocol mints 5(this can be more) wei(gvToken) for each `gvToken` every second since the last fee collection. For example, if the `totalSupply` of `gvToken` is 1000000e18 and the time difference between the current block and the last fee collection is 10 seconds, the amount of lp tokens minted as a fee will be ***50000000*** wei in terms of `gvToken`. This is acceptable when the protocol is functioning properly.

```js
function pendingFee(GMXTypes.Store storage self) public view returns (uint256) {
        uint256 totalSupply_ = IERC20(address(self.vault)).totalSupply();
        uint256 _secondsFromLastCollection = block.timestamp - self.lastFeeCollected;
        return (totalSupply_ * self.feePerSecond * _secondsFromLastCollection) / SAFE_MULTIPLIER;
    }
```
However, if the protocol needs to be paused due to a hack or other issues, and then the vault is reopened, let's say after 1 month of being paused, the time difference from `block.timestamp - _secondsFromLastCollection` will be = ***2630000s***

If the first user tries to deposit after the vault reopens, the fees charged will be 1000000e18 * 5 * 2630000 / 1e18 = ***1315000000000***

This is an unnecessary fee generated for the treasury because the vault was paused for a long time, but the fee is still generated without taking that into account. This can result in the treasury consuming a portion of the user shares.
## Impact
This will lead to a loss of user shares for the duration when the vault was not active. The severity of the impact depends on the fee the protocol charges per second, the totalSupply of vault tokens, and the duration of the vault being paused.

## Tools Used
manual review

## Recommendations
If the vault is being reopened, there should be a function to override the _store.lastFeeCollected = block.timestamp; with block.timestamp again.
## <a id='M-03'></a>M-03. `Compound()` will not work if there is only TokenA/TokenB in the trove.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXCompound.sol#L58

## Summary

The compound() function is designed to deposit Long tokens, Short tokens, or airdropped ARB tokens to the GMX for compounding. However, it will only work if there is ARB token in the trove. If there are only Long/Short tokens in the trove without any ARB, the function will not work.

## Vulnerability Details

The `compound()` function is intended to be called by the keeper once a day to deposit all the Long/Short or ARB tokens to the GMX for further compounding. However, the logic for depositing to the GMX is restricted by the condition that the trove must always hold an airdropped ARB token.

Here is the relevant code snippet from the GitHub repository:
```js
//@audit compound if only ARB is there, what about tokenA and tokenB?
if (_tokenInAmt > 0) {
      self.refundee = payable(msg.sender);

      self.compoundCache.compoundParams = cp;

      ISwap.SwapParams memory _sp;

      _sp.tokenIn = cp.tokenIn;
      _sp.tokenOut = cp.tokenOut;
      _sp.amountIn = _tokenInAmt;
      _sp.amountOut = 0; // amount out minimum calculated in Swap
      _sp.slippage = self.minSlippage;
      _sp.deadline = cp.deadline;

      GMXManager.swapExactTokensForTokens(self, _sp);

      GMXTypes.AddLiquidityParams memory _alp;

      _alp.tokenAAmt = self.tokenA.balanceOf(address(this));
      _alp.tokenBAmt = self.tokenB.balanceOf(address(this));

      self.compoundCache.depositValue = GMXReader.convertToUsdValue(
        self,
        address(self.tokenA),
        self.tokenA.balanceOf(address(this))
      )
      + GMXReader.convertToUsdValue(
        self,
        address(self.tokenB),
        self.tokenB.balanceOf(address(this))
      );

      GMXChecks.beforeCompoundChecks(self);

      self.status = GMXTypes.Status.Compound;

      _alp.minMarketTokenAmt = GMXManager.calcMinMarketSlippageAmt(
        self,
        self.compoundCache.depositValue,
        cp.slippage
      );

      _alp.executionFee = cp.executionFee;

      self.compoundCache.depositKey = GMXManager.addLiquidity(
        self,
        _alp
      );
    }

```

The code checks if there is a positive `_tokenInAmt` (representing ARB tokens) and proceeds with the depositing and compounding logic. However, if there is no ARB token but only tokenA and tokenB in the trove, the compounding will not occur and the tokens will remain in the compoundGMX contract indefinitely.

It is important to note that the airdrop of ARB tokens is a rare event, making it less likely for this condition to be met. Therefore, if there are no ARB tokens but a significant amount of tokenA and tokenB in the trove, the compounding will not take place.

## Impact
If the compounding doesn’t happen this could lead to the indirect loss of funds to the user and loss of gas for the keeper who always calls this function just to transfer tokens and check the balance of ARB.

## Tools Used
manual review

## Recommendations

To mitigate this issue, it is important to always check if either tokenA/tokenB or ARB is present in the trove. If either of these is present, then proceed with the compound action. Otherwise, return.

```js
if (_tokenInAmt > 0 || self.tokenA.balanceOf(address(this) > 0 || self.tokenB.balanceOf(address(this))  ) {
      self.refundee = payable(msg.sender);

      self.compoundCache.compoundParams = cp;

      ISwap.SwapParams memory _sp;

      _sp.tokenIn = cp.tokenIn;
      _sp.tokenOut = cp.tokenOut;
      _sp.amountIn = _tokenInAmt;
      _sp.amountOut = 0; // amount out minimum calculated in Swap
      _sp.slippage = self.minSlippage;
      _sp.deadline = cp.deadline;

      GMXManager.swapExactTokensForTokens(self, _sp);

      GMXTypes.AddLiquidityParams memory _alp;

      _alp.tokenAAmt = self.tokenA.balanceOf(address(this));
      _alp.tokenBAmt = self.tokenB.balanceOf(address(this));

      self.compoundCache.depositValue = GMXReader.convertToUsdValue(
        self,
        address(self.tokenA),
        self.tokenA.balanceOf(address(this))
      )
      + GMXReader.convertToUsdValue(
        self,
        address(self.tokenB),
        self.tokenB.balanceOf(address(this))
      );

      GMXChecks.beforeCompoundChecks(self);

      self.status = GMXTypes.Status.Compound;

      _alp.minMarketTokenAmt = GMXManager.calcMinMarketSlippageAmt(
        self,
        self.compoundCache.depositValue,
        cp.slippage
      );

      _alp.executionFee = cp.executionFee;

      self.compoundCache.depositKey = GMXManager.addLiquidity(
        self,
        _alp
      );
    }
```


# Low Risk Findings

## <a id='L-01'></a>L-01. In some extreme cases, oracles can be taken offline or token prices can fall to zero. Therefore a call to latestRoundData could potentially revert.            



## Summary
In some extreme cases, oracles can be taken offline or token prices can fall to zero. Therefore a call to chainlink’s latestRoundData could potentially revert and none of the circuit breakers would fallback to query any prices automatically.
## Vulnerability Details
The chainlink oracle is used in the system to convert the tokens into USD value. But the protocols assume that chainlink oracle never reverts and always returns something as `answer` when calling `latestRoundData` .

The issue arises from the possibility that Chainlink multisignature entities might intentionally block access to the price feed. In such a scenario, the invocation of the `latestRoundData` function could potentially trigger a revert, making checks like `_chainlinkIsFrozen()`  and `_badChainlinkResponse()` ineffective in mitigating the consequences, as they would be incapable of querying any price data or specific information.

In certain exceptional circumstances, Chainlink has already taken the initiative to temporarily suspend specific oracles. As an illustrative instance, during the UST collapse incident, Chainlink opted to halt the UST/ETH price oracle to prevent the dissemination of erroneous data to various protocols.

Additionally, these dangerous oracle's scenarios are very well documented by OpenZeppelin in https://blog.openzeppelin.com/secure-smart-contract-guidelines-the-dangers-of-price-oracles.

### Proof of concept

Here is the implementation of the chainlink function.

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/oracles/ChainlinkOracle.sol#L151C2-L170C4

Wrapper function where this data is consumed 

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXReader.sol#L62

Some places where this is used.

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXDeposit.sol#L104

https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXManager.sol#L88

## Impact
If a configured Oracle feed has malfunctioned or ceased operating, it will produce a revert when checking for `latestRoundData` that would need to be manually handled by the system.

## Tools Used
Manual review & previous audit example
## Recommendations
Encase the invocation of the function `latestRoundData()` within a `try-catch` construct instead of invoking it directly. In circumstances where the function call results in a revert, the catch block may serve the purpose of invoking an alternative oracle or managing the error in a manner that is deemed appropriate for the system.

example: https://github.com/liquity/dev/blob/main/packages/contracts/contracts/PriceFeed.sol#L522-L540

## <a id='L-02'></a>L-02. `processDeposit()` can cause a DoS if equityAfter is 0 and equityBefore > 0.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/0f909e2f0917cb9ad02986f631d622376510abec/contracts/strategy/gmx/GMXProcessDeposit.sol#L28

## Summary
When minting the user's `gvToken` in `ProcessDeposit.sol`, if equityAfter is 0 and equityBefore is a positive number, an evmRevert will occur due to arithmetic underflow.
## Vulnerability Details
The calculation for minting the user's gvToken (share token) after the user deposits into the vault is based on the equity value of the vault.
```js
function processDeposit(
    GMXTypes.Store storage self
  ) external {
    self.depositCache.healthParams.equityAfter = GMXReader.equityValue(self);

    self.depositCache.sharesToUser = GMXReader.valueToShares(
      self,
     //@audit if equityAfter is 0 this can cause evmRevert with arithmetic underflow
      self.depositCache.healthParams.equityAfter - self.depositCache.healthParams.equityBefore,
      self.depositCache.healthParams.equityBefore
    );

    GMXChecks.afterDepositChecks(self);
  }
```
If we examine the equity value calculation, it is simply the difference between the GM token value and the total debt value. If the equity value is less than the debt, the function returns 0 to avoid underflow within the function.
```js
function equityValue(GMXTypes.Store storage self) public view returns (uint256) {
        (uint256 _tokenADebtAmt, uint256 _tokenBDebtAmt) = debtAmt(self);

        uint256 assetValue_ = assetValue(self); //total value of GM held by vault

        uint256 _debtValue = convertToUsdValue(self, address(self.tokenA), _tokenADebtAmt)
            + convertToUsdValue(self, address(self.tokenB), _tokenBDebtAmt); //debt taken from lending vault

        // in underflow condition return 0
        unchecked {
            if (assetValue_ < _debtValue) return 0; //@audit returns 0 if debt > equity

            return assetValue_ - _debtValue;
        }
    }
```
After a deposit, if `_debtValue` is less than `assetValue`, then `equityValue` will return 0. This value is used in the `processDeposit` function, so `0 - equityBefore` will always result in an underflow, causing a DoS of the system.

The severity of this issue depends on the GM token value and the debt value of the vault. If the debt is greater, for example, for 10 days, the vault will be unusable for 10 days.


## Impact
DoS of the system until assetValue > _debtValue.
## Tools Used
manual review

## Recommendations
Do not allow the deposit if debt > equity until the rebalance has occurred.



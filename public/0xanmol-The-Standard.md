# The Standard - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01.  Attacker can stake 1 wei of TST and EUROs to make the 


# <a id='contest-summary'></a>Contest Summary

### Sponsor: The Standard

### Dates: Dec 27th, 2023 - Jan 10th, 2024

[See more contest details here](https://www.codehawks.com/contests/clql6lvyu0001mnje1xpqcuvl)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 1
  


# High Risk Findings

## <a id='H-01'></a>H-01.  Attacker can stake 1 wei of TST and EUROs to make the protocol DOS.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L134C4-L142C6

https://github.com/Cyfrin/2023-12-the-standard/blob/91132936cb09ef9bf82f38ab1106346e2ad60f91/contracts/LiquidationPool.sol#L211

## Summary
An attacker can exploit a vulnerability in the code to perform a Denial of Service (DOS) attack on the protocol. By creating multiple unique addresses and staking 1 wei of TST and EUROs, the attacker can fill up the holders array and consume all the available block gas limit, rendering the protocol non-functional.
## Vulnerability Details
When a user stakes TST or EUROs using the `LiqidationPool:increasePosition` function, their address is added to the holders array using the `addUniqueHolder` function if the address is new and has not been added before.

The holders array is later used in the `distributeAssets` function to loop through all the holders and process their positions.

In the `distributeAssets` function, there is a nested loop that iterates through the holders array twice. The first loop calculates the total staked amount in the `getStakeTotal` function, and the second loop processes individual stakes.

```js
function distributeAssets(ILiquidationPoolManager.Asset[] memory _assets, uint256 _collateralRate, uint256 _hundredPC) external payable {
        consolidatePendingStakes();
        (,int256 priceEurUsd,,,) = Chainlink.AggregatorV3Interface(eurUsd).latestRoundData();
        uint256 stakeTotal = getStakeTotal();
        uint256 burnEuros;
        uint256 nativePurchased;
        for (uint256 j = 0; j < holders.length; j++) {
            Position memory _position = positions[holders[j]];
            uint256 _positionStake = stake(_position);
            if (_positionStake > 0) {
                for (uint256 i = 0; i < _assets.length; i++) {
                    ILiquidationPoolManager.Asset memory asset = _assets[i];
                    if (asset.amount > 0) {
                        (,int256 assetPriceUsd,,,) = Chainlink.AggregatorV3Interface(asset.token.clAddr).latestRoundData();
                        uint256 _portion = asset.amount * _positionStake / stakeTotal;
                        uint256 costInEuros = _portion * 10 ** (18 - asset.token.dec) * uint256(assetPriceUsd) / uint256(priceEurUsd)
                            * _hundredPC / _collateralRate;
                        if (costInEuros > _position.EUROs) {
                            _portion = _portion * _position.EUROs / costInEuros;
                            costInEuros = _position.EUROs;
                        }
                        _position.EUROs -= costInEuros;
                        rewards[abi.encodePacked(_position.holder, asset.token.symbol)] += _portion;
                        burnEuros += costInEuros;
                        if (asset.token.addr == address(0)) {
                            nativePurchased += _portion;
                        } else {
                            IERC20(asset.token.addr).safeTransferFrom(manager, address(this), _portion);
                        }
                    }
                }
            }
            positions[holders[j]] = _position;
        }
        if (burnEuros > 0) IEUROs(EUROs).burn(address(this), burnEuros);
        returnUnpurchasedNative(_assets, nativePurchased);
    }
```

```js
function getStakeTotal() private view returns (uint256 _stakes) {
        for (uint256 i = 0; i < holders.length; i++) {
            Position memory _position = positions[holders[i]];
            _stakes += stake(_position);
        }
    }
```

Looping in Solidity can be inefficient, especially when dealing with unbounded loops, as it can consume all the available block gas limit. A single Ethereum block has a maximum gas limit of 30 million, and if the holders array is large enough, it can easily consume all the gas limit, causing the `distributeAssets` function to revert.

The audit page mentions that this issue is known, assuming that it would take a long time to fill up the holders array with true stakers. However, an attacker can exploit this vulnerability by creating thousands of unique addresses and staking 1 wei of both TST and EUROs, filling up the entire holders array and causing the protocol to experience a DOS attack.
## Impact
This vulnerability can lead to the protocol's liquidation and reward distribution being reverted, rendering the protocol non-functional.
## Tools Used
manual review
## Recommendations
It is recommended to implement a limit on the number of holders in the array to prevent this vulnerability from being exploited.




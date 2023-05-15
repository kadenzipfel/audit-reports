NeoTokyo Codearena Report

## Metadata
- Repo: https://github.com/code-423n4/2023-03-neotokyo#overview

## Security

#### Points rewarded for staked BYTES off by two orders of magnitude

##### Severity: High

##### Context:

- https://github.com/code-423n4/2023-03-neotokyo/blob/dfa5887062e47e2d0c801ef33062d44c09f6f36e/contracts/staking/NeoTokyoStaker.sol#LL1098C12-L1098C12

##### Description

###### Impact

Users receive 100 times the documented reward for staking BYTES.

###### Proof of Concept

The documentation states: "Staking participants may also stake BYTES 2.0 tokens into their S1 or S2 Citizens in order to boost the points weight of those Citizens at a rate of 200 BYTES per point."

We can also see that there is a constant value assigned that corresponds with this:

```
// NeoTokyoStaker.sol:L203
uint256 constant private _BYTES_PER_POINT = 200 * 1e18;
```

However, when we actually assign bonus points, we're first multiplying the `amount` by 100:

```
// _stakeBytes() - NeoTokyoStaker.sol:L1098-1100
uint256 bonusPoints = (amount * 100 / _BYTES_PER_POINT);
...
citizenStatus.points += bonusPoints;
```

As a result, when staking bytes, we actually receive 100 times as many points as intended, providing some users a significant unintended advantage.

###### Recommended Mitigation

Simply remove the `* 100` from the `bonusPoints` assignment, i.e.:

```
uint256 bonusPoints = (amount / _BYTES_PER_POINT);
```


#### Can withdraw LP tokens without reducing points

##### Severity: High

##### Context: 

- https://github.com/code-423n4/2023-03-neotokyo/blob/dfa5887062e47e2d0c801ef33062d44c09f6f36e/contracts/staking/NeoTokyoStaker.sol#L1623

##### Description

###### Impact

An attacker may artificially inflate their points as much as they like by exploiting a loss of precision.

###### Proof of Concept

In `_withdrawLP()`, there exists logic to check how many points to decrement the users LP pool position by, according to the amount they took out and the multiplier they had applied.

```
// _withdrawLP() - NeoTokyoStaker.sol:L1623-1627
uint256 points = amount * 100 / 1e18 * lpPosition.multiplier / _DIVISOR;
...
lpPosition.points -= points;
```

The problem however with this logic is that there is a loss of precision in calculating the points such that if the amount is less than 1e16, `points` will be 0.

This allows the following attack to take place:

- Deposit LP tokens
- Make many <0.01 LP token withdrawals
- Repeat as desired to obtain significant stake in total LP pool

Based on the maximum size of withdrawals for this attack to function, <0.01 LP tokens, it may seem unlikely to be possible. However, a very small number of LP tokens may correspond to a very valuable position. Take for example [this transaction](https://etherscan.io/tx/0x44ea2098438f507b6564d32ec369baf14464b95370b28a662e604a6157e908cc/) displaying a deposit of ~$7k in liquidity to the Uniswap v2 USDT/WETH pool, obtaining only ~0.000037 LP tokens. It's also important to consider other ways in which this attack may be possible, e.g. if the LP token has less than 18 decimals.

Therefore, it is easily possible for an attacker to significantly manipulate their `points` in the LP token pool such that they earn a significant portion, or even the entirety, of the rewards assigned to the pool.

###### Recommended Mitigation

It may be necessary to reconfigure `points` to be normalized to 18 decimal places throughout the protocol such that precision isn't lost.


#### Small LP deposits may not assign any points

##### Severity: High

##### Context:

- https://github.com/code-423n4/2023-03-neotokyo/blob/dfa5887062e47e2d0c801ef33062d44c09f6f36e/contracts/staking/NeoTokyoStaker.sol#L1155

##### Description

`_stakeLP()` contains a `points` assignment which calculates the amount of `points` the user should receive for their LP token deposit.

```
// _stakeLP() - NeoTokyoStaker.sol:L1155-1161
uint256 points = amount * 100 / 1e18 * timelockMultiplier / _DIVISOR;
...
stakerLPPosition[msg.sender].points += points;
```

The problem with this logic is that any `amount` below 1e16 LP tokens being deposited will result in 0 points being received.

It may seem like an insignificant amount of LP tokens to stake such that this is unlikely to occur. However, a very small number of LP tokens may correspond to a very valuable position. Take for example [this transaction](https://etherscan.io/tx/0x44ea2098438f507b6564d32ec369baf14464b95370b28a662e604a6157e908cc/) displaying a deposit of ~$7k in liquidity to the Uniswap v2 USDT/WETH pool, obtaining only ~0.000037 LP tokens. It's also possible that the LP token has less than 18 decimals.


#### Off-by-one error in `stake` and `withdraw` methods

##### Severity: Medium

##### Context: 

- https://github.com/code-423n4/2023-03-neotokyo/blob/dfa5887062e47e2d0c801ef33062d44c09f6f36e/contracts/staking/NeoTokyoStaker.sol#L1205
- https://github.com/code-423n4/2023-03-neotokyo/blob/dfa5887062e47e2d0c801ef33062d44c09f6f36e/contracts/staking/NeoTokyoStaker.sol#L1668

##### Description

```
// stake() - NeoTokyoStaker.sol:L1205-1207
if (uint8(_assetType) > 4) {
	revert InvalidAssetType(uint256(_assetType));
}
```

```
// withdraw() - NeoTokyoStaker.sol:L1668-1670
if (uint8(_assetType) == 2 || uint8(_assetType) > 4) {
	revert InvalidAssetType(uint256(_assetType));
}
```

The above revert checks seek to enforce valid `_assetType`'s. They intend to enforce that only the four `AssetType`'s are allowed, excluding `BYTES` in `withdraw`.

```
// NeoTokyoStaker.sol:L266-270
enum AssetType {
	S1_CITIZEN,
	S2_CITIZEN,
	BYTES,
	LP
}
```

In both methods, however, the revert checks mistakenly allow `4` as an `_assetType`, by reverting only if `> 4`, where no 4th index exists in the `AssetType` enum.

It doesn't appear as though any attack can take place as a result of this, since both methods should inevitably revert, but nevertheless it's a logical error which should be fixed.


#### Looping over reward windows may result in DoS by block gas limit

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2023-03-neotokyo/blob/dfa5887062e47e2d0c801ef33062d44c09f6f36e/contracts/staking/NeoTokyoStaker.sol#L1264

##### Description

Each stake and withdrawal transaction can only be executed if all the users accrued pool rewards are calculated and distributed. This logic is done in `getPoolReward`, which is called for each pool and loops over each reward window existing on the pool. There exist no bounds for the number of reward windows and admins are expected to continually add new windows. 

```
for (uint256 i; i < windowCount; ) {
	RewardWindow memory window = pool.rewardWindows[i];

	/*
		If the last reward time is less than the starting time of this 
		window, then the reward was accrued in the previous window.
	*/
	if (lastPoolRewardTime < window.startTime) {
		uint256 currentRewardRate = pool.rewardWindows[i - 1].reward;

		/*
			Iterate forward to the present timestamp over any unclaimed reward 
			windows.
		*/
		for (uint256 j = i; j < windowCount; ) {

			// If the current time falls within this window, complete.
			if (block.timestamp <= window.startTime) {
				unchecked {
					uint256 timeSinceReward = block.timestamp - lastPoolRewardTime;
					totalReward += currentRewardRate * timeSinceReward;	
				}

				// We have no forward goto and thus include this bastardry.
				i = windowCount;
				break;

			// Otherwise, accrue the remainder of this window and iterate.
			} else {
				unchecked {
					uint256 timeSinceReward = window.startTime - lastPoolRewardTime;
					totalReward += currentRewardRate * timeSinceReward;
				}
				currentRewardRate = window.reward;
				lastPoolRewardTime = window.startTime;

				/*
					Handle the special case of overrunning the final window by 
					fulfilling the prior window and then jumping forward to use the 
					final reward window.
				*/
				if (j == windowCount - 1) {
					unchecked {
						uint256 timeSinceReward =
							block.timestamp - lastPoolRewardTime;
						totalReward += currentRewardRate * timeSinceReward;
					}

					// We have no forward goto and thus include this bastardry.
					i = windowCount;
					break;

				// Otherwise, iterate.
				} else {
					window = pool.rewardWindows[j + 1];
				}
			}
			unchecked { j++; }
		}

	/*
		Otherwise, the last reward rate, and therefore the entireity of 
		accrual, falls in the last window.
	*/
	} else if (i == windowCount - 1) {
		unchecked {
			uint256 timeSinceReward = block.timestamp - lastPoolRewardTime;
			totalReward = window.reward * timeSinceReward;
		}
		break;
	}
	unchecked { i++; }
}
```

Consider, for example, a scenario where Alice deposits into each pool, with several of each season of citizens, BYTES, and LP tokens. Alice leaves her position for a few years until one day she decides to withdraw. At this point each pool has several reward pools which must be looped over, all within the same transaction or withdrawal will not be possible. Unfortunately for Alice, the excess of reward windows is such that Alice's withdraw transaction exceeds the block gas limit of 30M and as such it's impossible for her to withdraw.

It is possible for the admin to reconfigure the reward windows, reducing them and allowing Alice to process her withdrawal, but any reconfiguration changes the reward distribution,  compromising the fairness of the protocol by giving some users advantages over others.

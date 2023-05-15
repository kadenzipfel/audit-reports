Ethos Codearena Report

## Metadata

- Repo: https://github.com/code-423n4/2023-02-ethos

## Security

#### Initial vault deposits can be frontrun allowing attacker to steal entire deposit

##### Severity: High

##### Context:

- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L331-L335

##### Description

###### Impact

Attacker can steal initial deposits into the `ReaperVaultV2` contract by frontrunning.

###### Proof of Concept

The first deposit (or any deposit where the `totalSupply` is 0) in a `ReaperVaultV2` contract mints shares equivalent to the amount of tokens being deposited. Every other deposit will mint according to: `(_amount * totalSupply()) / freeFunds`.

```
if (totalSupply() == 0) {
	shares = _amount;
} else {
	shares = (_amount * totalSupply()) / freeFunds; // use "freeFunds" instead of "pool"
}
```

An attacker can take advantage of this logic as follows:
1. Victim goes to make first deposit of 100e18 tokens
2. Attacker frontruns victim by depositing 1 wei, receiving 1 share. In the same transaction, the attacker sends 101e18 tokens to the contract such that the token balance of the contract is 101e18. Since none of the tokens have been deposited in a yield strategy, `freeFunds` is also 101e18.
3. Victims transaction proceeds and the victim receives 0 shares because `100e18 * 1 / 101e18 = 0.9900990099009901` which is rounded down to 0.
4. Attacker now has 100% of the vault shares, corresponding to all 201e18 tokens and withdraws all of the tokens.

Although this logic is simplified to interacting with the deposit method directly, which is gated via `_atLeastRole(DEPOSITOR)`, this attack is still possible via opening troves.

Opening a trove results in sending collateral to `ActivePool`:

```
// openTrove() - BorrowerOperations.sol:L232
_activePoolAddColl(contractsCache.activePool, _collateral, _collAmount);
```

```
// BorrowerOperations.sol:L505-508
function _activePoolAddColl(IActivePool _activePool, address _collateral, uint _amount) internal {
	IERC20(_collateral).safeIncreaseAllowance(address(_activePool), _amount);
	_activePool.pullCollateralFromBorrowerOperationsOrDefaultPool(_collateral, _amount);
}
```

Which triggers a rebalance:

```
// pullCollateralFromBorrowerOperationsOrDefaultPool() - ActivePool.sol:L211
_rebalance(_collateral, 0);
```

Which then makes a deposit into the vault:

```
// _rebalance() - ActivePool.sol:L280
IERC4626(yieldGenerator[_collateral]).deposit(uint256(vars.netAssetMovement), address(this));
```

As we can see, we can perform the above example in an indirect manner by observing troves to be opened in the mempool and frontrunning them by opening our own troves with 1 wei of collateral and sending a larger amount of collateral than the victims deposit directly to the vault contract.

It is further possible for the attacker to continue to steal additional deposits so long as they have the capital to maintain a vault contract balance greater than the current victim deposit, stealing every deposit until an `emergencyShutdown` is enacted.

###### Recommended Mitigation

There are two possible mitigation strategies which can be taken:
1. Take a similar approach to that of Uniswap V2 and send an initial bunch of tokens, e.g. 1000 tokens, to the zero address. [Uniswap V2 implementation](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L119-L124)
2. Revert if the amount of shares to be minted is 0, e.g. `require(shares != 0, "No vault tokens minted")`.


#### Harvest swaps can be sandwiched with unbounded slippage

##### Severity: High

##### Context:

- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L109

##### Description:

###### Impact

Reward tokens can be taken via sandwich attack.

###### Proof of Concept

Vault strategy harvesting logic contains swaps of reward tokens:

```
// _harvestCore() - ReaperStrategyGranarySupplyOnly.sol:L117-125
for (uint256 i = 0; i < numSteps; i = i.uncheckedInc()) {
	address[2] storage step = steps[i];
	IERC20Upgradeable startToken = IERC20Upgradeable(step[0]);
	uint256 amount = startToken.balanceOf(address(this));
	if (amount == 0) {
		continue;
	}
	_swapVelo(step[0], step[1], amount, VELO_ROUTER);
}
```

The swaps being conducted do not enforce output amount of tokens, i.e. they have unbounded slippage:

```
// _swapVelo() - VeloSolidMixin.sol:L41
router.swapExactTokensForTokens(_amount, 0, routes, address(this), block.timestamp);
```

As a result, it's possible to sandwich the harvest transaction as follows:
1. Harvest transaction is broadcasted to the mempool
	1. Initial swap is e.g., token a => token b
2. Attacker sandwiches this transaction
	1. Attacker swaps a large amount of token a => token b using the same pool as the vault strategy contract
	2. Harvest transaction is placed with unbounded slippage, causing the output amount of token b to be significantly less
	3. Attacker swaps token b => token a, profiting off the change in relative pricing from harvest transaction

Given that harvesting rewards can regularly result in receiving significantly less tokens than expected, usage of the strategy to earn interest on positions is largely nullified.

###### Recommended Mitigation

It is recommended that the swap mechanism used during harvesting enforces a minimum amount out, e.g. retrieve the final quoted output for the swap and enforce that amount minus a reasonable amount of slippage.


#### Yield strategy loss results in protocol DoS

##### Severity: High

##### Context: 

- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/ActivePool.sol#L251

##### Description

###### Impact

Fundamental logic in the `ActivePool` contract is DoS'd if profit is negative.

###### Proof of Concept

`ActivePool` calculates profit in `_rebalance()` by subtracting the current allocation of collateral from the amount of assets corresponding to the amount of shares held by `ActivePool`.

```
// _rebalance() - ActivePool.sol:L251
vars.profit = vars.sharesToAssets.sub(vars.currentAllocated);
```

This calculation is done using `SafeMath.sub()` which reverts if it will underflow

```
// SafeMath.sol:L47-49
function sub(uint256 a, uint256 b) internal pure returns (uint256) {
	return sub(a, b, "SafeMath: subtraction overflow");
}

// SafeMath.sol:L62-67
function sub(uint256 a, uint256 b, string memory errorMessage) internal pure returns (uint256) {
	require(b <= a, errorMessage);
	uint256 c = a - b;

	return c;
}
```

This means that anytime the profit is negative, i.e. the strategy yields a loss, `_rebalance` will revert. This disables all functions whose logic routes through `_rebalance` in any way. Logic affected by this includes: opening/closing troves, redeeming collateral, liquidating troves, etc..

The only way to reenable affected logic is to send sufficient tokens directly to the yield vault contract such that `ActivePool`'s share of tokens is greater than or equal to its accounted amount of allocated tokens. There are however, two problems with this solution:

1. The sender of the tokens doesn't necessarily receive any share of the tokens they send, only the corresponding share of vault tokens which they hold either directly or indirectly. This is further exacerbated by the fact that:
2. There may be other EOAs and contracts making use of the same vault contract, resulting in `ActivePool`'s share of the collateral tokens being sent in depending upon their share of the total supply of the vault tokens. This means that the loss throughout the entire vault must be covered even if `ActivePool` only accounts for a small share.

It's important to also consider the possible effects of fundamental protocol logic being disabled, even if only temporary. Particularly considering loss in a strategy may have a greater effect on the overall defi ecosystem. Consider the following scenario:

- Vault strategy is providing liquidity to Aave
- Aave suffers a security incident, resulting in significant losses of underlying collateral
- Prices react to incident, causing liquidations
- Ethos users are unable to react since their funds are locked causing their positions to be liquidated

As we can see, this is a major flaw in the protocol and could result in losses being significantly exacerbated.

###### Recommended Mitigation

It is recommended that signed integers are used for any accounting logic which has the potential to go negative.


#### Deposits to vault with unrealized losses take an immediate loss

##### Severity: Medium

##### Context: 

- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L334

##### Description

###### Impact

Depositors take an immediate loss on their deposit if the vault has unrealized losses

###### Proof of Concept

Deposits made to `ReaperVaultV2` pay out shares corresponding to the amount of tokens deposited multiplied by the total supply of vault tokens divided by the `freeFunds` in the vault.

```
// _deposit() - ReaperVaultV2.sol:334
shares = (_amount * totalSupply()) / freeFunds;
```

A significant chunk of `freeFunds` comes from `totalAllocated`

```
// ReaperVaultV2.sol:L287-289
function _freeFunds() internal view returns (uint256) {
	return balance() - _calculateLockedProfit();
}
```

```
// ReaperVaultV2.sol:L278-280
function balance() public view returns (uint256) {
	return token.balanceOf(address(this)) + totalAllocated;
}
```

Thus `totalAllocated` plays a significant factor in determining the amount of shares the depositor will receive.

The problem with this lies in the fact that `totalAllocated` is not updated during deposit, rather it is only updated in withdrawal and harvest methods, causing depositors to receive an amount of shares based on the last reported values rather than current. As a result, if a user makes a deposit while the strategy has unrealized losses, the withdrawable value of the users deposit will immediately be less than they deposited.

###### Recommended Mitigation

It's recommended that losses are reported before calculating the amount of shares which the depositor should receive, allowing the received amount to be proportional to the actual underlying value.


#### Strategy loss can temporarily DoS vault withdrawals

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L385

##### Description

###### Impact

`ReaperVaultV2` withdrawals become temporarily DoS'd if the strategy to withdraw from has incurred a loss.

###### Proof of Concept

When a withdrawal is being made from `ReaperVaultV2`, if the amount to withdraw exceeds the contract's token balance, withdrawals are made directly from the strategies following a withdrawal queue.

```
// _withdraw() - ReaperVaultV2.sol:L368-408
if (value > token.balanceOf(address(this))) {
	uint256 totalLoss = 0;
	uint256 queueLength = withdrawalQueue.length;
	uint256 vaultBalance = 0;
	for (uint256 i = 0; i < queueLength; i = i.uncheckedInc()) {
		vaultBalance = token.balanceOf(address(this));
		if (value <= vaultBalance) {
			break;
		} 
		
		address stratAddr = withdrawalQueue[i];
		uint256 strategyBal = strategies[stratAddr].allocated;
		if (strategyBal == 0) {
			continue;
		} 
		
		uint256 remaining = value - vaultBalance;
		uint256 loss = IStrategy(stratAddr).withdraw(Math.min(remaining, strategyBal));
		uint256 actualWithdrawn = token.balanceOf(address(this)) - vaultBalance;
		
		// Withdrawer incurs any losses from withdrawing as reported by strat
		if (loss != 0) {
			value -= loss;
			totalLoss += loss;
			_reportLoss(stratAddr, loss);
		} 
		
		strategies[stratAddr].allocated -= actualWithdrawn;
		totalAllocated -= actualWithdrawn;
	} 
	
	vaultBalance = token.balanceOf(address(this));
	if (value > vaultBalance) {
		value = vaultBalance;
	} 
	
	require(
		totalLoss <= ((value + totalLoss) * withdrawMaxLoss) / PERCENT_DIVISOR,
		"Withdraw loss exceeds slippage"
	);
}
```

If the remaining amount to withdraw exceeds the allocated amount to the strategy, it attempts to withdraw the full allocated amount from the strategy.

```
// _withdraw() - ReaperVaultV2.sol:L385
uint256 loss = IStrategy(stratAddr).withdraw(Math.min(remaining, strategyBal));
```

However, if the amount we're trying to withdraw exceeds the total `want` held by the strategy, the transaction will be reverted.

```
// withdraw() - ReaperBaseStrategyV4.sol:L98
require(_amount <= balanceOf(), "Ammount must be less than balance");
```

```
// ReaperStrategyGranarySupplyOnly.sol:L218-236
/**
* @dev Function to calculate the total {want} held by the strat.
* It takes into account both the funds in hand, plus the funds in the lendingPool.
*/
function balanceOf() public view override returns (uint256) {
	return balanceOfPool() + balanceOfWant();
}

function balanceOfWant() public view returns (uint256) {
	return IERC20Upgradeable(want).balanceOf(address(this));
}

function balanceOfPool() public view returns (uint256) {
	(uint256 supply, , , , , , , , ) = IAaveProtocolDataProvider(DATA_PROVIDER).getUserReserveData(
		address(want),
		address(this)
	);
	return supply;
}
```

We can never withdraw the total amount allocated to the strategy, so the strategy will always be marked as having more to withdraw and we will not be able to pass it in the withdrawal queue.

```
// ReaperVaultV2.sol:L378-395
address stratAddr = withdrawalQueue[i];
uint256 strategyBal = strategies[stratAddr].allocated; // Always > 0
if (strategyBal == 0) { // Never fires
	continue;
}

uint256 remaining = value - vaultBalance;
// Doesn't execute unless remaining is less than strategyBal
uint256 loss = IStrategy(stratAddr).withdraw(Math.min(remaining, strategyBal));
uint256 actualWithdrawn = token.balanceOf(address(this)) - vaultBalance;

...

strategies[stratAddr].allocated -= actualWithdrawn; // Never gets to 0
```

Therefore, if amount of total `want` held by the strategy is less than the amount allocated to the strategy, i.e. a loss has been incurred, it will not be possible to withdraw more than the current total `want` held by the strategy from the vault, even if it is only the first in the withdrawal queue of many strategies. As a result, vault withdrawals may be effectively DoS'd until the admin updates the withdrawal queue.

Consider a worst case scenario to understand the impact:
1. Aave has a security breach in which a significant amount of aTokens are stolen
2. This triggers prices to drop and cascading liquidations
3. Among the chaos, users rush to withdraw remaining funds to pay off loans they may have but the withdrawal mechanism will not proceed
4. Admins are asleep and cannot update the withdrawal queue, causing users to incur significant losses as a result of a failure to withdraw in a timely manner

###### Recommended Mitigation

It's recommended that `strategyBal` is set as the minimum value of `strategies[stratAddr].allocated`, and `IStrategy(stratAddr).balanceOf()`. This will allow the loop to continue to the next strategy if there are no remaining tokens to be withdrawn, and will only withdraw at most the available amount from the strategy.


#### Profit distributed to StabilityPool from ActivePool may be unaccounted and permanently locked in the contract

##### Severity: Medium

##### Context: 

- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/ActivePool.sol#L304-L305

##### Description

###### Impact

Profit can be unaccounted for and permanently locked in the contract.

###### Proof of Concept

When rebalancing, `ActivePool` may distribute some profit to the `StabilityPool`.

```
// _rebalance() - ActivePool.sol:L302-306
vars.stabilityPoolSplit = vars.profit.sub(vars.treasurySplit.add(vars.stakingSplit));
if (vars.stabilityPoolSplit != 0) {
	IERC20(_collateral).safeTransfer(stabilityPoolAddress, vars.stabilityPoolSplit);
	IStabilityPool(stabilityPoolAddress).updateRewardSum(_collateral, vars.stabilityPoolSplit);
}
```

After transferring the tokens to `StabilityPool`, it triggers the contract to account for the sent collateral and issue a corresponding amount of OATH.

```
// StabilityPool.sol:L486-500
function updateRewardSum(address _collateral, uint _collToAdd) external override {
	_requireCallerIsActivePool();
	uint totalLUSD = totalLUSDDeposits; // cached to save an SLOAD
	if (totalLUSD == 0) { return; }
	
	_triggerLQTYIssuance(communityIssuance);
	
	(uint collGainPerUnitStaked, ) = _computeRewardsPerUnitStaked(_collateral, _collToAdd, 0, totalLUSD);
	
	_updateRewardSumAndProduct(_collateral, collGainPerUnitStaked, 0); // updates S
	
	uint sum = collAmounts[_collateral].add(_collToAdd);
	collAmounts[_collateral] = sum;
	emit StabilityPoolCollateralBalanceUpdated(_collateral, sum);
}
```

However, the problem with this logic is that if the `StabilityPool` does not have any LUSD deposits, we return early and ignore the necessary accounting logic.

```
// StabilityPool.sol:L488-489
uint totalLUSD = totalLUSDDeposits; // cached to save an SLOAD
if (totalLUSD == 0) { return; }
```

The transaction doesn't revert and the collateral is sent to the `StabilityPool` without accounting for it. As a result, the accounting logic in `StabilityPool` is unaware of any collateral being present, causing the tokens to be permanently locked.

###### Recommended Mitigation

It's recommended that instead of returning, we should revert. e.g.

```
uint totalLUSD = totalLUSDDeposits; // cached to save an SLOAD
require(totalLUSD != 0, "Zero LUSD");
```


#### OATH issuance can be censored or disabled by malicious owner

##### Severity: Medium

##### Context: 

- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/LQTY/CommunityIssuance.sol#L113
- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/LQTY/CommunityIssuance.sol#L120

##### Description

###### Impact

Malicious or compromised owner can censor and disable OATH issuance

###### Proof of Concept

The owner of the `CommunityIssuance` contract is responsible for funding and managing the `distributionPeriod`, having unilateral control over these actions.

```
// CommunityIssuance.sol:L101-122
function fund(uint amount) external onlyOwner {
	require(amount != 0, "cannot fund 0");
	OathToken.transferFrom(msg.sender, address(this), amount);

	// roll any unissued OATH into new distribution
	if (lastIssuanceTimestamp < lastDistributionTime) {
		uint timeLeft = lastDistributionTime.sub(lastIssuanceTimestamp);
		uint notIssued = timeLeft.mul(rewardPerSecond);
		amount = amount.add(notIssued);
	}

	rewardPerSecond = amount.div(distributionPeriod);
	lastDistributionTime = block.timestamp.add(distributionPeriod);
	lastIssuanceTimestamp = block.timestamp;

	emit LogRewardPerSecond(rewardPerSecond);
}

// Owner-only function to update the distribution period
function updateDistributionPeriod(uint256 _newDistributionPeriod) external onlyOwner {
	distributionPeriod = _newDistributionPeriod;
}
```

This gives a malicious or compromised owner the ability to:
1. Disable all issuance by setting the `distributionPeriod` to `type(uint256).max`
2. Censor individuals from receiving issuance by frontrunning `StabilityPool` withdrawal transactions by calling `fund`, pushing the distribution period forward, resulting in withdrawing users not receiving expected OATH issuance

For the sake of this report, we'll focus on number 2.

When the `CommunityIssuance` contract is `fund`ed, it sets the `lastIssuanceTimestamp` to the current timestamp and sets `lastDistributionTime` to `distributionPeriod` seconds in the future.

```
// CommunityIssuance.sol:L113-114
lastDistributionTime = block.timestamp.add(distributionPeriod);
lastIssuanceTimestamp = block.timestamp;
```

This affects issuance because it issues OATH according to the time since the `lastIssuanceTimestamp`, up until the `lastDistributionTimestamp`.

```
// CommunityIssuance.sol:L86-91
if (lastIssuanceTimestamp < lastDistributionTime) {
	uint256 endTimestamp = block.timestamp > lastDistributionTime ? lastDistributionTime : block.timestamp;
	uint256 timePassed = endTimestamp.sub(lastIssuanceTimestamp);
	issuance = timePassed.mul(rewardPerSecond);
	totalOATHIssued = totalOATHIssued.add(issuance);
}
```

We can see now how if a transaction issuing OATH is frontrun by a transaction funding OATH, that the change in `lastIssuanceTimestamp` will cause the `timePassed` to be 0, resulting in 0 OATH issuance. This gives the ability to the owner to censor any user they choose from having OATH issued, simply by calling `fund` with as little as 1 wei of tokens.

###### Recommended Mitigation

It is recommended that this issue is remediated by using a timelock.


#### Vault withdrawals can be censored or disabled by admin

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L258

##### Description

###### Impact

`ReaperVaultV2` admin can censor and disable vault withdrawals.

###### Proof of Concept

The `ReaperVaultV2` admin has full control over the `withdrawalQueue`.

```
// ReaperVaultV2.sol:L258-271
function setWithdrawalQueue(address[] calldata _withdrawalQueue) external {
	_atLeastRole(ADMIN);
	uint256 queueLength = _withdrawalQueue.length;
	require(queueLength != 0, "Queue must not be empty");

	delete withdrawalQueue;
	for (uint256 i = 0; i < queueLength; i = i.uncheckedInc()) {
		address strategy = _withdrawalQueue[i];
		StrategyParams storage params = strategies[strategy];
		require(params.activation != 0, "Invalid strategy address");
		withdrawalQueue.push(strategy);
	}
	emit UpdateWithdrawalQueue(_withdrawalQueue);
}
```

Although the length of the array passed in must not be 0, they can simply include only a strategy which has 0 allocation. Allowing the admin to effectively clear the `withdrawalQueue`.

If the `withdrawalQueue` is cleared, the withdrawing user may only receive, at most, the amount of tokens held by the vault contract, which could be 0, while still burning all the shares which they intend to withdraw from.

```
// _withdraw() - ReaperVaultV2.sol:L365-411
value = (_freeFunds() * _shares) / totalSupply();
_burn(_owner, _shares);

if (value > token.balanceOf(address(this))) {
	uint256 totalLoss = 0;
	uint256 queueLength = withdrawalQueue.length;
	uint256 vaultBalance = 0;
	for (uint256 i = 0; i < queueLength; i = i.uncheckedInc()) {
		vaultBalance = token.balanceOf(address(this));
		if (value <= vaultBalance) {
			break;
		}

		address stratAddr = withdrawalQueue[i];
		uint256 strategyBal = strategies[stratAddr].allocated;
		if (strategyBal == 0) {
			continue;
		}

		uint256 remaining = value - vaultBalance;
		uint256 loss = IStrategy(stratAddr).withdraw(Math.min(remaining, strategyBal));
		uint256 actualWithdrawn = token.balanceOf(address(this)) - vaultBalance;

		// Withdrawer incurs any losses from withdrawing as reported by strat
		if (loss != 0) {
			value -= loss;
			totalLoss += loss;
			_reportLoss(stratAddr, loss);
		}

		strategies[stratAddr].allocated -= actualWithdrawn;
		totalAllocated -= actualWithdrawn;
	}

	vaultBalance = token.balanceOf(address(this));
	if (value > vaultBalance) {
		value = vaultBalance;
	}

	require(
		totalLoss <= ((value + totalLoss) * withdrawMaxLoss) / PERCENT_DIVISOR,
		"Withdraw loss exceeds slippage"
	);
}

token.safeTransfer(_receiver, value);
emit Withdraw(msg.sender, _receiver, _owner, value, _shares);
```

As we can see above, the amount of underlying tokens the withdrawing user receives is at most the sum of tokens held by the vault contract plus the amount held by the strategies in the `withdrawalQueue`. If the vault holds 0 tokens and the admin has effectively cleared the `withdrawalQueue`, the user receives 0 underlying tokens while still burning their shares.

A malicious or compromised admin can use this power to frontrun a large withdrawal with a transaction to clear the withdrawal queue as explained above, causing the withdrawing user to burn their shares, and receiving as little as 0 underlying tokens.

The admin could be motivated to perform this attack because if they have many shares in the vault, this will burn shares while maintaining the underlying collateral, thereby increasing the value of the admins shares.

###### Proof of Concept

It is recommended that a timelock is enforced when updating the withdrawal queue.


#### Malicious price feed owner can arbitrarily manipulate collateral prices

##### Severity: Medium

##### Context: 

- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/PriceFeed.sol#L143-L172

##### Description

###### Impact

The `PriceFeed` owner can arbitrarily set collateral prices to the benefit of themselves or to the detriment of others, causing or preventing significant liquidations.

###### Proof of Concept

The owner of the `PriceFeed` contract can set the chainlink aggregator and tellor caller addresses to whichever address they like, so long as they adhere to the proper interfaces.

```
// PriceFeed.sol:L143-172
function updateChainlinkAggregator(
	address _collateral,
	address _priceAggregatorAddress
) external onlyOwner {
	_requireValidCollateralAddress(_collateral);
	checkContract(_priceAggregatorAddress);
	priceAggregator[_collateral] = AggregatorV3Interface(_priceAggregatorAddress);

	// Explicitly set initial system status
	status[_collateral] = Status.chainlinkWorking;

	// Get an initial price from Chainlink to serve as first reference for lastGoodPrice
	ChainlinkResponse memory chainlinkResponse = _getCurrentChainlinkResponse(_collateral);
	ChainlinkResponse memory prevChainlinkResponse = _getPrevChainlinkResponse(_collateral, chainlinkResponse.roundId, chainlinkResponse.decimals);

	require(!_chainlinkIsBroken(chainlinkResponse, prevChainlinkResponse) && !_chainlinkIsFrozen(chainlinkResponse),
		"PriceFeed: Chainlink must be working and current");

	_storeChainlinkPrice(_collateral, chainlinkResponse);
}

// Admin function to update the TellorCaller.
//
// !!!PLEASE USE EXTREME CARE AND CAUTION!!!
function updateTellorCaller(
	address _tellorCallerAddress
) external onlyOwner {
	checkContract(_tellorCallerAddress);
	tellorCaller = ITellorCaller(_tellorCallerAddress);
}
```

As a result of this unilateral control, a malicious or compromised owner can set these addresses to contracts which they control and return whichever prices they like. This can be used to liquidate positions and/or protect their own positions from liquidation.

Although the `PriceFeed` contract is listed as out of scope, due to the severity of a potential private key compromise, and the simple solution below, this change should be implemented.

###### Recommended Mitigation

It's recommended that owner control is removed and replaced with use of the [Chainlink feed registry](https://docs.chain.link/data-feeds/feed-registry/).


#### Loss of precision in `CommunityIssuance.fund()` can result in tokens being locked in the contract

##### Severity: Medium

##### Context: 

- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/LQTY/CommunityIssuance.sol#L112

##### Description

###### Impact

Loss of precision tracking `rewardPerSecond` can result in OATH tokens being locked in the `CommunityIssuance` contract.

###### Proof of Concept

The `rewardPerSecond` used to issue OATH tokens along a linear distribution schedule is calculated by dividing the amount being funded by the `distributionPeriod`.

```
// fund() - CommunityIssuance.sol:L112
rewardPerSecond = amount.div(distributionPeriod);
```

If the `distributionPeriod` is greater than the `amount`, the `rewardPerSecond` will be <1, causing it to be rounded down to 0. As a result, all the funds sent 

Although `distributionPeriod` is initially set as being 14 days in seconds, `1209600`, it can be updated by the owner to be any amount of time. The ability to increase the `distributionPeriod` to any arbitrary amount increases the likelihood of tokens being locked significantly, and we must consider all possible environments to determine the severity of the vulnerability.

###### Recommended Mitigation

Although this is listed as a known issue, the simplicity and lack of tradeoffs from implementing the following mitigation strategy should serve as a strong reason to make an implementation change.

Remediation of this vulnerability is as simple as replacing the current amount check 
```
// fund() - CommunityIssuance.sol:L102
require(amount != 0, "cannot fund 0");
```
with 
```
require(amount >= distributionPeriod, "results in 0 reward per second");
```


#### `ReaperVault` `feeBPS` is documented as being limited to a maximum of 20 BPS when actual limit is 2000 BPS

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L155
- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L181

##### Description

###### Impact

Strategy `feeBps` can be set 100x higher than it is documented as being intended.

###### Proof of Concept

The revert messages from the above listed lines reads: "Fee cannot be higher than 20 BPS", however the actual logic: `_feeBPS <= PERCENT_DIVISOR / 5`, with `PERCENT_DIVISOR` being a constant value of 10000 allows for a fee bps of up to 2000 bps.

###### Recommended Mitigation

Update require statement to properly enforce a maximum fee bps of 20:

```
require(_feeBPS <= PERCENT_DIVISOR / 500, "Fee cannot be higher than 20 BPS");
```


#### Unused internal method

##### Severity: Low

##### Context: 

- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/BorrowerOperations.sol#L432

##### Description

`BorrowerOperations._getUSDValue` is a internal method which is not called, and thus is dead code.


#### Events not being emitted

##### Severity: Low

##### Context:

- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/ActivePool.sol#L194
- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/ActivePool.sol#L201

##### Description

`ActivePoolLUSDDebtUpdated` events in `ActivePool` are missing the `emit` keyword and thus not actually being emitted as intended.


#### Unused variables

##### Severity: Low

##### Context:

- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/StabilityPool.sol#L339
- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/StabilityPool.sol#L384

##### Description

The above listed `LUSDLoss` variables are unused.


#### Use `safeTransfer/safeTransferFrom` instead of `transfer/transferFrom`

##### Severity: Low

##### Context:

- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/LQTY/CommunityIssuance.sol#L103
- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/LQTY/CommunityIssuance.sol#L127

##### Description

OATH transfers in `CommunityIssuance` contract are not using recommended `safeERC20` logic.


#### Outdated compiler version

##### Severity: Low

##### Description

The contracts in `ethos-core` use solidity v0.6.11, which is quite outdated, with the current version at the time of writing being v0.8.19. It is generally recommended that newer versions of the solidity compiler are used to avoid any bugs and issues which may exist on lower versions.


## Gas Optimizations

#### Pack Config into single storage slot

##### Severity: Gas Optimization

##### Context:

- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/CollateralConfig.sol#L27

##### Description

The current `Config` struct in `CollateralConfig` requires 4 storage slots.

```
struct Config {
	bool allowed;
	uint256 decimals;
	uint256 MCR;
	uint256 CCR;
}
```

The following will pack all the elements into a single storage slot where initializing `collateralConfig`s will only require 1 SSTORE instead of 4, similarly retrieving the config for a collateral will only require 1 SLOAD instead of 4.


#### Redundant negation

##### Severity: Gas Optimization

##### Context:

- https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/abstract/ReaperBaseStrategyv4.sol#L128

##### Description

The following line decrements `repayment` by `-roi`.

```
// ReaperBaseStrategyV4.sol:L128
repayment -= uint256(-roi);
```

This double negation is redundant, and instead should be replaced by a simple increment.

```
repayment += uint256(roi);
```


### Static Analysis Findings

#### Mark storage variables as `immutable` if they never change after contract initialization.

State variables can be declared as constant or immutable. In both cases, the variables cannot be modified after the contract has been constructed. For constant variables, the value has to be fixed at compile-time, while for immutable, it can still be assigned at construction time.

The compiler does not reserve a storage slot for these variables, and every occurrence is inlined by the respective value.

Compared to regular state variables, the gas costs of constant and immutable variables are much lower. For a constant variable, the expression assigned to it is copied to all the places where it is accessed and also re-evaluated each time. This allows for local optimizations. Immutable variables are evaluated once at construction time and their value is copied to all the places in the code where they are accessed. For these values, 32 bytes are reserved, even if they would fit in fewer bytes. Due to this, constant values can sometimes be cheaper than immutable values.


```js

contract GasTest is DSTest {
    Contract0 c0;
    Contract1 c1;
    Contract2  c2;
    
    function setUp() public {
        c0 = new Contract0();
        c1 = new Contract1();
        c2 = new Contract2();
        
    }

    function testGas() public view {
        c0.addValue();
        c1.addImmutableValue();
        c2.addConstantValue();
    }
}

contract Contract0 {
    uint256 val;

    constructor() {
        val = 10000;
    }

    function addValue() public view {
        uint256 newVal = val + 1000;
    }
}

contract Contract1 {
    uint256 immutable val;

    constructor() {
        val = 10000;
    }

    function addImmutableValue() public view {
        uint256 newVal = val + 1000;
    }
}

contract Contract2 {
    uint256 constant val = 10;

    function addConstantValue() public view {
        uint256 newVal = val + 1000;
    }
}

```

##### Gas Report
```js
╭────────────────────┬─────────────────┬──────┬────────┬──────┬─────────╮
│ Contract0 contract ┆                 ┆      ┆        ┆      ┆         │
╞════════════════════╪═════════════════╪══════╪════════╪══════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 54593              ┆ 198             ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg  ┆ median ┆ max  ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ addValue           ┆ 2302            ┆ 2302 ┆ 2302   ┆ 2302 ┆ 1       │
╰────────────────────┴─────────────────┴──────┴────────┴──────┴─────────╯
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract1 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 38514              ┆ 239             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ addImmutableValue  ┆ 199             ┆ 199 ┆ 199    ┆ 199 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract2 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 32287              ┆ 191             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ addConstantValue   ┆ 199             ┆ 199 ┆ 199    ┆ 199 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
```


##### Lines
- Migrations.sol:6




#### Use assembly for math (add, sub, mul, div)

Use assembly for math instead of Solidity. You can check for overflow/underflow in assembly to ensure safety. If using Solidity versions < 0.8.0 and you are using Safemath, you can gain significant gas savings by using assembly to calculate values and checking for overflow/underflow.

```js

contract GasTest is DSTest {
    Contract0 c0;
    Contract1 c1;
    Contract2 c2;
    Contract3 c3;
    Contract4 c4;
    Contract5 c5;
    Contract6 c6;
    Contract7 c7;

    function setUp() public {
        c0 = new Contract0();
        c1 = new Contract1();
        c2 = new Contract2();
        c3 = new Contract3();
        c4 = new Contract4();
        c5 = new Contract5();
        c6 = new Contract6();
        c7 = new Contract7();
    }

    function testGas() public {
        c0.addTest(34598345, 100);
        c1.addAssemblyTest(34598345, 100);
        c2.subTest(34598345, 100);
        c3.subAssemblyTest(34598345, 100);
        c4.mulTest(34598345, 100);
        c5.mulAssemblyTest(34598345, 100);
        c6.divTest(34598345, 100);
        c7.divAssemblyTest(34598345, 100);
    }
}

contract Contract0 {
    //addition in Solidity
    function addTest(uint256 a, uint256 b) public pure {
        uint256 c = a + b;
    }
}

contract Contract1 {
    //addition in assembly
    function addAssemblyTest(uint256 a, uint256 b) public pure {
        assembly {
            let c := add(a, b)

            if lt(c, a) {
                mstore(0x00, "overflow")
                revert(0x00, 0x20)
            }
        }
    }
}

contract Contract2 {
    //subtraction in Solidity
    function subTest(uint256 a, uint256 b) public pure {
        uint256 c = a - b;
    }
}

contract Contract3 {
    //subtraction in assembly
    function subAssemblyTest(uint256 a, uint256 b) public pure {
        assembly {
            let c := sub(a, b)

            if gt(c, a) {
                mstore(0x00, "underflow")
                revert(0x00, 0x20)
            }
        }
    }
}

contract Contract4 {
    //multiplication in Solidity
    function mulTest(uint256 a, uint256 b) public pure {
        uint256 c = a * b;
    }
}

contract Contract5 {
    //multiplication in assembly
    function mulAssemblyTest(uint256 a, uint256 b) public pure {
        assembly {
            let c := mul(a, b)

            if lt(c, a) {
                mstore(0x00, "overflow")
                revert(0x00, 0x20)
            }
        }
    }
}

contract Contract6 {
    //division in Solidity
    function divTest(uint256 a, uint256 b) public pure {
        uint256 c = a * b;
    }
}

contract Contract7 {
    //division in assembly
    function divAssemblyTest(uint256 a, uint256 b) public pure {
        assembly {
            let c := div(a, b)

            if gt(c, a) {
                mstore(0x00, "underflow")
                revert(0x00, 0x20)
            }
        }
    }
}


```

##### Gas Report

```js

╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract0 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 40493              ┆ 233             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ addTest            ┆ 303             ┆ 303 ┆ 303    ┆ 303 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract1 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 37087              ┆ 216             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ addAssemblyTest    ┆ 263             ┆ 263 ┆ 263    ┆ 263 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract2 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 40293              ┆ 232             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ subTest            ┆ 300             ┆ 300 ┆ 300    ┆ 300 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract3 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 37287              ┆ 217             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ subAssemblyTest    ┆ 263             ┆ 263 ┆ 263    ┆ 263 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract4 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 41893              ┆ 240             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ mulTest            ┆ 325             ┆ 325 ┆ 325    ┆ 325 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract5 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 37087              ┆ 216             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ mulAssemblyTest    ┆ 265             ┆ 265 ┆ 265    ┆ 265 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract6 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 41893              ┆ 240             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ divTest            ┆ 325             ┆ 325 ┆ 325    ┆ 325 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract7 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 37287              ┆ 217             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ divAssemblyTest    ┆ 265             ┆ 265 ┆ 265    ┆ 265 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯

```


##### Lines
- ReaperVaultERC4626.sol:53
- ReaperVaultERC4626.sol:125
- ReaperVaultERC4626.sol:68
- ReaperVaultERC4626.sol:272
- ReaperVaultERC4626.sol:68
- ReaperVaultERC4626.sol:53
- ReaperVaultERC4626.sol:82
- ReaperVaultERC4626.sol:141
- ReaperVaultERC4626.sol:185
- MultiTroveGetter.sol:44
- MultiTroveGetter.sol:53
- ReaperVaultV2.sol:386
- ReaperVaultV2.sol:405
- ReaperVaultV2.sol:421
- ReaperVaultV2.sol:231
- ReaperVaultV2.sol:244
- ReaperVaultV2.sol:526
- ReaperVaultV2.sol:296
- ReaperVaultV2.sol:245
- ReaperVaultV2.sol:330
- ReaperVaultV2.sol:405
- ReaperVaultV2.sol:419
- ReaperVaultV2.sol:155
- ReaperVaultV2.sol:235
- ReaperVaultV2.sol:466
- ReaperVaultV2.sol:528
- ReaperVaultV2.sol:334
- ReaperVaultV2.sol:421
- ReaperVaultV2.sol:279
- ReaperVaultV2.sol:324
- ReaperVaultV2.sol:334
- ReaperVaultV2.sol:524
- ReaperVaultV2.sol:463
- ReaperVaultV2.sol:535
- ReaperVaultV2.sol:288
- ReaperVaultV2.sol:125
- ReaperVaultV2.sol:231
- ReaperVaultV2.sol:365
- ReaperVaultV2.sol:156
- ReaperVaultV2.sol:181
- ReaperVaultV2.sol:440
- ReaperVaultV2.sol:421
- ReaperVaultV2.sol:384
- ReaperVaultV2.sol:296
- ReaperVaultV2.sol:365
- ReaperVaultV2.sol:419
- ReaperVaultV2.sol:440
- ReaperVaultV2.sol:466
- ReaperVaultV2.sol:533
- ReaperVaultV2.sol:405
- ReaperVaultV2.sol:533
- ReaperVaultV2.sol:125
- ReaperVaultV2.sol:237
- ReaperVaultV2.sol:237
- ReaperVaultV2.sol:463
- ReaperStrategyGranarySupplyOnly.sol:93
- ReaperStrategyGranarySupplyOnly.sol:81
- ReaperStrategyGranarySupplyOnly.sol:100
- ReaperStrategyGranarySupplyOnly.sol:136
- ReaperStrategyGranarySupplyOnly.sol:132
- ReaperStrategyGranarySupplyOnly.sol:223
- ActivePool.sol:145
- ActivePool.sol:277
- ActivePool.sol:145
- ActivePool.sol:277
- TroveManager.sol:55
- TroveManager.sol:54
- TroveManager.sol:54
- TroveManager.sol:55
- PriceFeed.sol:617



#### Use assembly to hash instead of Solidity

```js

contract GasTest is DSTest {
    Contract0 c0;
    Contract1 c1;

    function setUp() public {
        c0 = new Contract0();
        c1 = new Contract1();
    }

    function testGas() public view {
        c0.solidityHash(2309349, 2304923409);
        c1.assemblyHash(2309349, 2304923409);
    }
}

contract Contract0 {
    function solidityHash(uint256 a, uint256 b) public view {
        //unoptimized
        keccak256(abi.encodePacked(a, b));
    }
}

contract Contract1 {
    function assemblyHash(uint256 a, uint256 b) public view {
        //optimized
        assembly {
            mstore(0x00, a)
            mstore(0x20, b)
            let hashedVal := keccak256(0x00, 0x40)
        }
    }
}
```

##### Gas Report

```js
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract0 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 36687              ┆ 214             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ solidityHash       ┆ 313             ┆ 313 ┆ 313    ┆ 313 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract1 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 31281              ┆ 186             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ assemblyHash       ┆ 231             ┆ 231 ┆ 231    ┆ 231 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
```


##### Lines
- HintHelpers.sol:178
- LUSDToken.sol:121
- LUSDToken.sol:120
- LUSDToken.sol:305
- LUSDToken.sol:284
- LUSDToken.sol:283
- ReaperVaultV2.sol:76
- ReaperVaultV2.sol:75
- ReaperVaultV2.sol:74
- ReaperVaultV2.sol:73




#### Use multiple require() statments insted of require(expression && expression && ...)
You can safe gas by breaking up a require statement with multiple conditions, into multiple require statements with a single condition.

```js
contract GasTest is DSTest {
    Contract0 c0;
    Contract1 c1;

    function setUp() public {
        c0 = new Contract0();
        c1 = new Contract1();
    }

    function testGas() public {
        c0.singleRequire(3);
        c1.multipleRequire(3);
    }
}

contract Contract0 {
    function singleRequire(uint256 num) public {
        require(num > 1 && num < 10 && num == 3);
    }
}

contract Contract1 {
    function multipleRequire(uint256 num) public {
        require(num > 1);
        require(num < 10);
        require(num == 3);
    }
}
```

##### Gas Report

```js
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract0 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 35487              ┆ 208             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ singleRequire      ┆ 286             ┆ 286 ┆ 286    ┆ 286 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract1 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 35887              ┆ 210             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ multipleRequire    ┆ 270             ┆ 270 ┆ 270    ┆ 270 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯

```


##### Lines
- RedemptionHelper.sol:304
- LUSDToken.sol:352
- LUSDToken.sol:347
- BorrowerOperations.sol:653
- TroveManager.sol:1539
- PriceFeed.sol:131
- PriceFeed.sol:158



#### Mark storage variables as `constant` if they never change.

State variables can be declared as constant or immutable. In both cases, the variables cannot be modified after the contract has been constructed. For constant variables, the value has to be fixed at compile-time, while for immutable, it can still be assigned at construction time.

The compiler does not reserve a storage slot for these variables, and every occurrence is inlined by the respective value.

Compared to regular state variables, the gas costs of constant and immutable variables are much lower. For a constant variable, the expression assigned to it is copied to all the places where it is accessed and also re-evaluated each time. This allows for local optimizations. Immutable variables are evaluated once at construction time and their value is copied to all the places in the code where they are accessed. For these values, 32 bytes are reserved, even if they would fit in fewer bytes. Due to this, constant values can sometimes be cheaper than immutable values.


```js

contract GasTest is DSTest {
    Contract0 c0;
    Contract1 c1;
    Contract2  c2;
    
    function setUp() public {
        c0 = new Contract0();
        c1 = new Contract1();
        c2 = new Contract2();
        
    }

    function testGas() public view {
        c0.addValue();
        c1.addImmutableValue();
        c2.addConstantValue();
    }
}

contract Contract0 {
    uint256 val;

    constructor() {
        val = 10000;
    }

    function addValue() public view {
        uint256 newVal = val + 1000;
    }
}

contract Contract1 {
    uint256 immutable val;

    constructor() {
        val = 10000;
    }

    function addImmutableValue() public view {
        uint256 newVal = val + 1000;
    }
}

contract Contract2 {
    uint256 constant val = 10;

    function addConstantValue() public view {
        uint256 newVal = val + 1000;
    }
}

```

##### Gas Report
```js
╭────────────────────┬─────────────────┬──────┬────────┬──────┬─────────╮
│ Contract0 contract ┆                 ┆      ┆        ┆      ┆         │
╞════════════════════╪═════════════════╪══════╪════════╪══════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 54593              ┆ 198             ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg  ┆ median ┆ max  ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ addValue           ┆ 2302            ┆ 2302 ┆ 2302   ┆ 2302 ┆ 1       │
╰────────────────────┴─────────────────┴──────┴────────┴──────┴─────────╯
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract1 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 38514              ┆ 239             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ addImmutableValue  ┆ 199             ┆ 199 ┆ 199    ┆ 199 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract2 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 32287              ┆ 191             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ addConstantValue   ┆ 199             ┆ 199 ┆ 199    ┆ 199 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
```



##### Lines
- StabilityPool.sol:160




#### Consider using assembly with overflow/undeflow protection for math (add, sub, mul, div) instead of SafeMath

Consider using assembly for math instead of Solidity. You can check for overflow/underflow in assembly to ensure safety. If using Solidity versions < 0.8.0 and you are using Safemath, you can gain significant gas savings by using assembly to calculate values and checking for overflow/underflow.

```js

contract GasTest is DSTest {
    Contract0 c0;
    Contract1 c1;
    Contract2 c2;
    Contract3 c3;
    Contract4 c4;
    Contract5 c5;
    Contract6 c6;
    Contract7 c7;

    function setUp() public {
        c0 = new Contract0();
        c1 = new Contract1();
        c2 = new Contract2();
        c3 = new Contract3();
        c4 = new Contract4();
        c5 = new Contract5();
        c6 = new Contract6();
        c7 = new Contract7();
    }

    function testGas() public {
        c0.addTest(34598345, 100);
        c1.addAssemblyTest(34598345, 100);
        c2.subTest(34598345, 100);
        c3.subAssemblyTest(34598345, 100);
        c4.mulTest(34598345, 100);
        c5.mulAssemblyTest(34598345, 100);
        c6.divTest(34598345, 100);
        c7.divAssemblyTest(34598345, 100);
    }
}

contract Contract0 {
    //addition in Solidity
    function addTest(uint256 a, uint256 b) public pure {
        uint256 c = a + b;
    }
}

contract Contract1 {
    //addition in assembly
    function addAssemblyTest(uint256 a, uint256 b) public pure {
        assembly {
            let c := add(a, b)

            if lt(c, a) {
                mstore(0x00, "overflow")
                revert(0x00, 0x20)
            }
        }
    }
}

contract Contract2 {
    //subtraction in Solidity
    function subTest(uint256 a, uint256 b) public pure {
        uint256 c = a - b;
    }
}

contract Contract3 {
    //subtraction in assembly
    function subAssemblyTest(uint256 a, uint256 b) public pure {
        assembly {
            let c := sub(a, b)

            if gt(c, a) {
                mstore(0x00, "underflow")
                revert(0x00, 0x20)
            }
        }
    }
}

contract Contract4 {
    //multiplication in Solidity
    function mulTest(uint256 a, uint256 b) public pure {
        uint256 c = a * b;
    }
}

contract Contract5 {
    //multiplication in assembly
    function mulAssemblyTest(uint256 a, uint256 b) public pure {
        assembly {
            let c := mul(a, b)

            if lt(c, a) {
                mstore(0x00, "overflow")
                revert(0x00, 0x20)
            }
        }
    }
}

contract Contract6 {
    //division in Solidity
    function divTest(uint256 a, uint256 b) public pure {
        uint256 c = a * b;
    }
}

contract Contract7 {
    //division in assembly
    function divAssemblyTest(uint256 a, uint256 b) public pure {
        assembly {
            let c := div(a, b)

            if gt(c, a) {
                mstore(0x00, "underflow")
                revert(0x00, 0x20)
            }
        }
    }
}


```

##### Gas Report

```js

╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract0 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 40493              ┆ 233             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ addTest            ┆ 303             ┆ 303 ┆ 303    ┆ 303 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract1 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 37087              ┆ 216             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ addAssemblyTest    ┆ 263             ┆ 263 ┆ 263    ┆ 263 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract2 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 40293              ┆ 232             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ subTest            ┆ 300             ┆ 300 ┆ 300    ┆ 300 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract3 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 37287              ┆ 217             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ subAssemblyTest    ┆ 263             ┆ 263 ┆ 263    ┆ 263 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract4 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 41893              ┆ 240             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ mulTest            ┆ 325             ┆ 325 ┆ 325    ┆ 325 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract5 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 37087              ┆ 216             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ mulAssemblyTest    ┆ 265             ┆ 265 ┆ 265    ┆ 265 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract6 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 41893              ┆ 240             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ divTest            ┆ 325             ┆ 325 ┆ 325    ┆ 325 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯
╭────────────────────┬─────────────────┬─────┬────────┬─────┬─────────╮
│ Contract7 contract ┆                 ┆     ┆        ┆     ┆         │
╞════════════════════╪═════════════════╪═════╪════════╪═════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 37287              ┆ 217             ┆     ┆        ┆     ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg ┆ median ┆ max ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ divAssemblyTest    ┆ 265             ┆ 265 ┆ 265    ┆ 265 ┆ 1       │
╰────────────────────┴─────────────────┴─────┴────────┴─────┴─────────╯

```


##### Lines
- LUSDToken.sol:316
- LUSDToken.sol:331
- LUSDToken.sol:323
- LUSDToken.sol:248
- LUSDToken.sol:332
- LUSDToken.sol:324
- LUSDToken.sol:238
- LUSDToken.sol:243
- LUSDToken.sol:315
- SortedTroves.sol:197
- SortedTroves.sol:150
- LQTYStaking.sol:159
- LQTYStaking.sol:209
- LQTYStaking.sol:193
- LQTYStaking.sol:219
- LQTYStaking.sol:155
- LQTYStaking.sol:183
- LQTYStaking.sol:219
- LQTYStaking.sol:181
- LQTYStaking.sol:191
- LQTYStaking.sol:124
- LQTYStaking.sol:209
- LQTYStaking.sol:120
- LQTYStaking.sol:191
- LQTYStaking.sol:181
- LQTYStaking.sol:219
- LQTYStaking.sol:209
- CollSurplusPool.sol:102
- CollSurplusPool.sol:87
- CollSurplusPool.sol:111
- DefaultPool.sol:97
- DefaultPool.sol:111
- DefaultPool.sol:104
- DefaultPool.sol:86
- ActivePool.sol:207
- ActivePool.sol:267
- ActivePool.sol:193
- ActivePool.sol:257
- ActivePool.sol:288
- ActivePool.sol:200
- ActivePool.sol:262
- ActivePool.sol:266
- ActivePool.sol:271
- ActivePool.sol:269
- ActivePool.sol:291
- ActivePool.sol:251
- ActivePool.sol:272
- ActivePool.sol:258
- ActivePool.sol:262
- ActivePool.sol:296
- ActivePool.sol:296
- ActivePool.sol:264
- ActivePool.sol:258
- ActivePool.sol:291
- ActivePool.sol:288
- ActivePool.sol:302
- ActivePool.sol:302
- ActivePool.sol:175
- CommunityIssuance.sol:113
- CommunityIssuance.sol:107
- CommunityIssuance.sol:90
- CommunityIssuance.sol:109
- CommunityIssuance.sol:108
- CommunityIssuance.sol:89
- CommunityIssuance.sol:112
- CommunityIssuance.sol:88
- PriceFeed.sol:493
- PriceFeed.sol:526
- PriceFeed.sol:516
- PriceFeed.sol:512
- PriceFeed.sol:458
- PriceFeed.sol:524
- PriceFeed.sol:440
- PriceFeed.sol:493
- PriceFeed.sol:440
- PriceFeed.sol:440
- PriceFeed.sol:493
- PriceFeed.sol:425



#### Use assembly to write storage values

```js

contract GasTest is DSTest {
    Contract0 c0;
    Contract1 c1;

    function setUp() public {
        c0 = new Contract0();
        c1 = new Contract1();
    }

    function testGas() public {
        c0.updateOwner(0x158B28A1b1CB1BE12C6bD8f5a646a0e3B2024734);
        c1.assemblyUpdateOwner(0x158B28A1b1CB1BE12C6bD8f5a646a0e3B2024734);
    }
}

contract Contract0 {
    address owner = 0xb4c79daB8f259C7Aee6E5b2Aa729821864227e84;

    function updateOwner(address newOwner) public {
        owner = newOwner;
    }
}

contract Contract1 {
    address owner = 0xb4c79daB8f259C7Aee6E5b2Aa729821864227e84;

    function assemblyUpdateOwner(address newOwner) public {
        assembly {
            sstore(owner.slot, newOwner)
        }
    }
}

```

##### Gas Report
```js
╭────────────────────┬─────────────────┬──────┬────────┬──────┬─────────╮
│ Contract0 contract ┆                 ┆      ┆        ┆      ┆         │
╞════════════════════╪═════════════════╪══════╪════════╪══════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 60623              ┆ 261             ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg  ┆ median ┆ max  ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ updateOwner        ┆ 5302            ┆ 5302 ┆ 5302   ┆ 5302 ┆ 1       │
╰────────────────────┴─────────────────┴──────┴────────┴──────┴─────────╯
╭────────────────────┬─────────────────┬──────┬────────┬──────┬─────────╮
│ Contract1 contract ┆                 ┆      ┆        ┆      ┆         │
╞════════════════════╪═════════════════╪══════╪════════╪══════╪═════════╡
│ Deployment Cost    ┆ Deployment Size ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ 54823              ┆ 232             ┆      ┆        ┆      ┆         │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ Function Name      ┆ min             ┆ avg  ┆ median ┆ max  ┆ # calls │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
│ assemblyUpdateOwner┆ 5236            ┆ 5236 ┆ 5236   ┆ 5236 ┆ 1       │
╰────────────────────┴─────────────────┴──────┴────────┴──────┴─────────╯
```


##### Lines
- LUSDToken.sol:170
- LUSDToken.sol:174
- LUSDToken.sol:332
- LUSDToken.sol:106
- LUSDToken.sol:143
- LUSDToken.sol:149
- LUSDToken.sol:102
- LUSDToken.sol:117
- LUSDToken.sol:138
- LUSDToken.sol:178
- LUSDToken.sol:110
- LUSDToken.sol:114
- LUSDToken.sol:156
- LUSDToken.sol:323
- SortedTroves.sol:85
- LQTYStaking.sol:90
- LQTYStaking.sol:193
- LQTYStaking.sol:88
- LQTYStaking.sol:124
- LQTYStaking.sol:159
- LQTYStaking.sol:89
- ReaperVaultV2.sol:620
- ReaperVaultV2.sol:122
- ReaperVaultV2.sol:535
- ReaperVaultV2.sol:537
- ReaperVaultV2.sol:541
- ReaperVaultV2.sol:123
- ReaperVaultV2.sol:569
- ReaperVaultV2.sol:580
- ReaperVaultV2.sol:609
- ReaperVaultV2.sol:630
- ReaperVaultV2.sol:125
- ReaperVaultV2.sol:124
- Migrations.sol:14
- Migrations.sol:18
- CollSurplusPool.sol:58
- CollSurplusPool.sol:59
- CollSurplusPool.sol:56
- CollSurplusPool.sol:57
- DefaultPool.sol:53
- DefaultPool.sol:54
- DefaultPool.sol:52
- StabilityPool.sol:779
- StabilityPool.sol:592
- StabilityPool.sol:537
- StabilityPool.sol:585
- StabilityPool.sol:576
- StabilityPool.sol:454
- StabilityPool.sol:615
- StabilityPool.sol:578
- StabilityPool.sol:529
- ActivePool.sol:97
- ActivePool.sol:96
- ActivePool.sol:122
- ActivePool.sol:147
- ActivePool.sol:146
- ActivePool.sol:100
- ActivePool.sol:134
- ActivePool.sol:148
- ActivePool.sol:101
- ActivePool.sol:102
- ActivePool.sol:98
- ActivePool.sol:99
- ActivePool.sol:103
- CollateralConfig.sol:75
- BorrowerOperations.sol:147
- BorrowerOperations.sol:146
- BorrowerOperations.sol:152
- TroveManager.sol:1417
- TroveManager.sol:1491
- TroveManager.sol:227
- TroveManager.sol:266
- TroveManager.sol:1504
- TroveManager.sol:271
- TroveManager.sol:294
- CommunityIssuance.sol:112
- CommunityIssuance.sol:121
- CommunityIssuance.sol:93
- CommunityIssuance.sol:114
- CommunityIssuance.sol:113
- CommunityIssuance.sol:77
- CommunityIssuance.sol:58
- CommunityIssuance.sol:75
- CommunityIssuance.sol:90
- PriceFeed.sol:137

Asymmetry Codearena Report

## Metadata
- Repo: https://github.com/code-423n4/2023-03-asymmetry

## Security

#### `Reth.sol` can be partially drained via price manipulation

##### Severity: High

##### Context:

- https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/derivatives/Reth.sol#L211-L216

##### Description

###### Impact

Inconsistency between pricing used to determine amount of SafEth to mint and actual amount of rETH tokens obtained allows attacker to manipulate oracle and partially drain `Reth.sol` rETH balance.

###### Proof of Concept

`Reth.ethPerDerivative` returns ETH/rETH pricing of the deposit pool *if the `_amount` passed can be deposited*, otherwise it returns the current Uniswap price.

```
// Reth.sol:L211-216
function ethPerDerivative(uint256 _amount) public view returns (uint256) {
	if (poolCanDeposit(_amount))
		return
			RocketTokenRETHInterface(rethAddress()).getEthValue(10 ** 18);
	else return (poolPrice() * 10 ** 18) / (10 ** 18);
}
```

`SafEth.stake` determines the amount of tokens to mint based on the underlying value of the derivatives contracts.

```
// stake() - SafEth.sol:L68-99
uint256 underlyingValue = 0;

// Getting underlying value in terms of ETH for each derivative
for (uint i = 0; i < derivativeCount; i++)
	underlyingValue +=
		(derivatives[i].ethPerDerivative(derivatives[i].balance()) *
			derivatives[i].balance()) /
		10 ** 18;

uint256 totalSupply = totalSupply();
uint256 preDepositPrice; // Price of safETH in regards to ETH
if (totalSupply == 0)
	preDepositPrice = 10 ** 18; // initializes with a price of 1
else preDepositPrice = (10 ** 18 * underlyingValue) / totalSupply;

uint256 totalStakeValueEth = 0; // total amount of derivatives worth of ETH in system
for (uint i = 0; i < derivativeCount; i++) {
	uint256 weight = weights[i];
	IDerivative derivative = derivatives[i];
	if (weight == 0) continue;
	uint256 ethAmount = (msg.value * weight) / totalWeight;

	// This is slightly less than ethAmount because slippage
	uint256 depositAmount = derivative.deposit{value: ethAmount}();
	uint derivativeReceivedEthValue = (derivative.ethPerDerivative(
		depositAmount
	) * depositAmount) / 10 ** 18;
	totalStakeValueEth += derivativeReceivedEthValue;
}
// mintAmount represents a percentage of the total assets in the system
uint256 mintAmount = (totalStakeValueEth * 10 ** 18) / preDepositPrice;
_mint(msg.sender, mintAmount);
```

The problem with this logic is that to retrieve the underlying value, the pricing of each derivative is retrieved with `derivatives[i].ethPerDerivative(derivatives[i].balance()`. Since the `_amount` passed to `ethPerDerivative` is the balance of the derivative contract, and not necessarily the amount being deposited, it's possible that the actual deposit amount is little enough that it can be deposited while the full balance of the contract may not be able to be deposited, causing a price discrepancy between the price used to determine the SafEth mint amount and the actual price used in obtaining rETH tokens. This price discrepancy opens up the following attack vector:

1. Attacker takes large rETH flashloan (or happens to be a RETH whale)
2. Attacker sells large amount of rETH in uniswap pool used in Reth.sol, causing the price to decrease significantly
3. Attacker calls `stake` with an amount which the rocket deposit pool can process when the total Reth.sol balance is greater than the rocket deposit pool can process
	1. Attacker receives amount of tokens to mint denominated partially in artificially reduced price of rETH in Uniswap pool, resulting in more SafEth being minted than corresponding value added throughout derivatives
4. Attacker reverses initial Uniswap trade, repaying flashloan if necessary
5. Attacker calls `unstake` to withdraw their share of SafEth now that the Uniswap pool is no longer being manipulated
	1. Attacker receives significantly more ETH than they initially staked
6. Attacker repeats the process as long as it's profitable, draining a significant portion of rETH from Reth.sol
	1. Can be repeated atomically such that pausing staking/unstaking will have no effect

###### Recommended Mitigation

Ensure pricing is always consistent between execution and retrieving underlying value.


#### Staking and unstaking may be temporarily DoS'd via external dependencies

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L91
- https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L118

##### Description

Each underlying derivative token has logic to pause deposits. For example, we can see that wSteth deposits may be paused or limited:

```
// Steth - https://etherscan.io/address/0x47ebab13b806773ec2a2d16873e2df770d130b50#code
StakeLimitState.Data memory stakeLimitData = STAKING_STATE_POSITION.getStorageStakeLimitStruct();
require(!stakeLimitData.isStakingPaused(), "STAKING_PAUSED");

if (stakeLimitData.isStakingLimitSet()) {
	uint256 currentStakeLimit = stakeLimitData.calculateCurrentStakeLimit();

	require(msg.value <= currentStakeLimit, "STAKE_LIMIT");

	STAKING_STATE_POSITION.setStorageStakeLimitStruct(
		stakeLimitData.updatePrevStakeLimit(currentStakeLimit - msg.value)
	);
}
```

Similarly, withdrawals are processed via curve/uniswap pools, enforcing a minimum amount of ETH to receive given a maximum slippage. It's possible that the token price may be low enough that the swap fails.

```
// withdraw() - WstEth.sol:L60-61
uint256 minOut = (stEthBal * (10 ** 18 - maxSlippage)) / 10 ** 18;
IStEthEthPool(LIDO_CRV_POOL).exchange(1, 0, stEthBal, minOut);
```

If any of the deposits or withdrawals revert due to one of the above reasons, it's not possible to deposit or withdrawal at all until an admin changes the weights and/or changes the max slippage, potentially causing a loss of funds for users.


#### Staking and unstaking may be DoS'd if a faulty derivative contract is added

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2023-03-asymmetry/blob/44b5cd94ebedc187a08884a7f685e950e987261c/contracts/SafEth/SafEth.sol#L182

##### Description

`SafEth.sol` includes logic for adding new derivatives but does not include logic for removing derivatives. 

```
// SafEth.sol:L182-195
function addDerivative(
	address _contractAddress,
	uint256 _weight
) external onlyOwner {
	derivatives[derivativeCount] = IDerivative(_contractAddress);
	weights[derivativeCount] = _weight;
	derivativeCount++;

	uint256 localTotalWeight = 0;
	for (uint256 i = 0; i < derivativeCount; i++)
		localTotalWeight += weights[i];
	totalWeight = localTotalWeight;
	emit DerivativeAdded(_contractAddress, _weight, derivativeCount);
}
```

Both `stake` and `unstake` call `.balance()` on each derivative contract. 

```
// stake() - SafEth.sol:L71-75
for (uint i = 0; i < derivativeCount; i++)
	underlyingValue +=
		(derivatives[i].ethPerDerivative(derivatives[i].balance()) *
			derivatives[i].balance()) /
		10 ** 18;
```

```
// unstake() - SafEth.sol:L113-116
for (uint256 i = 0; i < derivativeCount; i++) {
	// withdraw a percentage of each asset based on the amount of safETH
	uint256 derivativeAmount = (derivatives[i].balance() *
		_safEthAmount) / safEthTotalSupply;
```

The problem is that if an incorrect address is used to add a new derivative, `stake` and `unstake` will revert every time and there's no logic to then remove that derivative. The only solution would be to upgrade the contract to include derivative removal functionality. To avoid this issue, derivative removal functionality should be included in the initial deployment.
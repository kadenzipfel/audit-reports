Wenwin Codearena Report

## Metadata

- Repo: https://github.com/code-423n4/2023-03-wenwin

## Security

#### Staking contract can be drained by staking and exiting with many accounts

##### Severity: High

##### Context: 

- https://github.com/code-423n4/2023-03-wenwin/blob/91b89482aaedf8b8feb73c771d11c257eed997e8/src/staking/Staking.sol#L62

##### Description

###### Impact

Attacker can drain staking contract.

###### Proof of Concept

User staking rewards are calculated using the `earned` method:

```
// Staking.sol:L61-63
function earned(address account) public view override returns (uint256 _earned) {
	return balanceOf(account) * (rewardPerToken() - userRewardPerTokenPaid[account]) / 1e18 + rewards[account];
}
```

The flaw with this method is that it will always return a positive amount as long as the account has a staking token balance, `rewardPerToken` is positive, and the account has not already claimed rewards, i.e. `userRewardPerTokenPaid[account]` is 0. 

A possible attack would proceed as follows:
1. Attacker stakes tokens with account A
2. Attacker calls exit, withdrawing tokens and receiving rewards
	1. Since `earned` will return a positive amount under the above conditions, the attacker will earn some rewards
3. Attacker transfers withdrawn tokens to account B
4. Attacker stakes tokens with account B
5. Attacker call exit, withdrawing tokens and receiving rewards
6. Attacker repeats this process until the contract is drained

This attack can be executed after each draw, allowing an attacker to steal nearly all reward tokens that ever get transferred to the staking contract.

###### Recommended Mitigation

This can be mitigated by simply setting `userRewardPerTokenPaid[account]` to be `rewardPerTokenStored` any time users stake tokens from a starting staking token balance of 0.


#### Lack of randomness on resulting winning ticket numbers

##### Severity: High

##### Context:

- https://github.com/code-423n4/2023-03-wenwin/blob/91b89482aaedf8b8feb73c771d11c257eed997e8/src/TicketUtils.sol#L57-L61

##### Description

###### Impact

Odds obtained from building the winning ticket from the retrieved random number using `reconstructTicket` are not as intended or documented, providing an advantage to math savvy participants and increasing the likelihood of the protocol running out of reward tokens.

###### Proof of Concept

To initially select numbers in `reconstructTicket`, we use the following loop:

```
// TicketUtils.sol:L54-61
uint8[] memory numbers = new uint8[](selectionSize);
uint256 currentSelectionCount = uint256(selectionMax);

for (uint256 i = 0; i < selectionSize; ++i) {
	numbers[i] = uint8(randomNumber % currentSelectionCount);
	randomNumber /= currentSelectionCount;
	currentSelectionCount--;
}
```

This should return an array of numbers with each being any possible number in the selectionMax range. However, in reality, for each subsequent number, the modulus is decremented, giving us values in the following ranges:

`numbers[0]`: 0 to selectionMax - 1
`numbers[1]`: 0 to selectionMax - 2
...
`numbers[n]`: 0 to selectionMax  - n - 1

This reduces the likelihood of higher numbers to be selected, increasing the likelihood of lower numbers. Providing an advantage to savvy participants.

To understand the impact, consider the following scenario:
A lottery is created with a high `selectionSize` relative to `selectionMax`, e.g. `selectionSize = 5` and `selectionMax = 10`. The odds of drawing numbers are as follows:
- 10: 0%
- 9: 1/50 - 2%
- 8: 1/40 - 2.5%
- 7: 1/30 - 3.33%
- 6: 1/20 - 5%
- 5: 1/10 - 10%
- 4: ~15.43%
- 3: ~15.43%
- 2: ~15.43%
- 1: ~15.43%
- 0: ~15.43%

We can see how this provides savvy participants a significantly higher likelihood of winning. 

It's also necessary to consider how this can affect the protocol as a whole. If a participant has a significantly higher chance of winning, then the previously calculated odds of the protocol running out of funds of 0.3% will be significantly higher.

###### Recommended Mitigation

To solve this problem, it's recommended that `currentSelectionCount` is not decremented and that `currentSelectionCount + 1` is used.


#### Missing winning numbers result in jackpot being impossible nearly half the time

##### Severity: High

##### Context:

- https://github.com/code-423n4/2023-03-wenwin/blob/91b89482aaedf8b8feb73c771d11c257eed997e8/src/TicketUtils.sol#L43

##### Description

###### Impact

~47% of all draws result in a jackpot being impossible and secondary prize odds being significantly reduced.

###### Proof of Concept

`TicketUtils.reconstructTicket` is used to build the winning ticket from the retrieved random number. To do this, it attempts to randomly select numbers for each possible value. The problem with the logic here is that there is no prevention of the same number appearing twice in the resulting array. As a result of the way the numbers are saved in the bitmap (or'ing in 1 at the bit for currentNumber), placing a duplicate number does not change the resulting value.

This causes the jackpot to be impossible whenever there's a duplicate value. Additionally, it reduces is the odds of receiving a secondary prize as well.

It may seem unlikely for a duplicate number to occur, however, as calculated using the birthday paradox formula, the odds of a duplicate number occuring with e.g., `selectionSize = 7`  and `selectionMax = 35` are ~47%. This implies that  nearly half of all draws will result in a duplicate number occuring preventing jackpots and significantly reducing odds of secondary prizes.

###### Recommended Mitigation

Enforce no duplicate winning ticket numbers in `reconstructTicket`.


#### Invalid ticket numbers

##### Severity: High

##### Context:

- https://github.com/code-423n4/2023-03-wenwin/blob/91b89482aaedf8b8feb73c771d11c257eed997e8/src/TicketUtils.sol#L17
- https://github.com/code-423n4/2023-03-wenwin/blob/91b89482aaedf8b8feb73c771d11c257eed997e8/src/TicketUtils.sol#L43

##### Description

###### Impact

Actual selectable ticket numbers is one less than intended, and winning numbers can be out of range in either direction, causing jackpots to be impossible for that draw and reducing odds of secondary prizes.

###### Proof of Concept

`isValidTicket` checks for numbers from 0 to `selectionMax` - 1, reverting if there are not `selectionSize` unique non-zero numbers.

```
// isValidTicket() - TicketUtils:L27-32
uint256 ticketSize;
for (uint8 i = 0; i < selectionMax; ++i) {
	ticketSize += (ticket & uint256(1));
	ticket >>= 1;
}
return (ticketSize == selectionSize) && (ticket == uint256(0));
```

Similarly, `reconstructTicket` builds the ticket numbers from the retrieved random number by using the the `selectionMax` value as the starting modulus, enforcing the generated number to be from 0 to `selectionMax` - 1.

```
// reconstructTicket() - TicketUtils.sol:L55-61
uint256 currentSelectionCount = uint256(selectionMax);

for (uint256 i = 0; i < selectionSize; ++i) {
	numbers[i] = uint8(randomNumber % currentSelectionCount);
	randomNumber /= currentSelectionCount;
	currentSelectionCount--;
}
```

With `reconstructTicket`, however, it's possible for a winning number to be set as 0 due to a lack of checks. It's also possible for a number to exceed `selectionMax`.

```
// reconstructTicket() - TicketUtils.sol:L66-72
uint8 currentNumber = numbers[i];
// check current selection for numbers smaller than current and increase if needed
for (uint256 j = 0; j <= currentNumber; ++j) {
	if (selected[j]) {
		currentNumber++;
	}
}
```

In the case that `currentNumber` is the initial max value of `selectionMax` - 1, and two numbers exist in the `selected` array already, `currentNumber` will be incremented up to `selectionMax` + 1.

As explained above, the range of possible ticket numbers to select is reduced by one, and it's possible for a selected winning number to be out of range in both directions. The result of an out of range number is jackpots becoming impossible for the current draw and odds of secondary prizes being significantly reduced.

###### Recommended Mitigation

Enforce checks in both functions that the numbers used are in the intended range.


#### Fixed reward overflows if in excess of 6553 DAI

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2023-03-wenwin/blob/91b89482aaedf8b8feb73c771d11c257eed997e8/src/LotterySetup.sol#L170

##### Description

###### Impact

Fixed rewards in excess of 6553 DAI impossible to set without overflowing.

###### Proof of Concept

When packing fixed rewards in `LotterySetup.packFixedRewards`, the following line exists to reduce the reward value into a uint16 so it can be packed along with the other fixed rewards in a single uint:

```
// packFixedRewards() - LotterySetup:L170
uint16 reward = uint16(rewards[winTier] / divisor);
```

If we try to run this method with `rewards[winTier] = 6554e18`, it will divide it by 1e17 to 65540. Since uint16 has a max value of 65536 (2**16 = 65536), our number will overflow.

Sponsor has noted in discord that these contracts should work generally such that they can be deployed and used with different values, so it should allow secondary prize rewards up to a much higher value. Consider for example a high stakes lottery with 16 numbers to select from. 15/16 matching numbers should pay > 6554 DAI for such an unlikely outcome.


#### Randomness manipulation possible in certain conditions

##### Severity: Medium

##### Context: 

- https://github.com/code-423n4/2023-03-wenwin/blob/91b89482aaedf8b8feb73c771d11c257eed997e8/src/RNSourceController.sol#L60

##### Description

###### Impact

Under certain conditions it's possible to manipulate the random number being retrieved to change resulting winning ticket to benefit attacker.

###### Proof of Concept

After calling `requestRandomNumber`, it's possible to call the `retry` method to re-request a new random number. This is possible if it's been longer than `maxRequestDelay` since the last call, and the `fulfillRandomWords` callback has not yet been received. 

This ability to re-request the random number is in defiance of chainlink's security consideration to not [re-request randomness](https://docs.chain.link/vrf/v2/security#do-not-re-request-randomness).

An attack is possible if:
- `maxRequestDelay` is unsafely set
	- There is no minimum value enforced, so can be set as little as 0
- Network conditions are such that the callback can not be quickly transacted
	- High gas climates, e.g. Black Thursday conditions
	- Chain outage (has happened to polygon in the past)

###### Recommended Mitigation

Do not allow re-requesting of random number.


#### `fulfillRandomWords` should never revert

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2023-03-wenwin/blob/91b89482aaedf8b8feb73c771d11c257eed997e8/src/VRFv2RNSource.sol#L33-L35

##### Description

As per chainlink's security considerations, [fulfillRandomWords must not revert](https://docs.chain.link/vrf/v2/security#fulfillrandomwords-must-not-revert). However, the callback contains a revert in the case that `randomWords.length != 1`.

```
// fulfillRandomWords() - VRFv2RNSource.sol:L33-35
if (randomWords.length != 1) {
	revert WrongRandomNumberCountReceived(requestId, randomWords.length);
}
```

This pattern is unecessary since we're using `randomWords[0]` regardless, and >1 `randomWords.length` would not cause any issues.

```
// fulfillRandomWords() - VRFv2RNSource.sol:L37
fulfill(requestId, randomWords[0]);
```


#### Net profit estimation can lead to higher likelihood of running out of reward tokens

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2023-03-wenwin/blob/91b89482aaedf8b8feb73c771d11c257eed997e8/src/LotteryMath.sol#L35
- https://github.com/code-423n4/2023-03-wenwin/blob/91b89482aaedf8b8feb73c771d11c257eed997e8/src/LotteryMath.sol#L96

##### Description

###### Impact

Using an estimated value corresponding to a volatile actual balance to make payments can result in the actual balance depleting rapidly.

###### Proof of Concept

Net profit is estimated based in part on `expectedPayout` after each draw, with `expectedPayout` being an estimate for the average payout per ticket given the odds of different prizes. 

```
// LotteryMath.sol:L35-56
function calculateNewProfit(
	int256 oldProfit,
	uint256 ticketsSold,
	uint256 ticketPrice,
	bool jackpotWon,
	uint256 fixedJackpotSize,
	uint256 expectedPayout
)
	internal
	pure
	returns (int256 newProfit)
{
	uint256 ticketsSalesToPot = (ticketsSold * ticketPrice).getPercentage(TICKET_PRICE_TO_POT);
	newProfit = oldProfit + int256(ticketsSalesToPot);

	uint256 expectedRewardsOut = jackpotWon
		? calculateReward(oldProfit, fixedJackpotSize, fixedJackpotSize, ticketsSold, true, expectedPayout)
		: calculateMultiplier(calculateExcessPot(oldProfit, fixedJackpotSize), ticketsSold, expectedPayout)
			* ticketsSold * expectedPayout;

	newProfit -= int256(expectedRewardsOut);
}
```

Net profit is then later used to calculate ticket reward bonuses.

```
// LotteryMath.sol:L96-112
function calculateReward(
	int256 netProfit,
	uint256 fixedReward,
	uint256 fixedJackpot,
	uint256 ticketsSold,
	bool isJackpot,
	uint256 expectedPayout
)
	internal
	pure
	returns (uint256 reward)
{
	uint256 excess = calculateExcessPot(netProfit, fixedJackpot);
	reward = isJackpot
		? fixedReward + excess.getPercentage(EXCESS_BONUS_ALLOCATION)
		: fixedReward.getPercentage(calculateMultiplier(excess, ticketsSold, expectedPayout));
}
```

The problem with this is that the actual average payout per ticket may vary significantly from the `expectedPayout`. If the actual average payout is less than expected, the net profit will be reported as being higher than it is, which causes the proportional amount to be paid out to be higher than it should be, further decreasing the actual net profit. This can cause a loop in which the actual net profit continues to decline until the protocol runs out of reward tokens to fund payouts.

Although the possibility of running out of funds is marked as a known issue, the probability doesn't take into account this possible feedback loop and is thereby much higher than estimated.

###### Recommended Mitigation

Net profit should be tracked using actual contract balances rather than estimates.


#### Lack of support for fee-on-transfer and rebasing tokens

##### Severity: Medium

##### Description:

###### Impact

Fee on transfer and rebasing tokens can result in incorrect amounts being transferred to and from the protocol.

###### Proof of Concept

Sponsor has noted that we should consider tokens other than DAI to be used with the protocol. Here we consider fee on transfer and rebasing tokens. 

Each ticket purchased with a fee on transfer token or rebasing token will result in a different amount being received by the protocol than intended. There exist no checks that the actual amount received will be as intended. 

Consider for example a fee on transfer token with a 2% fee on each transfer. When a user buys a ticket, they will pay the ticket cost, but the protocol will only receive 98% of that amount. Later when a user goes to claim a ticket, they will also only receive 98% of the expected amount.

###### Recommended Mitigation

Either:
- Add a check to enforce the change in balance being sufficient to cover e.g. the ticket cost, or
- Add documentation noting not to use fee on transfer or rebasing tokens with the protocol


#### Unchecked transfer of reward token

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2023-03-wenwin/blob/91b89482aaedf8b8feb73c771d11c257eed997e8/src/staking/StakedTokenLock.sol#L55

##### Description

`StakedTokenLock.getReward` transfers the rewardsToken to the contract owner using an unsafe `transfer` method, documenting that it is safe to do so because "only trusted reward token is used". However, the protocol is built to support various tokens and the sponsor has noted that we should take into account other tokens as well. 

`safeTransfer` should be used to avoid issues with certain tokens, e.g. USDT and BUSD.
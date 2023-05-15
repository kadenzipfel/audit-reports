# Inverse Codearena Report

## Metadata

- Repo: https://github.com/code-423n4/2022-10-inverse

## Security Findings

#### Oracle manipulation attacks possible if no `getPrice` calls in two days

##### Severity: HIGH

##### Context:

- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Oracle.sol#L112

##### Description:

`Oracle.getPrice` attempts to get the lowest token price of the day, falling back on the previous day's price if necessary. However, in the case that no `getPrice` calls are made within two days, the price returned is the current price. This allows for an attacker to perform a classic flashloan oracle manipulation attack.

The effect of this attack is somewhat dampened by ChainlinkFeed's oracle aggregation, but in the case that a token is highly concentrated on one exchange or there are enough available tokens to flashloan relative to the size of the pool being manipulated, the dampening effect will be insufficient.

##### Remediation:

It is recommended that Inverse reconsider the oracle logic to prevent oracle manipulation.

#### Incorrect decimal values may be returned from `getPrice`

##### Severity: HIGH

##### Context:

- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Oracle.sol#L119
- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L326
- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L360
- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L596

##### Description:

The `normalizedPrice` returned from `Oracle.getPrice` assumes that price is normalized to 18 decimals, as evident by `price / 1 ether` logic executed where it is returned in `Market`. However, this is not always the case.

The price normalization logic works by taking the `ChainlinkFeed.decimals` and collateral `decimals` and subtracting both values from 36, and computing `uint normalizedPrice = price * (10 ** decimals);`. The fixed value of 36 effectively assumes that `ChainlinkFeed.decimals` will always return 18 so that the resulting `normalizedPrice` always ends up normalized to 18 decimals (36 - 18 == 18). The problem with this assumption is that `ChainlinkFeed.decimals` does not always return 18. For example, the [USDC/USD feed](https://etherscan.io/address/0x8fFfFfd4AfB6115b954Bd326cbe7B4BA576818f6#readContract) returns 8 decimals.

Returning the price normalized with an incorrect amount of decimals will result in, e.g. the collateral value being several orders of magnitude higher than it should be, allowing the user to borrow several orders of magnitude more Dola than they should be able to.

##### Remediation:

It is recommended that Inverse simply replace:

```
uint8 feedDecimals = feeds[token].feed.decimals();
uint8 tokenDecimals = feeds[token].tokenDecimals;
uint8 decimals = 36 - feedDecimals - tokenDecimals;
uint normalizedPrice = price * (10 ** decimals);
```

with:

```
uint256 tokenDecimals = feeds[token].tokenDecimals;
uint normalizedPrice = price * (10 ** (18 - tokenDecimals));
```

#### `Oracle.fixedPrices` mapping can never be safely used

##### Severity: HIGH

##### Context:

- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Oracle.sol#L26
- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Oracle.sol#L113

##### Description:

In `Oracle.getPrice`, if a value in the `fixedPrices` mapping exists for a given token, it simply returns the price from the mapping instead of fetching price data. Using a fixed price between two tokens, Dola and collateral, is never safe since every token is freely traded and can fluctuate in value. This logic is especially concerning given that this represents a relative price between two tokens which may fluctuate in price unboundedly.

In the case that a fixed price is used and one or both of the tokens deviate in price, it is possible to, e.g. borrow beyond what should be possible given the true value of the underlying collateral. Furthermore, in the specific case that Dola deviates in value, this may trigger a positive feedback loop whereby the price is reduced and the reduction in price allows for undercollateralized positions, further reducing the price, until the entire system is drained.

##### Remediation:

It is recommended that Inverse removes this `fixedPrices` logic entirely and enforces usage of the oracle for every collateral.

#### Contracts can circumvent `BorrowController` allowlist

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/BorrowController.sol#L47

##### Description:

In `BorrowController.borrowAllowed`, a check is performed to see if the `msg.sender` is the `tx.origin` in an attempt to allow EOA's to execute logic but not contracts. However, it is possible for a contract to circumvent this by making a call to the protocol from within it's constructor, retaining the contract address as both `msg.sender` and `tx.origin`.

#### Missing lower bound for `replenishmentIncentiveBps`

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L76
- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L564

##### Description:

In the `constructor` of `Market`, the intended lower bound of 1 for `replenishmentIncentiveBps`, as documented in `setReplenismentIncentiveBps`, is missing. This lower bound is responsible for ensuring that the replenisher in `forceReplenish` is rewarded for keeping the system safe.

In the case that the value is set as 0 on deployment and goes unnoticed, there will be no incentive to call `forceReplenish`, possibly resulting in users operating at a deficit without accruing debt.

#### Lack of address(0) checks

##### Severity: Low

##### Context:

- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L130
- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/BorrowController.sol#L26
- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Fed.sol#L48

##### Description:

In several places, setters for addresses controlling the contracts are missing important validation to ensure that `operator`/`gov` do not get unintentionally set as, e.g. `address(0)`. A human error could result in these authorization positions becoming permanently locked.

##### Remediation:

Ideally, a two-step transfer process is used for these authorization positions, but at the very least a check should be included that ensures the address being set is not `address(0)`.

#### User `debt` being decremented before token transfer could be exploited in case of reentrancy

##### Severity: Low

##### Context:

- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L531

##### Description:

In `Market.repay`, the users `debt` gets decremented before transferring Dola to repay the debt. This can be exploited in the case that an attacker manages to reenter the function before the `transferFrom`.

It doesn't appear that there is a way to reenter the function, however, if the Inverse team decides to update `DBR.onRepay` to include a callback, this would be exploitable.

##### Recommendation:

It is recommended that the transfer of Dola is done before decrementing the users debt.

#### Precompute constant hashes

##### Severity: Gas optimization

##### Context:

- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L432
- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L496
- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L105
- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L106
- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L107
- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/DBR.sol#L233

##### Description:

As listed in above context, there are several `keccak256` operations performed on constant values. Instead of performing them on each invocation, these hashes should be precomputed and used directly to save gas.

#### Use `unchecked` blocks when safe to do so

##### Severity: Gas optimization

##### Context:

- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/DBR.sol#L111
- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/DBR.sol#L124
- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/DBR.sol#L137

##### Description:

As listed in above context, there are several places in which math operations are executed in which we can be certain that an overflow/underflow is impossible. In these cases, we should place this logic within `unchecked` blocks to save gas.

#### Avoid recomputing logic which can be passed as parameters

##### Severity: Gas optimization

##### Context:

- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/DBR.sol#L325
- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L559

##### Description:

In `Market.forceReplenish`, the values `deficit` and `replenishmentCost` are computed. Despite the fact that `Market.forceReplenish` is the only possible caller of `DBR.onForceReplenish`, these values are re-computed, when they can be more efficiently passed as parameters.

#### Avoid revert strings greater than 32 bytes

##### Severity: Gas optimization

##### Context:

- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L76

##### Description:

The revert string listed above in context is greater than 32 bytes long, costing additional gas on revert by requiring the management of multiple words in memory.

#### Can use uint256 instead of some smaller sized uints

##### Severity: Gas optimization

##### Context:

- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Oracle.sol#L119

##### Description:

As listed above in context, occasionally there are instances of uints of <256 bits used when not necessary. Usage of smaller sized uints generally requires more gas as the values have to be converted to 256 bits to execute operations on them.

#### Use `msg.sender` directly instead of passing as param

##### Severity: Gas optimization

##### Context:

- https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/BorrowController.sol#L46

#### Description:

In `BorrowController.borrowAllowed`, `msg.sender` gets passed as a param when it would be more efficient to simply use `msg.sender` directly.

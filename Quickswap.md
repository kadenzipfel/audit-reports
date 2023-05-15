# Quickswap Codearena Report

## Metadata

- Repo: https://github.com/code-423n4/2022-09-quickswap

## Security Findings

#### Unprotected `address(this)` checks allow attacker to delegatecall from another contract to spoof values such as the token balances of `AlgebraPool` instances

##### Severity: CRITICAL

##### Context:

- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L71
- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L75
- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L451:L455
- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L612:L614
- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L643:L645

##### Description:

`AlgebraPool` token balance checks are intended to exclusively read the token balances of the `AlgebraPool` instance. However, it is possible for an attacker to make a delegatecall into one of the methods reading the token balance, overriding the `address(this)` check to be read in the context of the calling contract.

##### Remediation:

It is recommended that a check is included to confirm that `address(this)` is the correct address, e.g.:

```
address private immutable original;

constructor() {
    original = address(this);
}

function checkNotDelegateCall() private view {
    require(address(this) == original);
}

modifier noDelegateCall() {
    checkNotDelegateCall();
    _;
}
```

#### Low-level call token transfer may lead to false positive

##### Severity: CRITICAL

##### Context:

- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/libraries/TransferHelper.sol#L21
- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L478:L483
- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L505:L506
- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L548
- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L604
- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L610
- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L658:L664
- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L906
- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L913
- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L929
- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L943

##### Description:

`AlgebraPool` uses `TransferHelper.safeTransfer` to execute token transfers. `safeTransfer` performs a low-level call to the token contract, executing the transfer method. However, low-level calls in solidity will always return `success` if the calling account is non-existent. As a result, calls using this `safeTransfer` method may falsely succeed and continue execution of the method with a failed token transfer in the case that the token contract has been self destructed or simply does not exist.

##### Remediation:

It is recommended that either:

- A check is added to ensure the contract being called exists, or
- A high level transfer call is used in place of the low-level call.

#### Owner role could become indefinitely locked due to missing validation

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraFactory.sol#L77:L80

##### Description:

`AlgebraFactory.setOwner`, does not validate that the owner address being set is correct and could lead to the role becoming locked indefinitely due to human error.

##### Remediation:

It is recommended that a two step ownership transfer process is implemented to ensure that the updated owner does not become locked.

#### Owner role could become indefinitely locked due to missing ownership transfer method

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPoolDeployer.sol#L19
- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPoolDeployer.sol#L26:L29

##### Description:

`AlgebraDeployer`'s owner role is set at deployment and there exists no logic to transfer ownership of the contract. In the case that ownership must be transferred, the owner role could become indefinitely locked.

##### Remediation:

It is recommended that a transfer ownership method is implemented in the `AlgebraDeployer` contract. Ideally, it is a two-step transfer method.

#### Incorrect `require` statements

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L739
- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L743

##### Description:

The following require statements are documented as being greater/less than or equal to the min/max sqrt ratios and the `limitSqrtPrice`. However, they are being enforced as strictly greater/less than.

`require(limitSqrtPrice < currentPrice && limitSqrtPrice > TickMath.MIN_SQRT_RATIO, 'SPL');`
`require(limitSqrtPrice > currentPrice && limitSqrtPrice < TickMath.MAX_SQRT_RATIO, 'SPL');`

##### Remediation:

It is recommended that the above require statements are modified to allow equality, e.g.:

`require(limitSqrtPrice <= currentPrice && limitSqrtPrice >= TickMath.MIN_SQRT_RATIO, 'SPL');`
`require(limitSqrtPrice >= currentPrice && limitSqrtPrice <= TickMath.MAX_SQRT_RATIO, 'SPL');`

#### Unprotected `initialize` method may lead to frontrunning

##### Severity: Medium

##### Context:

- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L193:206

##### Description:

`AlgebraPool.initialize` allows anyone to call, setting the initial price of the pool. An attacker could watch the mempool for pool deployments and subsequent initializations, and frontrun an initialization with an unfair price to then drain the initial deposits.

##### Remediation:

It is recommended that either:

- The price initialization is moved to the constructor, or
- Access controls are added to `initialize`

#### Earned interest to tokens in pool can be stolen

##### Severity: Low

##### Context:

- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L603:615

##### Description:

In `AlgebraPool.swap`, the token balance of the contract is called before and after the `_swapCallback` is executed to enforce the user to transfer an appropriate amount of tokens to the contract. It's possible that in the case of a non-standard ERC-20, an attacker may call a method on the token to accrue interest to the pool contract, such that the balance increases sufficiently for them to execute their swap without actually transferring any tokens.

##### Remediation:

It is recommended that the possible effects of non-standard ERC-20 tokens are well documented to minimize loss of user funds.

#### Balance checks can be optimized to avoid redundant `extcodesize` check

##### Severity: Gas optimization

##### Context:

- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPool.sol#L70:#L76

##### Description:

High-level `balanceOf` methods to check token balances make use of a redundant `extcodesize` check and can be optimized as, e.g.:

```
(bool success, bytes memory data) =
    token0.staticcall(abi.encodeWithSelector(IERC20Minimal.balanceOf.selector, address(this)));
require(success && data.length >= 32);
return abi.decode(data, (uint256));
```

#### The factory contract can inherit the deployer contract to remove unecessary logic

##### Severity: Gas optimization

##### Context:

- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPoolDeployer.sol#L49
- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPoolDeployer.sol#L21:L24

##### Description:

The `AlgebraPoolFactory` contract is intended to be the only allowed executor of `AlgebraPoolDeployer.deploy`, because of this, checks are included to ensure that the sender is the factory contract. Redundant logic can be removed by having the factory contract inherit the deployer contract and making `AlgebraPoolDeployer.deploy` internal.

#### Parameters can be safely deleted after pool deployment

##### Severity: Gas optimization

##### Context:

- https://github.com/code-423n4/2022-09-quickswap/blob/15ea643c85ed936a92d2676a7aabf739b210af39/src/core/contracts/AlgebraPoolDeployer.sol#L44:L53

##### Description:

`AlgebraPoolFactory.deploy` shares the parameters with the `AlgebraPool` instance. However, since these parameters are only used in the pool contract's constructor, they can be deleted once the pool is created, reducing gas consumption.

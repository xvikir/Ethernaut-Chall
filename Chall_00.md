# Ethernaut Level 0 — Hello Ethernaut

## Introduction

**Hello Ethernaut** is Level 0 of OpenZeppelin's Ethernaut Web3 CTF.

This level is an introduction to interacting with Ethereum smart contracts through the browser's developer console.

The main objective is to interact with the deployed level contract, follow the information provided by its functions, discover the password, and use it to authenticate successfully.

Unlike many of the later Ethernaut levels, this one does not require exploiting a vulnerability. Instead, it teaches the basics of smart contract interaction and contract reconnaissance.

### What this level teaches

- How to interact with a deployed smart contract
- How to use the browser developer console
- How to inspect a contract object
- How to use the contract's ABI
- How to call contract functions
- How to interpret function return values
- How public variables can expose data through getter functions
- How to send a transaction to modify contract state

---

## Objective

The objective of this level is to find the password stored by the contract and successfully call:

```javascript
authenticate(password)
```

with the correct password.

---

## 1. Getting the Level Instance

After setting up MetaMask and connecting it to the required test network, I opened the Ethernaut Level 0 — Hello Ethernaut page.

Ethernaut provides a browser console environment that allows us to interact with the deployed contracts.

Before solving the level, I clicked the:

**Get New Instance**

button at the bottom of the page.

Ethernaut then deployed a new instance of the `Hello Ethernaut` contract for my wallet.

Once the deployment transaction was completed, the level contract became available through the `contract` variable in the browser console.


---

## 2. Starting the Investigation

The level instructions suggested calling the contract's `info()` function.

I started with:

```javascript
await contract.info()
```

The contract returned:

```text
"You will find what you need in info1()."
```

This was the first clue.

The contract was essentially giving me a set of hints one by one, where each function would tell me what to investigate next.

Based on the response, the next function to call was:

```javascript
await contract.info1()
```

The contract returned:

```text
"Try info2(), but with "hello" as a parameter."
```

This indicated that `info2()` required an argument, specifically the string `"hello"`.

So I called:

```javascript
await contract.info2("hello")
```

The contract returned:

```text
"The property infoNum holds the number of the next info method to call."
```

This told me that I needed to find the value of `infoNum`.

At this point, I did not initially know whether `infoNum` was a property or a function that could be accessed directly.

Instead of guessing, I decided to inspect the contract object and its ABI.

---

## 3. Inspecting the Contract and ABI

I inspected the contract object by entering:

```javascript
contract
```

The browser console displayed the contract object along with the methods exposed through its ABI.

Among the available functions, I could see:

```text
info()
info1()
info2()
info42()
infoNum()
theMethodName()
method7123949()
password()
authenticate()
```

This was useful because the ABI showed me which functions were actually available to interact with.

The ABI, or **Application Binary Interface**, defines the interface between an application and a smart contract. It provides information about the functions that can be called, their parameters, and their return values.

Since `infoNum()` was present in the contract interface, I could call it directly.

I executed:

```javascript
await contract.infoNum()
```

The returned value was:

```text
42
```

The previous clue said that `infoNum` contains the number of the next information method to call.

Therefore, the value `42` indicated that the next function to investigate was:

```javascript
await contract.info42()
```

---

## 4. Finding `theMethodName()`

I called the function indicated by `infoNum()`:

```javascript
await contract.info42()
```

The contract returned:

```text
"theMethodName is the name of the next method to call."
```

This told me that the next method was named `theMethodName`.

I checked the contract ABI again and confirmed that `theMethodName()` was available.

I then called:

```javascript
await contract.theMethodName()
```

The contract returned:

```text
"The method name is method7123949."
```

This gave me the name of the next function:

```text
method7123949
```

Since I had already inspected the contract ABI and confirmed that `method7123949()` was an available function, I called it:

```javascript
await contract.method7123949()
```

The contract returned:

```text
"If you know the password, submit it to authenticate()."
```

This gave me the next important clue.

The contract was telling me that I needed to find the password and submit it to the `authenticate()` function.

At this point, I checked the contract ABI again to see whether there was a function that could provide the password.

---

## 5. Finding the Password

After inspecting the contract ABI again, I noticed that there was a function called:

```text
password()
```

Since the previous message specifically told me that I needed the password, I tried calling it:

```javascript
await contract.password()
```

The contract returned:

```text
"ethernaut0"
```

Therefore, I had successfully retrieved the password required for authentication:

```text
ethernaut0
```

This was an important observation because the password was accessible through the contract's public interface.

Instead of trying to guess the password, I was able to retrieve it directly by inspecting the functions exposed by the contract.

---

## 6. Authenticating With the Password

Now that I had discovered the password:

```text
ethernaut0
```

I needed to submit it to the `authenticate()` function, as instructed by the previous contract response.

I checked the ABI once more and confirmed that the contract exposed:

```text
authenticate()
```

I then called:

```javascript
await contract.authenticate("ethernaut0")
```

This time, unlike the previous read-only function calls, a transaction was created.

The console returned a transaction object containing information such as:

```text
tx: 0xf32d895dca65f01eff885b488d6c7592d897af2e136da08354dbe46da1e97bba
```

The transaction was successfully submitted and confirmed.

This meant that the password was correct and the contract's authentication state had been successfully updated.

---

## 7. Completing the Level

After successfully calling:

```javascript
await contract.authenticate("ethernaut0")
```

the authentication requirement of the level was satisfied.

I then clicked:

**Submit Instance**

Ethernaut checked the state of my deployed level instance and confirmed that the required condition had been met.

The level was successfully completed.

### Final Password

```text
ethernaut0
```

### Final Function Call

```javascript
await contract.authenticate("ethernaut0")
```

## Level 0 — Completed ✅

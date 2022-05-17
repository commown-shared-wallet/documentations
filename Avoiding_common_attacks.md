# Avoiding Common Attacks

Explaination of how we avoid the common attacks.

---

Legend:
-   :white_square_button: To check
-   :white_check_mark: Validated

## List

###  Saving unencrypted confidential data on the blockchain 
We do not store any confidential information nor on chain, nor off chain.  
:white_check_mark: Done 

### Integer overflow and underflow
Solidity solveds this issue in 0.8.  
:white_check_mark: Done 

### Unchecked call return values  
In the withdraw function of CommownSW.sol
```
(bool success, ) = payable(msg.sender).call{value: _amount}("");
require(success, "transaction failed"); //Require the transactoin success or revert
```
:white_check_mark: Done

### Re-entrancy attacks
In the same function "withdraw" of CommownSW.sol, we use the "non reentrancy pattern" by updating the balance before sending ethers to the caller.
```
balancePerUser[msg.sender] -= _amount; //Update the amount before the transaction is called to avoid reentrancy attack

(bool success, ) = payable(msg.sender).call{value: _amount}("");
require(success, "transaction failed"); //Require the transactoin success or revert
```
:white_check_mark: Done

### Oracle manipulation
We do not use oracle at the time. But we will in the futur so we have to check on that. A way would be to use Oracle call off chain, and to avoid centralized oracle call to call several of them for the same transaction.
:white_check_mark: Done

### Denial Of Service attacks
All loops have a max define by max lenght of owners of 255. Moreover, no call yet to external contracts.
:white_check_mark: Done

### Front Running attacks
We don't have a menacism of rewards for being the first at something for example.  
:white_check_mark: Done

### Replay signatures attacks
At the time of the POC, we do not yet have the signature off chain module. So we will probably exposed to that attack and should take care of it. A common solution is adding a nonce per account to a signed message and asap the message is signed, it increments the nonce.  
See [eip-712](https://eips.ethereum.org/EIPS/eip-712)  
:ballot_box_with_check: To check when implementing

### Function default visibility
Function internals are for proxy upgrade, or technical function like the initialize are covered by modifier.  
:white_check_mark: Done

### Floating pragma
Pragma is fixed at 0.8.13.  
:white_check_mark: Done

### Wrong inheritance
Inheritance from most generic to most specific : Initializable, UUPSUpgradeable, OwnableUpgradeable, IERC721Receiver to avoid linearization problem.  
:white_check_mark: Done

### Unexpected ether balance
We would have to add an ether balance uint inside the contrat and compare it each time we do a withdraw to the real ether balance. If different, that means we have this unexpected ether balance. Althoug, our contract being upgradable we would be able to withdram from it.  
:white_check_mark: Done

### Delegate calls to untrusted sources
The delegate call is used only in the proxy, and not used in the logic contract. We have to check on that in the byte code of a pocket, when we will execute the transaction to ensure we do not execute a delegate call "bytes coded".  
So for now, :white_check_mark: Done, for after :ballot_box_with_check: To check when implementing

### (Regular) calls to untrusted sources
We do not have call for external sources yet.  
:white_check_mark: Done

### Insecure randomness
We do not use randomness.  
:white_check_mark: Done

### Block Timestamp manipulation
We do not use block.timestamp in a mecanism yet, and if so, we will respect the `15sec` rule.  
:white_check_mark: Done

### Contracts with zero code
We use a whitelist system because of the definition of owners, so not really concern at the time.  
:white_check_mark: Done


---

<sup>Made with ♥ by CommOwn Teams.</sup>

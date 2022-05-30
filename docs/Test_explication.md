# Test_explication.md

-   Reminder: to launch tests `npx hardhat test` 
-   Folder test content
    -   [01_CommownSWProxyFactory.ts](#commownsw-proxy-factory)
    -   [02_CommownSW.ts](#commownsw)
-   All tests follow the same structure (simple use case, reverts and require, more complicated use case)
-   All tests are numeroted


# Commown Shared Wallet - Proxy Factory - Tests <a name="commownsw-proxy-factory"></a>

### Goal of the contract
The goal of that contract is to deploy the main logic contract first. Then, when user is creating a CSW, it calls the createProxy function. When users will interact with the logic contract, states variables will be stored in that instance.
So, tests here concerns all these functionnalities.

### 01_CommownSWProxyFactory__01_deployProxyFactory
#### 01_01_01: it deploys the CommownSWProxyFactory and the CommownSW
Single test here to ensure both contracts are deployed
We can access to proxy factory state variable logic representing the logic contract. It's then a success.

### 01_CommownSWProxyFactory__02_createProxy
#### 01__02-01: it deploys a proxy from sign0
Once the CSW Proxy Factory deployed, sign0 calls createProxy to create a proxy, and it does.
#### 01__02-02: it deploys a proxy from sign0 and emit en event
Once the CSW Proxy Factory deployed, sign0 calls createProxy to create a proxy, and it does, and emits an event associated.
#### 01__02-03: it deploys a proxy from sign1
As sign0 is also the owner of the contract factory, this test here is to ensure it works too with another user.

### 01_CommownSWProxyFactory__03_stateVariable
#### 01__03-01: it saves the proxy in the global proxiesList
To ensure our application manage all the proxies created we need to store there contract addresses in a state mapping available in the CSW ProxyFactory.
#### 01__03-02: it saves the proxy in commownProxyPerUser for each 'owners'
Same test than before, but this time we save the proxy address for each user concerned by this CSW creation. Then, when a user connects to our front app, we can retrieve easily his wallet.
#### 01__03-03: it increments the nb of proxy for each 'owners'
Same test than before, we increments the number of proxy a user can have, because in the futur, a user could have several CSW if he wants.
#### 01__03-04: it handles multi wallet created per user i.e. multi proxies
That one is trickier as we create two proxies. The first one, created by sign0, has 3 addresses to define owners : sign0, sign1 and sign2. 2 of these 3 addresses, sign0 and sign1, are used by sign1 to create a second proxy. We get back two different proxy addresses, one for each wallet, and state variable are updated in the proxy factory with sign0 and sign1 getting two proxies, sign3 only got one created by sign0.

### 01_CommownSWProxyFactory__04_upgradeLogicAndProxies
This test block concerns the update and related safety logic. Before starting each test, we deploy the proxy factory and create a proxy for each user. 
#### 01__04_01: it updates the CSW logic contract and all proxies already deployed from the CSW proxy factory
For this first test we make sure that the upgrade mechanism works, that is, that the logical contract is upgradeable, and that we can update each of the proxies with the new implementation address of the logical contract. We use the OZ plugin forceImport function to get back the proxies created by the factory and make them embeddable by the plugin.
#### 01__04_02: it updates the CSW logic address in the Factory contract and new users can use it to create new proxy (CSW)
In this test we perform the same manipulation as in the first test but we make sure that the logical address of the CSW contract is updated in the factory and that the new users who create a CSW get a proxy of the upgraded contract.
#### 01__04_03: After an update of a logic, a pre init contract can not call again initialize method
Ici nous testons la sécurité du contrat logic, du fait qu'il ne puisse pas être réinitialisé par un utilisateur qui l'aurait déjà fait auparavant. Cela permet d'éviter qu'un utilisateur efface ses utilisateurs et retirent les fonds par exemple...

# Commown Shared Wallet - Logic contract - Tests <a name="commownsw"></a>

### Goal of the contract
The goal of that contract is to handle all the logic of a Commown Shared Wallet. So we will find here all the features waited : deposit/withdraw funds, creating a pocket of investment...

### 02_CommownSW__01_deployementAndInitializer
First, we prepare the global use case by deploying the CSW Proxy Factory hence the CSW logic, then we create a proxy from sign0 and get the proxy address. 
#### 02__01-01: in the proxy it reads the initialize state of CommownSW
That test basically confirms we can read state variables in the proxy contract by attaching this proxy address to the logic contract.
#### 02__01-02: Owner(Admin) of CommownSW and CommownSWProxyFactory is sign0
Owners (admin) of the CSW Logic and the CSW Proxy Factory is the sign0 from hardhat. In a future release, a good practice would be to set a multi sig owner for security purposes.
#### 02__01-03: it can transfer ownership
In the case of a changing of governance around these contracts as mentionned above, we will need to transfer ownership.
#### 02__01-04: it handles different state per proxy
We complete here the before each statement by creating a new proxy from sign0, and we want to see each proxy has its own state variables.

### 02_CommownSW__02_ReceiveAndWithdrawETH
This group of test is for testing one of the main features of the CSW : add and retreive investment in ETH.
Before each test, we prepare the global use case by deploying the CSW Proxy Factory hence the CSW logic, then we create a proxy from sign0 and get the proxy address.
#### 02__02-01: it receives ETH for a CSW owner, updates his balance and updates the global balance
We sign a transaction from sign0 sending some ETH and check it updates what it needs to.
#### 02__02-02: it does not receive ETH for a non CSW owner, or 0 wei sent, balances remain unchange
Same test than the previous but this time that transaction is signed by sign3 who is not an owner. Though, it will not emplayu that contract can not receive ethers from another way like a selfdestruct instruct for instance. But not the purpose here for that test.
We check all the require statement via modifier or inside the function.
#### 02__02-03: it emits an event when receiving ETH
An event of deposit is emit in the receive function.
#### 02__02-04: it withdraw ETH for a CSW owner and update balances
We check that after deposit a user can withdraw an amount and updates his balance.
#### 02__02-05: it does not withdraw ETH for a non CSW owner, or 0 wei asked, balances remain unchanged
We check here all the require and revert reason loke 0 amount or a non CSW Owner.
#### 02__02-06: it emits an event when withdrawing ETH
Event emitted when withdraw.
#### 02__02-07: it handles different deposit of different users and withdraw dont affect others
We ensure in that test that when a user does a deposit/withdraw it does not affect other users.

### 02_CommownSW__03_createPocket
A pocket is mandatory to invest in something, because we have to track to whom the transaction is destinated, and the amount fixed each user invest. That way, we can, once the item sold, increments/decrements proportionally users' balance.
For that usecase, we deploy the proxy factory, create a proxy, and we simulate a function call of a smart contract invented for this occasion. We use the Interface from ethers.utils to get the bytes data associated to that call.
#### 02__03-01: it proposes a Pocket which can be retrieve from array
We create a pocket using the before each statement and store that pocket in the array.
#### 02__03-02: it proposes a Pocket which can be retrieve from emitted event
We create a pocket using  the before each statement and emit an event.
#### 02__03-03: it handles all requirement and revert if not validated
We test here all the requires if it is not an owner, mismatch... etc.
#### 02__03-04: creator counts for one vote
We test here the creator of the pocket's vote is incremented by one in the mean time he creates the pocket.
#### 02__03-05: it saves the share per user correctly
We test here it saves for each user the shares with the struc define in CSW

### 02_CommownSW__04_votePocket
#### 02__04-01: it can vote for a Pocket, increments number of vote, isSigned becomes true
This test ensure the vote for a pocket is possible for a CSW owner
#### 02__04-02: it reverts if not an owner
Ensures that only a CSW owner can vote
#### 02__04-03: it reverts if not an existing pocket
Ensures that only a vote is possible for an existing pocket
#### 02__04-04: it reverts if pocket already signed
Ensures a CSW can not vote several times for the same pocket
#### 02__04-05: it reverts if not at the good stage
Ensures it reverts if not at the good stage, for exmeple, voting/signing is closed, or NFT or token already aquired.

# Design pattern

Explaination of the `Commown Shared Wallet`'s design pattern

-   [Proxy pattern](#proxy-pattern)
    -   [Why a proxy contract ?](#why-a-proxy)
    -   [Different type of proxy](#different-proxy)
    -   [OpenZeppelin plugin](#oz-plugin)
    -   [CSW - Proxy factory contract](#csw-proxy-factory)
    -   [CSW - Logic contract](#csw-logic-contract)
-   [Security and behavioral pattern](#security-pattern)
    -   [Guard check, behavioral check and state machine](#detailed-pattern)
    -   [Access restriction](#access-pattern)
-   [Features & heritance](#csw-features)


# Proxy pattern <a name="proxy-pattern"></a>
The Commown Shared Wallet contracts is base on the proxy concept and composed of two contracts.

## Why a proxy contract ? <a name="why-a-proxy"></a>
Contracts are immutable once deployed on the blockchain. That's the safety point every one wants to reach.
But, on the other hand, sometimes, it would be nice to be able to update contract for bug fixing, or patching, or product improvements ask by your community (DAO like). Moreover, as our contract will handle money, it is preferable to have that possibility.
Obviously, all the contract is not updatable and we have to follow some guide lines detailed above.
The process is always the same : 

> One main contract for n proxies.
> All proxy patterns depend on an EVM primitive, the DELEGATECALL opcode.

> Include here schema of difference between "call" and "delegate call"

The main contract contains all the logic of the dapp, and proxies contain state variables of the logic contract.
So, the first step was to choose, which type of proxy we need.

## Different type of proxies <a name="different-proxy"></a>
There are several proxy models, each adapted to different use cases.

### Clone
The "clone" one is the cheapest one, but it's not really a proxy for upgradable purpose. The only thing it does is cloning contract functionality in an immutable way and delegate all calls to the main contract. So it does not allow for upgrade the logic contract. Thus, it is very usefull once you know taht our contract is safe and well designed or when your contract is mature and you are pretty sure it will not need any further upgrades.  
See [eip-1167](https://eips.ethereum.org/EIPS/eip-1167)

### Transparent
The transparent proxy is based on the normal proxy pattern. It is called transparent by OpenZeppelin because of the "non conflit" tools it provides.
It relies on the caller before the function selector, then the transparent proxy recognizes if it has to delegate the call to the main logic contract as if a user was calling a function, and vice versa when it's the owner of the proxy. No conflict then, it's "transparent".
Downside of that proxy is each call requires an additional read from storage to load the admin address which is gas costly. Then, there is another pattern...  
See [transparent proxies](https://blog.openzeppelin.com/the-state-of-smart-contract-upgrades/#transparent-proxies) and [eip-1967](https://eips.ethereum.org/EIPS/eip-1967)

### Universal
Universal upgradeable proxiy standard or UUPS as it stands, comes from [eip-1822](https://eips.ethereum.org/EIPS/eip-1822). It is almost the same pattern than the transparent one, but it places upgrade logic in the implementation contract instead of the proxy itself. Then, it avoids the additionnal storage read.
Since the proxy uses delegate calls, if the implementation address is define in the logic contract, then, it "is" in the proxy storage.  
See [transparent vs uups](https://docs.openzeppelin.com/contracts/4.x/api/proxy#transparent-vs-uups) and [eip-1967](https://eips.ethereum.org/EIPS/eip-1967)

### Beacon
Beacon is the one we hesitate to choose. UUPS is costless than transparent, as seen above, because of the upgradeable logic is in the logic contract. But, when several proxies are deployed, if you want to upgrade the address of the logic contract, you have to upgrade through all proxies. Beacon is a solution for this problem, but beacon is also costly than UUPS. So we have chosen UUPS.  
See [beacon](https://blog.openzeppelin.com/the-state-of-smart-contract-upgrades/#beacons)

## OpenZeppelin plugin <a name="oz-plugin"></a>
OpenZeppelin Upgrades plugin handles the difficulty of upgrading a smart contract. Indeed, there are some rules to follow, for exemple :
- Storage has to be in the same order between first logic contract and the updated version
- Only owner/admin can update the smart contract
- You should not have any selfdestruct or delegatecall in your logic contract as it can induce the lost of the logic contract for all proxies
- You should not have a constructor  
For more guidelines, check the [usefull links](#usefull-links).

### Frontend
So, the plugin mention above is very handy when you want to handle deployment or upgrade of proxies as it checks for all this rules and warns you in case of a danger.
But, you can not use it in the front end. It only works for an admin, and you can not use it in the front, for example after a user action is triggered.
Moreover, this would have been not so safe as a creation proxy pattern.
That's the main reason why we decided to use a proxy factory in solidity, which will be deploy by our team, and be called by users when they would like to create a Commown Shared Wallet.

## CSW - Proxy factory contract <a name="csw-proxy-factory"></a>
### Goal
The proxy factory contract's goal is to create proxies for users willing to build a Commown Shared Wallet. It also manages the proxy list, and saves the bound between a user and his sahred wallet. All this is permited by states mapping : `mapping(address => address[]) public commownProxiesPerUser`,`mapping(address => uint256) public nbProxiesPerUser`.

### Create proxy
Proxy are created using the `ERC1967Proxy` from OpenZeppelin which will call the `initilize` function with a selector, to create an instance of an UUPS.

### For future updates
At this time, the proxy factory is not updatable, that means, if we want to update it we have to replace it with another, and be sure that the storage of the first one is get for the new proxy factory. It would be nicer to have that proxy factory, itself updatable or at least to be able to change that address of the logic contract though it delegates call.

## CSW - Logic contract <a name="csw-logic-contract"></a>
### Being a proxy
The logic contract respect the standard define by OZ.
- `function initialize(address[] memory _owners, uint8 _confirmationNeeded) public initializer` this is the function which is called by the proxy factory when initializing a new Commown Shared Wallet. This is a "constructor" like, and guaranted to be called once with the modifier `initializer`. Inside we have the `__Ownable_init()` and the `__UUPSUpgradeable_init()` to ensure it is also initialized.
- The constructor is empty and annoted with the custom natspec for the OZ plugin
- As it is an UUPS logic contract, it's mandatory to have the `_authorizeUpgrade(address newImplementation) internal override onlyOwner` function to ensure the futur updates possible

### For future updates
- As it is an UUPS logic contract the futur updates contract also has to have the `_authorizeUpgrade(address newImplementation) internal override onlyOwner` function.

For future updates of the logic contract, we have to force the OZ plugin to recognize the list of all proxies, as reals proxies of our logic contract, using the force update method provided in the sdk.

# Security and behavioral pattern <a name="security-pattern"></a>
## Guard check, behavioral check and state machine <a name="detailed-pattern"></a>
Guard check and state machine are used almost every where, and very often, the first one to check all entry params when required and the second one to represent several stages.
- Guard check : `require(_owners[i] != address(0), "owner is address(0)"); //Not the 0 address`
- State machine : `modifier pocketNotExecuted(uint256 _pocketID) {
        require(
            pockets[_pocketID].pStatus != PocketStatus.Executed,
            "Pocket already executed"
        );
        _;
    }`
-  Behavioral : `require(_owners.length > 0, "owners required")`

## Access restriction <a name="access-pattern"></a>
As shown for the proxy upgrade, there are some access restriction guaranted by the modifier onlyOwner. For the rest of functions, there is protection of access called : commownOwners. These are addresses of the owners define while creating a Commown Shared Wallet.  
`modifier isCommownOwner(address _sender) {
        require(isOwner[_sender], "not an owner");
        _;
    }`

# Features and heritance <a name="csw-features"></a>
## Deposit/Withdraw Ether
Providing ether to the wallet by the owners will permit future transactions. Obviously they can withdraw too.
## Creation of a "Pocket"
A pocket is a Struct defining a futur investment pocket. A pocket has to be defined by the participants and then will be executed.
## ERC20, ERC721 and ERC1155 receiver
As the pocket management will handle buying of different kind of tokens, we have to ensure our contract can handle it.

# Usefull links <a name="usefull-links"></a>
-   [uups-proxies-tutorial-solidity-javascript](https://forum.openzeppelin.com/t/uups-proxies-tutorial-solidity-javascript/7786)
-   [Github OpenZeppelino proxy](https://github.com/OpenZeppelin/openzeppelin-contracts/tree/master/contracts/proxy)
-   [UUPS Modern walkthrough](https://r48b1t.medium.com/universal-upgrade-proxy-proxyfactory-a-modern-walkthrough-22d293e369cb)
-   [UUPS vs Transparent & Deploying more efficient proxy](https://www.youtube.com/watch?v=kWUDTZhxKZI)
-   [Old SDK Package from OZ](https://github.com/OpenZeppelin/openzeppelin-sdk/tree/master/packages/lib/contracts/upgradeability)
-   [UUPS Factory](https://forum.openzeppelin.com/t/deploying-upgradeable-proxies-and-proxy-admin-from-factory-contract/12132/12)

---

<sup>Made with ♥ by CommOwn Teams.</sup>

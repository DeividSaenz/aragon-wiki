<center>
# AragonOS

*Reflects [aragon-core](https://github.com/aragon/aragon-core) 2.0.1 implementation. Updated Dec 4th, 2017*

An [exokernel](https://en.wikipedia.org/wiki/Exokernel)-inspired architecture for modular, upgradeable and secure DAOs
</center>

## Introduction

This exokernel-inspired architecture enables modular, upgradeable and secure DAOs (Decentralized Autonomous Organizations). A UNIX-inspired permissions system, that allows safe and efficient control over software resources by utilizing smart contracts.

Smart contracts can define the rules and penalties around an agreement. They can also make decisions as well as enforce said commitments. Such as the ability to move large amounts of funds. But such significant responsibilities also make smart contracts a prime target for attacks.  Smart contracts are essential to to the Ethereum ecosystem. Testing and certifying them is not an absolute remedy because, inevitably, humans make mistakes.

So how does one guarantee DAOs that run efficiently and securely? And with upgradeability, ensuring the safety and evolution of the Ethereum ecosystem?

We present our take on our secure decentralized operating system: _AragonOS_.

This document provides a technical overview about the architecture. For a less technical oriented introduction to AragonOS, you can check the [announcement blogpost](https://blog.aragon.one/introducing-aragonos-say-hi-to-modular-and-extendable-organizations-8555af1076f3).

## 1. Kernel and the Access Control List

The Access Control List (ACL) is the filter which determines whether an address has a certain role in the context of an app. It either permits or prohibits performing actions in a given context, such as an update, as well as determining the entities with power over that permission.

The ACL permissions system is inspired by UNIX. ACL controls the permissions in the system afforded to a user and can ease the transfer of permissions. Similar to the `sudo` command in UNIX, users can escalate permissions, but with more flexibility and granularity.

### Apps
Apps are smart contracts that rely on the Kernel--the core of the operating system--for their own ACL and code upgrades. Because apps are upgradeable, we can rely on them to exist for a long period of time. This means it is safe for an app to be responsible for owning different assets on behalf of the DAO (e.g. tokens and ENS names). Apps can implement multiple actions and these actions can be protected or not by the ACL, in case one action is protected, execution of the action will only be allowed when permitted by the ACL.

### Roles

Another feature in ensuring security within the ecosystem is the ease of delegating responsibilities for autonomous execution. Apps can define any number of roles and roles can group one or more actions behind them. In order to perform an action protected by the ACL, the entity wishing to perform it must have the permission to execute that role’s actions.

Roles are always unique to each app. Being able to perform a given role in one app does not grant the permission to perform the role in other apps. An entity can have many roles in one app.

Example:

```
contract TokenApp is App {
	bytes32 constant public MINT_ROLE = 0x1234;
	function mint(address _receiver, uint256 _amount) auth(MINT_ROLE) { … }
}
```

### Entity

An Entity is any actor that is represented by an Ethereum address, such as a multisig (an account that needs multiple signatures before executing an action), an app (for example a voting app that will execute an action if token holders vote favorably) or a simple private key controlled account.

The system can delegate permissions to groups of entities by implementing a Group app. As other apps, it can rely on the ACL for protecting important functions such as adding or removing members of the group. When group members want to execute a specific action, the Group app acts as a proxy contract that performs the action on behalf of the group.

### Permissions

A **Permission** is defined as the ability to perform actions (grouped in a role)
in a certain app instance (identified by its address).

We refer to a **Permission Instance** to whether an entity has or doesn't have a
certain **Permission**.

When a permission is created, a **Permission Manager** is set for that specific
**Permission**. The permission manager is able to grant or revoke permission
instances for that permission.

#### Create Permission

```
kernel.createPermission(address entity, address app, bytes32 role, address manager)
```

`createPermission` will fail if any permission instance for that permission was created before.

This call is mostly identical to `grantPermission`, with the exception that it allows the creation of a new permission from scratch if it doesn’t yet exist.

The `createPermission` action needs to be protected by the ACL with a role. It is an important function that could be used in malicious ways. When the Kernel is initialized, it creates the permission for an address to create new permissions.

If the ACL is checked for a permission that hasn’t been created yet, the ACL won’t allow the action to be performed by default.

#### Grant Permission

```
kernel.grantPermission(address entity, address app, bytes32 role)
```

The `entity` would then be allowed to call all actions that their `role` can perform on that particular `app`. This would be in effect until the permission manager revokes it by calling `revokePermission`.

The `grantPermission` action doesn’t need to be protected with the ACL as an entity can only make changes if it is the `manager` for a given permission.

#### Revoke Permission

```
kernel.revokePermission(address entity, address app, bytes32 role)
```

`revokePermission` can be called at any time by the `manager` of a certain permission and will remove the ability of `entity` to have `role` on `app`.

The `revokePermission` action doesn’t need to be protected by the ACL either, as an entity can only make changes if it is the `manager` for a given permission.


`createPermission`, `grantPermission` and `revokePermission` fire the same event that Aragon clients must cache and use to build a locally stored version of the ACL.

```
SetPermission(address indexed from, bytes32 indexed role, address indexed to, bool allowed)
```

#### Set Permission Manager

```
kernel.setPermissionManager(address newManager, address app, bytes32 role)
```

Only the permission manager of a permission can call this function to set a new manager.

Setting a new permission manager results in the the old permission manager losing
management power for that permission

`createPermission` executes a special case of setting permission manager for the
first time. From that point forward, the manager for that permission can only be
changed by the manager with `setPermissionManager`.

Changing permission manager fires the following event:

```
ChangePermissionManager(address indexed app, bytes32 indexed role, address indexed manager)
```

#### Example

As an example, the following shows a complete flow for user Root to create a new DAO that has the basic permissions set so a Voting app could manage funds in a Vault app:

1. Deploy the Kernel
2. Performing `kernel.initialize(rootAddress)` creates the following permission under the hood:  
`createPermission(rootAddress, kernelAddress, PERMISSIONS_CREATOR_ROLE, rootAddress)`
3. Deploy the Voting app
4. Make it so that the Voting app can call `createPermission`:  
`grantPermission(votingAppAddress, PERMISSIONS_CREATOR_ROLE, kernelAddress)` (has to be executed by `rootAddress`)
5. Deploy the Vault app, which has a signature called `transferTokens`
6. Create a new vote to create the `TRANSFER_TOKENS_ROLE` permission  
`createPermission(votingAppAddress, vaultAppAddress, TRANSFER_TOKENS_ROLE, votingAppAddress)`
7. If vote passes, the Voting app can then call `TRANSFER_TOKENS_ROLE` actions, which in this case is just `transferTokens` in the Vault
8. Votes can be created to transfer funds
9. The voting app will be able to revoke or regrant the permission as it is the
permission manager for `TRANSFER_TOKENS_ROLE` on `vaultAppAddress`

An implementation of the explained ACL can be found in [aragon-core’s Kernel](https://github.com/aragon/aragon-core/blob/dev/contracts/kernel/Kernel.sol) file.

#### Checking Permissions

Apps can decide whether to protect actions behind the ACL or not, as for some actions it makes sense to have them completely public. Protecting an action behind the ACL is done simply by adding the authentication modifier which passes the role required for performing the action as a parameter. The auth modifier will check with the Kernel whether the entity performing the call has that role or not.

#### Escalating Permissions

Consider kernel **K**, an entity **E**_0_ and an app **A**. **E** wants to perform action **A**_sig_ on app **A**. The client knows (provided as part of app metadata) that in order to perform **A**_sig_ an entity is required to have **A**_role_.
The client should know that **E**_0_ cannot directly call **A**_sig_, as it doesn’t have that role, but that a list of Entities [**E**_1_, **E**_2_] do have that role on app **A**. The client should then show the user the multiple possible forwarding paths to pass the call to **E**_1_, so then **E**_0_ could perform **A**_sig_.

Calculating a forwarding path requires knowing what Forwarders entity **E**_0_ can perform actions through.
The user or contract performing this action could then choose their preferred route to forward permissions in order to perform **A**_sig_. For example, **E**_1_ may be a Voting app **V**, so the action would be to create a new vote that, in case of being approved, would call **A**_sig_. Since **V** has role **A**_role_ it has permission to execute **A**_sig_, therefore we would have successfully completed a permission escalation.  
Note that permission escalation can occur instantly or it can be delayed and require further action by other entities like in the case of the voting app.

<center><img src="../../images/aragonos/permission_escalation.png"></center>

##### Forwarders

In order to make permission escalation easier, we can define a common interface for how this happens:

```
contract Forwarder {
   bool isForwarder = true;
   function canForward(address sender, bytes evmCallScript) view returns (bool);
   function forward(bytes evmCallScript);
}
```

##### EVM Call Script

The `evmCallScript` parameter allows for executing multiple calls with just one transaction. This is useful, for example, in the case of Votings, as it allows approving multiple actions in just one vote.

An `evmCallScript` is the concatenation of multiple `evmCallActions`. A `evmCallActions` payload is:

``[ to (address: 20 bytes) ] [ calldataLength (uint32: 4 bytes)  ] [ calldata (calldataLength bytes) ]``

When executed, actions in the `evmCallScript` are executed one at a time using _CALL_. If just one of the actions fails (call returns 0), the entire execution is reverted.

An implementation for EVM Call Script can be found here: [`EVMCallScript.sol`](https://github.com/aragon/aragon-core/blob/dev/contracts/common/EVMCallScript.sol).

If a user wants to enact an action it cannot perform directly, it can check if there are forwarders. This checks if there are entities which can perform the action that any of the user addresses can use.

This can be multiple levels deep. For example, if a user address holds a number of tokens, the user is allowed to use the token manager entity. This is something that forwards when the sender owns tokens. The token manager entity is then the only one allowed to create a new vote.

## 2. Upgradeability

Upgradeability of the system is achieved by using the [DelegateProxy](https://github.com/aragon/aragon-core/blob/dev/contracts/common/DelegateProxy.sol) pattern. Kernel and apps (`KernelProxy` and `AppProxy`) both use DelegateProxy making some slight modifications.

<center><img src="../../images/aragonos/delegateproxy.png"></center>

Given that new versions of apps or the kernel will run in the exact same context as previous versions, the old storage layout must be taken into account. Inheriting from the old contract storage before adding new storage variables is considered a safe practice. It is recommended to make sure the upgrade doesn't break the storage before pushing a new version. We will work on tooling to prevent issues with storage on upgrades.

### 2.1. Kernel upgradeability

For the [Kernel](https://github.com/aragon/aragon-core/blob/dev/contracts/kernel/Kernel.sol) to be easily upgradeable, more efficient and cheaper to deploy, we will use a proxy type construct. Deploying a new DAO is done by deploying a [KernelProxy](https://github.com/aragon/aragon-core/blob/dev/contracts/kernel/KernelProxy.sol) contract that just delegates all calls to a kernel implementation at a given address, while still maintaining its own storage. Upgrading the kernel is as easy as changing the reference to another implementation. Even though it is fairly easy to do, this action is really critical and should be protected accordingly.

### 2.2. App-space upgradeability

As apps can be used as entities (e.g. a voting app) it is important that the apps can keep their addresses fixed for maintaining their identities even if their underlying logic changes. This is in order to keep the permissions not having to change at the kernel level with the upgrades.

An idea of how this can be achieved is by using the concept of [AppProxy](https://github.com/aragon/aragon-core/blob/dev/contracts/apps/AppProxy.sol) contracts (inspired by [augur-core’s Delegators](https://github.com/AugurProject/augur-core/blob/develop/source/contracts/libraries/Delegator.sol)). Deploying an app with an AppProxy contract just needs a reference to the Kernel and the app identifier. When the app receives a call, it gets intercepted by the fallback function. At this point the proxy contract will ask the Kernel for the address of the latest version of the app code for a given app identifier. The [AppProxy](https://github.com/aragon/aragon-core/blob/dev/contracts/apps/AppProxy.sol) contract will then `delegatecall` into this address, making sure that the latest version of the app is always running.

<center><img src="../../images/aragonos/appproxy_delegatecall.png"></center>

Given that the Kernel will have a centralized record of the latest version of the code for the app identifiers, changing one reference in the Kernel will effectively update all instances of the app relying on the Kernel. These upgrades could then be delegated to another contract, Aragon Network, in case the users don’t wish to handle manual upgrades of the apps.

```
kernel.setAppCode(bytes32 appId, address appCode)
```

This action updates the implementation code registered for a given `appId`. That will effectively upgrade all apps that depend on the Kernel. Making it so that all future calls will use the new app code.
In case the `appId` for an `appCode` hasn’t been set, all calls to the app will fail.

## 3. Initialization

The contract that gets deployed doesn’t contain the business logic for the component. This is due to the Kernel and apps relying on a Proxy-like architecture for upgradeability. It is merely a way to point to the given logic when called.

This impedes the ability to call the Solidity constructor. Constructors are only ran when creating the contract, but not stored as part of the account code.

Initialization is performed with a 'regular function' that should be called after the `AppProxy` contract has been deployed. To impede the attack in which a malicious actor tries to get an initialization transaction between the deployment and the legit initialization transaction, `AppProxy` allows to pass an initialization payload that will be executed on deployment. This allows to have atomic initialization of apps.

It is important that this initialization function can only be called once. Instead of just saving a boolean when a component has been initialized, we store the block number when the initialization happened. This can then be used by client to know from which block they need to filter for events.

## 4. App discoverability and package versioning

An app **A** is nothing more than a simple AppProxy contract with a reference to a kernel **K** and an `appId`. If **A** doesn’t have any permissions, both incoming or outgoing, set in **K**, it effectively means that it’s irrelevant to the DAO, since nothing can call it and it cannot call anything. So the notion of installing an app is replaced with the notion of creating permissions for an app (what needs to happen for different functions of the app to be called and what can the app do in regard to other apps).

All relevant apps in a DAO could be obtained by traversing its ACL, checking if an address there is an AppProxy, and checking if the AppProxy has a reference to the DAO’s kernel. If that happens, an app with the AppProxy’s `appId` is effectively installed in the DAO.

Application identifiers (`appId`) can be used as a the identifier for the full package (frontend for the app and other artifacts).
The `appId` of an app should be the ENS name for the package. The `namehash` function recursively hashes different components of a name ([reference implementation](https://www.npmjs.com/package/eth-ens-namehash))

```
appId = namehash(“voting.aragonpm.eth”)
```

Aragon owns _aragonpm.eth_ and will allow developers to create packages, but the system is designed to be domain agnostic (for example, district0x could decide to have all their packages under `packages.district.eth`)

After discovering an entity in the DAO that is an app, we can fetch its appId and because the appId is a ENS name, we can lookup in the ENS to find the full repo contract for an appId.

```
repo = Resolver(ens.resolver(appId)).addr(appId)
```

Repo contracts conform to the original Aragon Versioning Protocol but add more on-chain checks for what a valid version update is.  

The Repo has an owner which is an Ethereum address that can create new versions of the package. This ownership right can be transferred to another address by the owner.  

A Repo allows to keep versioned state over:  

  - Smart contract app code (`contractAddress`): the app code is the address of the deployed version of the app. The kernel can then point to this address and all instances of the app, depending on the kernel it will use.
  - Package content (`contentURI`): defined by a location id of where the package is hosted (IPFS, Swarm, etc.) and a content hash to fetch it.

By having both the app code reference and the package content, we can assert some guarantees of what the semantic versioning means.

  - **Patch version**: Minor changes to the package contents (frontend). Update can be performed silently for users.
  - **Minor version**: Major changes to the package contents, but still works with the current  smart contract code (frontend). Users should be notified of the update.
  - **Major version**: Changes to both the smart contracts and the frontend. In order to use this version of the frontend, an upgrade to new smart contract(s) is required. User interaction is needed to upgrade.

  By having this check performed at the smart contract level, we can load the correct version of the frontend just by looking at an instance of an app. This is done by checking that the version of a smart contract is linked to a given app by getting the appId and the appCode.

  A correct version raise for a package is defined by the following rules:

  - It can only increase one member of the version by 1. The version components at the left of the raised member must stay the same and the components at the right must be 0.
    - Example: From 2.1.3 the only allowed rises are to 3.0.0 (major version), 2.2.0 (minor version) and 2.1.4 (patch version).
  - Changes to the app code address can only be done if the raise changes the major version (turning it into x.0.0 by the above rule).


The package that is stored off-chain needs to contain a standard [manifest.json](https://w3c.github.io/manifest/) file.
Apart from that, making the interaction with the smart contracts possible, we introduce a specific `eth.json` file with the following keys:

  - `abi`: Standard [Ethereum Contract ABI](https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI)
  - `bytecode`: Init code for deploying the app contract to the network.
  - `functions`: An array of all the relevant function signatures for the contract, with its [natspec](https://github.com/ethereum/wiki/wiki/Ethereum-Natural-Specification-Format) description, argument names and whether the function is protected by the ACL or not. These can be automatically generated on package publish. What role is needed to call a function is specified in the function object.
  - `permissions`: An array of what permissions the app needs to have over other entities in order to work. This will probably be dependent on initialization parameters. For example: a Finance app will need permissions over the Vault it is initialized with.
  - `verification`: An object providing the needed information to independently verify the source code of the deployed code. Values needed are: `deployTxId`, `sourceCode` and the compiler settings, solc version and optimization settings.

An initial implementation of the contracts supporting package management can be found on the apm-contracts Github [repo](https://github.com/aragon/apm-contracts).

## Leaks

1. Replay attack

A replay attack on a smart contract is a type of security vulnerability that allows an attacker to reuse a validly signed transaction multiple times. We saw on the previous chapter how to do offchain replay attacks, but it is possible to do also onchain replay attacks.
Besides, Tezos prevents this kind of vulnerabilities, it is possible to write a code that does this attack sending the same operation several times for execution.

Compile and simulate the replay attack :

```bash
taq compile 1-replay.jsligo
taq simulate 1-replay.tz --param=1-replay.parameter.parameter.tz
```

The simulation will tell you that several internal transaction will be executed.
But if you deploy the code and try to really execute it :

```bash
taq deploy 1-replay.tz --mutez 1000 -e testing

taq transfer KT1VMt7t4CboRP6jYBUdBQowHb4NR1UtmDrz -e testing
```

Then, the Tezos will detect the flaw

```logs
  "message": "(transaction) proto.017-PtNairob.internal_operation_replay"
```

2. Memory overflow

Memory overflow is a kind of attack that fulfills the memory of a smart contract resulting on making this contract unusable. Even simply loading the data into memory and deserializing it at the beginning of the call, could use so much gas that any call to the contract would fail. All the funds would be forever locked into the contract.

Here are the list of dangerous types to use carefully :

- integers and nats : as they can be increased to an arbitrary large value
- strings : as there is no limit on their lengths
- lists, sets, maps : that can contain an arbitrary number of items

&rarr; **SOLUTION** :

- ask the user to pay a minimum tez for each call
- set a threshold limit
- store data on a big_map
- avoid unnecessary onchain computation that can be done off chain. Ex : do not loop onchain and just update a part of a map

Example with the current on FA1.2 implementation : https://inference.ag/blog/2023-10-09-FA12_spenders/

You can have a look on the Ligo implementation of fa1.2 on the Ligo registry [here](https://packages.ligolang.org/package/ligo-fa1.2)

The code respect the Standard but you can see that the [Allowance type is actually a map](https://github.com/frankhillard/ligoFA12/blob/main/lib/asset/allowance.mligo#L3C8-L3C8). It would have been better to change the Standard and use a `big_map` instead a `map`. If you implement the Standard differently, then your smart contract storage definition and entrypoint signatures will not match anymore and will not be supported by other platforms

3. Re-entrancy

These attacks allow an attacker to repeatedly call a contract function in a way that drains the contract’s resources, leading to a denial of service (DoS) attack

One of the most well-known examples of a re-entrancy attack occurred in 2016, when an attacker exploited a vulnerability in the DAO (Decentralized Autonomous Organization) contract on the Ethereum blockchain. But this popular hack is still actively used :

- Uniswap/Lendf.Me hacks (April 2020) – $25 mln, attacked by a hacker using a re-entrancy.
- The BurgerSwap hack (May 2021) – $7.2 mln, because of a fake token contract and a re-entrancy exploit.
- The SURGEBNB hack (August 2021) – $4 mln, seems to be a re-entrancy-based price manipulation attack.
- CREAM FINANCE hack (August 2021) – $18.8 mln, re-entrancy vulnerability allowed the exploiter for the second borrow.
- Siren protocol hack (September 2021) – $3.5 mln, AMM pools were exploited through re-entrancy attack.

This kind of attack is quite simple to put in place with Solidity the way it works.

Consider this scenario :

```mermaid
sequenceDiagram
  User->>MaliciousContract: deposit funds
  MaliciousContract->>LedgerContract: deposit funds
  User->>MaliciousContract: call withdraw
  MaliciousContract->>LedgerContract: call withdraw
  Note right of LedgerContract: checkBalance
  Note right of LedgerContract: sendFunds
  LedgerContract->>MaliciousContract: sendFunds operation
  Note right of MaliciousContract: loop calling withdraw ... x times
  MaliciousContract->>LedgerContract: call withdraw
  LedgerContract->>MaliciousContract: sendFunds operation  ... x times
  Note right of LedgerContract: ... Once finish ... UpdateBalance
```

Why this scenario is not possible on Solidity ?
On Solidity, the operation will call directly the smart contract like doing a stop, call a synchronous execution and continue the flow.

Why this scenario is not possible on Tezos ?
On Tezos, the first transaction will update the state and will execute a list of operation at the end of execution. Next executions will encounter an updated state

Let's implement a more complex scenario, now :

```mermaid
sequenceDiagram
  User->>MaliciousContract: deposit cookies
  MaliciousContract->>LedgerContract: deposit cookies
  User->>MaliciousContract: sell cookies
  MaliciousContract->>OfferContract: sell cookies
  Note right of OfferContract: checkBalance
  OfferContract->>LedgerContract: call hasCookies view
  Note right of OfferContract: prepare post operations (sendFund + changeOwner)
  OfferContract->>MaliciousContract: sendFund
  Note right of MaliciousContract: while receiving fund on default entrypoint will loop on selling cookies
  MaliciousContract->>OfferContract: sell cookies
  Note right of OfferContract: checkBalance
  OfferContract->>LedgerContract: call hasCookies view
  Note right of OfferContract: prepare post operations (sendFund + changeOwner)
  OfferContract->>MaliciousContract: sendFund
  MaliciousContract->>OfferContract: sell cookies
  Note right of OfferContract: checkBalance
  OfferContract->>LedgerContract: call hasCookies view
  Note right of OfferContract: prepare post operations (sendFund + changeOwner)
  OfferContract->>MaliciousContract: sendFund
  OfferContract->>LedgerContract: call changeOwner
```

The issue here is clearly that we send money without updating the state first

&rarr; **SOLUTION** :

- Mutex safeguard : The goal is to avoid that multiple internal operations are generated. A boolean `isRunning` will lock only one operation for the full transaction flow.

  1. Check the isRunning is false
  2. Set isRunning to true
  3. Do logic code ...
  4. Create a last operation transaction to reset the boolean to false

- Check-and-send pattern : Principle of separating state changes from external contract interactions. First, update the contract’s state, then interact with other contracts

Compile/Run the hack test first

```bash
taq test 3-reentrancyTest.jsligo
```

The logs seems to be fine, but it is hard to guess the internal transactions and to separate the fees from the hack on the attacker balance

```logs
┌─────────────────────────┬─────────────────────────────────────────────┐
│ Contract                │ Test Results                                │
├─────────────────────────┼─────────────────────────────────────────────┤
│ 3-reentrancyTest.jsligo │ "ledgerContract"                            │
│                         │ KT1LQyTHEZeaecRj7hWgkzPEBD6vMEKXYzoo(None)  │
│                         │ "offerContract"                             │
│                         │ KT1M4nPCej4va4Q2iMPX2FKt8xLw5cfGjBv9(None)  │
│                         │ "maliciousContract"                         │
│                         │ KT1B7RgF6j7UpAybpdfxhLCp7hf41pNFcxyS(None)  │
│                         │ "admin initialize cookies to malicious KT1" │
│                         │ Success (1299n)                             │
│                         │ "COOKIES OWNERS"                            │
│                         │ {KT1B7RgF6j7UpAybpdfxhLCp7hf41pNFcxyS}      │
│                         │ "BALANCE OF SENDER"                         │
│                         │ 3799985579750mutez                          │
│                         │ Success (1798n)                             │
│                         │ "AFTER RUN - BALANCE OF SENDER"             │
│                         │ 3799984579749mutez                          │
│                         │ {KT1LQyTHEZeaecRj7hWgkzPEBD6vMEKXYzoo}      │
│                         │ "END RUN - BALANCE OF SENDER"               │
│                         │ 3799984579749mutez                          │
│                         │ {KT1LQyTHEZeaecRj7hWgkzPEBD6vMEKXYzoo}      │
│                         │ Everything at the top-level was executed.   │
│                         │ - testReentrancy exited with value true.    │
│                         │                                             │
│                         │ 🎉 All tests passed 🎉                      │
└─────────────────────────┴─────────────────────────────────────────────┘
```

To have a better visualization of the hack, the contract should be deployed

Compile the first contract, the Ledger contract, and deploy it

```bash
taq compile 3-reentrancyLedgerContract.jsligo
taq deploy 3-reentrancyLedgerContract.tz -e testing
```

Copy the contract address, in my case KT1BJZfhC459WqCVJzPmu3vJSWFFkvyi9k1u, and paste it on the file `3-reentrancyOfferContract.storageList.jsligo` with your value

Compile/deploy the second contract, the Offer contract, putting some money on the contract for the thieves

```bash
taq compile 3-reentrancyOfferContract.jsligo
taq deploy 3-reentrancyOfferContract.tz -e testing --mutez 10000000
```

Copy the contract address, in my case KT1CHJgXEdBPktNNPGTDaL8XEAzJV9fjSkrZ, and paste it on the file `3-reentrancyMaliciousContract.storageList.jsligo` with your value

Compile/deploy the last contract, the Malicious contract who will loop and steal the funds of the Offer contract

```bash
taq compile 3-reentrancyMaliciousContract.jsligo
taq deploy 3-reentrancyMaliciousContract.tz -e testing
```

Copy the contract address, in my case KT1NKLZE9HkGJxjopowLqxA4pswutgMrrXyE, and initialize the Ledger contract as the Malicious contract as some cookies. Paste the value in the file `3-reentrancyLedgerContract.parameterList.jsligo`

Once done, compile the Ledger contracts and call with this parameter

```bash
taq compile 3-reentrancyLedgerContract.jsligo
taq call 3-reentrancyLedgerContract --param 3-reentrancyLedgerContract.parameter.default_parameter.tz -e testing
```

Context is ready :

- the Malicious contract has cookies on the Ledger contract
- all deployed contract points to the correct addresses

Now the Malicious contract will try to steal funds from the Offer contract, run the command to start the attack the transaction flow

```bash
octez-client transfer 0 from alice to KT1NKLZE9HkGJxjopowLqxA4pswutgMrrXyE --entrypoint attack --arg 'Unit' --burn-cap 1
```

Here you can see the result on the Ghostnet : https://ghostnet.tzkt.io/KT1NKLZE9HkGJxjopowLqxA4pswutgMrrXyE/operations/

3 refunds will be emitted instead of one

&rarr; **SOLUTION** : on the `3-reentrancyOfferContract.jsligo` file, line 34, swap the order of operation execution

from

```
return [list([opTx, opChangeOwner]), s];
```

to

```
return [list([opChangeOwner,opTx]), s];
```

and rerun the scenario from scratch redeploying the contracts. IT should be impossible to run the attack, as the transaction will fail

```logs
"message":"user do not have cookies"
```

- Authorize withdraw transfer only to user account : As User wallet cannot do callback loops, it solves the issue but this solution is not always feasible and limiting. To check if an address is implicit, the Tezos.get_sender and the Tezos.get_source are always equal.

- Audit External Contract calls : This is very hard to check, for example on withdraw for a token transfer, any contract can receive funds.

- Call third-party security experts or employ automated security tools : If you are not sure about your code, they will identify weaknesses and validate the contract’s security measures.

4. Overflow

Manipulating arithmetic operation can lead to overflows and underflows

- On Solidity : SafeMath is a library in Solidity that was designed to provide safe mathematical operations. It prevents overflow and underflow errors when working with unsigned integers (uint), which can lead to unexpected behavior in smart contracts. However since Solidity v0.8.0, this library has been made obsolete as the language itself starts to include internal checking.

- On Ligo : For the nat, int, and timestamp types, the Michelson interpreter uses arbitrary-precision arithmetic provided by the [OCaml Zarith library](https://github.com/ocaml/Zarith). It means that their size is only limited by gas or storage limits. You can store huge numbers in a contract without reaching the limit. However, in LIGO, an overflow will cause the contract to fail.

&rarr; **SOLUTION** :

- For large Tez values, do operation on int or nat as it has larger memory values
- There is no other solution than using types with larger values as the default behavior is to reject the transaction in case of overflow

> Do not confuse with [Ligo MathLib library](https://packages.ligolang.org/package/@ligo/math-lib) providing manipulation of floats and rationals instead of using basic types.

## Leaks

- replay attack : return same op 2 times on the result list !!! //TODO code

- memory (overflow (ex on FA1.2 : https://inference.ag/blog/2023-10-09-FA12_spenders/))

  - integers and nats, as they can be increased to an arbitrary large value
  - strings, as there is no limit on their lengths
  - lists, sets, maps, that can contain an arbitrary number of items

  Ex : Anyone could attack this contract by calling the lockAmount entry point with 0 tez many times, to add so many entries into the lockedAmount list, that simply looping through the entries would consume too much gas.
  From then on, all the funds would be forever locked into the contract.
  Even simply loading the list into memory and deserializing the data at the beginning of the call, could use so much gas that any call to the contract would fail.

  => solution :

  - ask the user to pay a minimum tez for each call
  - set a threshold limit
  - store data on a big_map
  - avoid unnecessary onchain coputation that can be do off chain. Ex : do not loop onchain and just update the part to purge a map in cache on an indexer ?

### Reentrancy

It is hard but still possible on Tezos

the ETH loop of callback does not work but ... . Ex : no idea if possible to send money and loop on withdraw because balance is reset to
0 ???). Implement the test and see that it is not possible on a Test ! //TODO demonstarte it + SHOW this example (https://opentezos.com/smart-contracts/avoiding-flaws/#10-re-entrancy-flaws)

- overflow ??? There is no SafeMath in ligo . Do not confuse with https://packages.ligolang.org/package/@ligo/math-lib that is to manipulate float instead or multiply/deivide by 10^6. For the nat, int, and timestamp types, the Michelson interpreter uses arbitrary-precision arithmetic provided by the OCaml Zarith library. It means that their size is only limited by gas or storage limits. You can store huge numbers in a contract without reaching the limit

=> solution : do operation on int or nat instead of tez as it has larger values

- frontRunning / MEV : It can be done by the baker itself as the list is known in advance at each period ... or any bots litening to the gossip network ...
  - buy before big BUY TX , sell token after === sandwich attack
  - BOT : whataver increase user balance, you just copy with higher fees to pass first
  - BAKER : just change the order

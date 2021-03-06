# Launch scope

## Goals
* Add an RPC to metamask (with wallet_addEthereumChain hopefully, or manually)
* Switch the compiler from solc -> solcovm
* Optimism L2 should behave like L1
  * Value works in metamask like normal
  * Contracts work compiled with ovm
  * Uniswap and Synthetix should work
* Native currency on both is ETH, and fees are paid similarly
  * Put more weight on size of calldata? (like TxDataNonZeroGasEIP2028)
* Fast deposit by sending ETH to a special address on L1, shows up in L2
  * Get the ENS name l2deposit.eth or something. This can also be triggered from a web ui, but it's just a send on metamask.
* Slow withdrawal by sending ETH to a special address on L2, shows up back in L1 7 days later
  * More thought should be given to this, as it's a "burn" on L2, right?
* Generic trusted messaging bridge people can use as a primitive (TODO: add link here to API)
* Avoid regenesis and keep continuity with existing L2 mainnet chain

## Non Goals
* For now, a fully controlled Proof of Authority chain is okay
* Single trusted sequencer, no fraud proofs
* Contracts must be compiled with OVM, and hence must be smaller since the OVM adds some overhead, and deployment uses a lot of gas (due to safety checking?)
* Value does not work in contracts (contracts can use the ERC20 backend @ 0x42....06)

# Implementation

## Flow of a tx
* tx is submitted into [l2geth](https://github.com/ethereum-optimism/go-ethereum) (TODO: rename this repo to l2geth?)
* It is processed mostly by the [contracts](https://github.com/ethereum-optimism/contracts) that define the OVM operating system (TODO: rename this ovm-os-contracts or something)
* The single [batch submitter](https://github.com/ethereum-optimism/batch-submitter) submits both the transaction and the new state root
* The [data transport layer](https://github.com/ethereum-optimism/data-transport-layer) (TODO: rename this repo to l1-tx-indexer or something) reads the submitted transaction and state root in L1.

## Number of lines
* Compiler -- 300?
* l2geth -- 1000?
  * This should fetch the addresses from address manager through data-transport-layer
  * This should do very little processing, and leave that to the Operating System Contracts
* data-transport-layer -- 2243 (this should merge into l2geth)
* batch-submitter -- 1243
* optimism-ts-services -- 2312 (this should merge with batch-submitter)
  * Message passer
  * Fraud prover submission
* Contracts -- 8414!
  * 4406 in the OVM
    * accounts -- 137
    * bridge -- 587
    * chain -- 1165
    * execution -- 1577
    * precompiles -- 334
    * verification -- 606
  * See [list of L1 contracts](https://github.com/cheapETH/cheapoptimism/blob/master/addresses.json)
  * See [list of L2 contracts](https://github.com/cheapETH/cheapoptimism/blob/master/state-dump.latest.json)
  * Operating System Contracts (L2 contracts)
  * Fraud prover contracts (L1 contracts)
  * Communication contracts
  * NOTE: I would separate this contracts into "groups" more than the current dir structure. Which form clusters and which can be run independently?
  * NOTE: This is the most important part to have a spec for. If you want to get contracts audited, the auditer must understand the intended functionality before understanding what a deviation looks like.
  * NOTE: The contracts here that keep L2 chain state should be the slowest to change, if ever.
  * Other thoughts:
    * Remove all interfaces for contracts that shouldn't be accessed from the outside world
    * Why is the AddressManager in lib? It's a contract that's deployed, right?
    * What should OVMCodec be called? What's a codec?
    * I don't think the "wrappers" should be in lib
    * All this "compression" stuff should go, seems like premature optimization

## Upgradablity
* Contracts on L1 can be upgraded?
  * Be careful with where the submitted state actually lives, that's the thing to not lose
  * OVM_CanonicalTransactionChain and OVM_StateCommitmentChain are accessors?
  * I also see three OVM_ChainStorageContainer, this is the actual state? Is it all of it?
    * Is there an OVM state explorer which can query and parse these container contracts?
* L2 OS contracts come from a special genesis contract on L1 (remove state-dump.latest.json)
* The L2 contracts can be upgraded using transactions from L1, and this keeps them in time order. This will be key to preventing regenesis.


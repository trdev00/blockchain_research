# notes from _The Bitcoin Lightning Network: Scalable Off-Chain Instant Payments_

- by Joseph Poon and Thaddeus Dryja
- January 2016

## The Bitcoin Blockchain Scalability Problem

- the Bitcoin blockchain excells as a distributed ledger
- as a large scale payment platform it suffers from scaling issues
- the blockchain is a gossip protocol
  - all state modifications to the ledger are broadcasted to all participants
  - consensus of the state is agreed upon through this protocol
  - each node in the Bitcoin network must know about every single transaction
    that occurs in the network
- for Bitcoin to scale to the amount of transactions needed to use it for daily
  transactions for the majority of the population of the world it would require
  huge amounts of data and is not feasible with current technology
- increasing block sizes to handle more transactions per second could result
  in only miners who could handle such large data loads to be able to validate
  the blocks
  - this can result in centralization of the network validators
  - can result in higher fees
- to avoid centralization Bitcoin needs to be able to be validated by a single
  consumer level computer on a home broadband connection
  - full validation must be able to occur cheaply - ensures low transaction fees
- currently in order to greatly increase the amount of transactions per second
  with Bitcoin - transactions must be made off the Bitcoin blockchain
  - on a sidechain

## A Network of Micropayment Channels Can Solve Scalability

- in a blockchain - if only 2 participants care about an everyday recurring
  transaction - it is not necessary for all other nodes in the network to
  kow about that transaction
  - preferable to only have the bare minimum information on the blockchain
  - a net settlement of the relationship between these 2 participants can be
    completed at a later date and added to the blockchain at that time]
    - this allows for many transactions to be completed without bloating the
      main blockchain
    - also removes the need for a trusted centralized counterparty
- a trustless structure can be achieved by using time locks as a component
  to global consensus
- using a network of micropayment channels allows for Bitcoin to scale to
  billions of transactions per day using readily available modern compute power
- many payments can be sent in a single micropayment channel
  - enables one to send large amounts of funds to another party in a
    decentralized manner
- micropayment channels create a relationship between 2 parties to perpetually
  update balances
  - the broadcast to the blockchain is deferred and sent in a single transaction
    which nets out the total balance between the 2 parties
  - allows the financial relationships between 2 parties to be trustlessly
    deferred to a later date without risk of counterparty default
- micropayment channels used real Bitcoin transactions
  - broadcasts to the blockchain are deferred so that both parties can gaurantee
    their current balance on the blockchain
    - real Bitcoin communicated and exchanged off-chain

### Micropayment Channels Do Not Require Trust

- cryptographic signatures allow a blockchain to prove who owns what
- a blockchain ledger is used as a timestampting system
- it is desireable to create a system which does not actively use this
  timestamping system unless absolutely necessary
  - it is costly to the network to use
  - both parties could commit to signing a transaction and not broadcasting the
    transaction
- with micropayment channels only 2 states are required
  1. current correct balance
  2. any old deprecated balances
- there is onle a single correct current balance
  - can be many old balances which are deprecated
- it is possible in Bitcoin to devise a Bitcoin script where all old
  transactions are invalidated
  - only the new transaction is valid
- invalidation is enforced by a Bitcoin output script and dependent transactions
  which force the other party to give all their funds to the channel
  counterparty
  - by taking all funds as a penalty to give to the other - all old transactions
    are thereby invalidated
  - this invalidation process can exist through a process of channel consensus
    where if both parties agree on current ledger states
    - as well as building new states
    - then the real balance gets updated
  - the balance is reflected on the blockchain only when a single party
    disagrees
  - this system is not an independent overlay network
  - this system is a deferral of state on the current system
    - enforcement is still occurring on the blockchain itself
      - deferred to future dates and transactions

### A Network of Channels

- micropayment channels only create a relationship between 2 parties
  - everyone is required to create channels with everyone else
  - this does not solve the scalability problem
- Bitcoin scalability can be achieved using a large network of micropayment
  channels
- it is possible to create a near-infinite amount of transactions in the network
  - in a large network of channels on the Bitcoin blockchain where all users are
    participating on this graph by having at least 1 channel open on the
    Bitcoin blockchain
  - only transactions that are broadcasted on the Bitcoin blockchain prematurely
    are with uncooperative channel counterparties
- the Bitcoin transaction outputs have a hashlock and a timelock
  - the channel counterparty is unable to steal funds
  - Bitcoins can be exchanged without counterparty theft
  - by using staggered timeouts - it is possible to send funds via multiple
    intermediaries in a network without risk of intermediary theft of funds

## Bidirectional Payment Channels

- micropayment channels permit a simple deferral of a transaction state to be
  broadcast at a later time
- contracts are enforced by creating a responsibility for 1 party to
  broadcast transactions before or after certain dates
- in a blockchain with a decentralized timestamping system:
  - possible to used clocks as a component of decentralized consensus to
    determine data validity and present states as a method to order events
- timeframes are created in which certain states can be broadcasted and then
  later invalidated
  - possible to create complex contracts using Bitcoin transaction scripts
  - the Lightning Network's bidirectional microparment channel requires
    the malleability soft-fork to enable near-infinite scalability while
    mitigating risks of intermediate node default
- multiple micropayment channels can be chained together allowing the creation
  of a network of transaction paths
  - paths can be routed using BGP-like systems
  - sender can designate a particular path to the recipient
  - output scripts are encumbered by a hash
    - the hash is generated by the recipient
  - by disclosing the input to the hash - the recipient's counterparty is able
    to pull funds along the route

### The Problem of Blame in Channel Creation

- in order to participate in this payment network - one must create a
  micropayment channel with another participant on this network

#### Creating an Unsigned Funding Transaction

- initial channel Funding Transaction is created
  - 1 or both channel counterparties fund the inputs of the transaction
- both parties create the inputs and outputs for the transaction
  - both do not sign the transaction
- the output for the Funding Transaction is a single 2-of-2 multisignature
  script
- both participants do not exchange signatures for the Funding Transaction
  until they have created spends from this 2-of-2 output refunding the original
  amount back to its respective funders
  - transactions are not signed to allow for one to spend from a transaction
    which does not yet exist
- if 2 parties exchange the signatures from the Funding Transaction without
  being able to broadcast spends from the Funding transaction
  - the funds may be locked up forever if either party does not cooperate
    - or is other coin loss occurs through hostage scenarios
      - where one pays for the cooperation from the counterparty
  - the 2 parties exchange inputs to fund the Funding Transaction and exchange
    1 key to use to sign with at a later point
    - this allows for knowledge of which inputs are used to determine the total
      value of the channel
    - the key is used for the 2-of-2 output for the Funding Transaction
      - both parties must agree to spend from the Funding Transaction

#### SPending from an Unsigned Transaction

- Lightning Network uses a ```SIGHASH_NOINPUT``` transaction to spend from the
  2-OF-2 Funding Transaction output
  - necessary to spend from a tranaction for which the signatures are not yet
    exchanged
  - ```SIGHASH_NOINPUT``` is implemented into Bitcoin by a soft-fork
    - it ensures transactions can be spent from before being signed by all
      parties
  - without ```SIGHASH_NOINPUT``` it is not possible to generate a spend from
    a transaction without exchanging signatures
    - spending the Funding Transaction (parent) requires a transaction ID as
      part of the signature in the child's input
    - the parent's signature is a component of the transaction ID
    - both parties need to exchange their signatures of the parent transaction
      before the child can be spent
    - one or both parties are able to broadcast the parent before the child
      exists
    - ```SIGHASH_NOINPUT``` is used to get around this restriction by permitting
      the child to spend without signing the input
- ```SIGHASH_NOINPUT``` order of operations:
  1. create the parent Funding Transaction
  2. create the children Commitment Transactions and all spends from the
     Commitment Transactions
  3. sign the children
  4. exchange the signatures for the children
  5. sign the parent
  6. exchange the signatures for the parent
     - if 1 party fails on this step - the parent can either be spent and become
       the parent transaction or the inputs to the parent transaction can be
       double-spent - this entire transaction path is invalidated
  7. broadcast the parent on the blockchain
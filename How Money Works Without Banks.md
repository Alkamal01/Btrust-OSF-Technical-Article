# How Money Works Without Banks

## Understanding Bitcoin's Approach to Digital Payments

If you've heard about Bitcoin but aren't sure how it works, this article explains the basic ideas behind the protocol. It starts with the problem Bitcoin was designed to address, then looks at the main parts of the system and how they work together.

*AI Usage Disclosure: I wrote and researched this article myself. I used AI only to check grammar and language, not for research, structure, or content.*

## The Problem That's Older Than Computers

Imagine you want to buy a laptop from someone online. You have never met them, and they have never met you. Both sides have a reason to be cautious.

**Your concern:** If you pay first, the seller could take your money and never send the laptop.

**Their concern:** If they send the laptop first, you could receive it and never pay.

This is known as the **fair exchange problem**. People have dealt with it for thousands of years.

For a long time, the solution has been to use a trusted third party:

* **Banks:** Hold and transfer money between the parties.
* **PayPal:** Provides a digital payment service between buyers and sellers.
* **Escrow services:** Hold funds until the agreed conditions are met.

These services are useful, but they also introduce risks and costs:

* Transactions can sometimes be reversed.
* Fees are charged for the service.
* A security breach can put funds at risk.
* A company can fail or become unavailable.
* A service provider can refuse to serve a customer.

Satoshi Nakamoto's question was whether people could exchange money directly without relying on a central intermediary.

Bitcoin's answer was to use cryptography, a public transaction history, and proof-of-work to let participants verify transactions without a central authority.

## Key Terms You Need to Know

**Ledger:** A record of transactions that shows the movement of money.

**Decentralized:** No single person or company controls the Bitcoin network. Many computers, called nodes, maintain and verify the transaction history.

**Consensus:** The process by which participants in the network agree on which transactions and blocks are valid.

**Proof-of-Work:** A system that requires miners to perform computational work before they can add a new block to the blockchain.

**Transaction:** A record that transfers control of Bitcoin from one set of keys to another.

## The Breakthrough: Make Cheating Expensive

Imagine a town where people do not trust one another. They need a way to keep a shared record without relying on one person to maintain it.

Bitcoin uses proof-of-work as part of that solution.

When miners compete to add a new block, they must perform computational work. The winning miner publishes the block, and other participants verify it.

If someone wants to change an old transaction, changing that record alone is not enough. They would also need to redo the proof-of-work for that block and the blocks that follow it, while keeping up with the honest network.

That makes rewriting the transaction history expensive.

This is the basic role of **proof-of-work** in Bitcoin. It makes attacks on the transaction history require substantial computing power and energy.

## How Bitcoin Works

When I started learning Bitcoin, I initially saw these concepts as separate pieces. They are easier to understand when viewed as parts of the same system.

A central question is how the network prevents someone from spending the same Bitcoin twice.

### 1. Confirmed Transactions Become Difficult to Reverse

When you send Bitcoin and the transaction is confirmed, there is no central authority that can simply reverse it.

**Why does this matter?**

* A merchant can treat a confirmed payment as final.
* A buyer cannot ask a central payment provider to reverse a Bitcoin transaction.
* The system does not need a central intermediary to settle the payment.

Reference: [Bitcoin Whitepaper, Introduction](https://bitcoin.org/bitcoin.pdf)

### 2. The Network Maintains a Shared Transaction History

Bitcoin does not depend on one bank or company to maintain its transaction record. Nodes around the world store and verify copies of the blockchain.

The blockchain is a sequence of blocks containing transaction data. Each block is linked to the previous one.

**Why does this matter?**

* There is no single computer whose failure would stop the network.
* Participants can independently verify the transaction history.
* Changing old records requires overcoming the security provided by the network's consensus and proof-of-work.

### 3. Proof-of-Work Makes Attacks Expensive

Miners compete to find a valid proof-of-work for each block. The miner that produces the accepted block receives the block subsidy and transaction fees.

**Why does this matter?**

* Rewriting recent transaction history requires substantial computing power.
* A successful attack against the network would require significant resources.
* Miners have an economic reason to follow the rules and produce valid blocks.

Reference: [Bitcoin Whitepaper, Section 4](https://bitcoin.org/bitcoin.pdf)

### 4. Digital Signatures Authorize Transactions

When you spend Bitcoin, your wallet uses a private key to create a digital signature. The network uses that signature to verify that the transaction was authorized by someone who controls the relevant private key.

**Why does this matter?**

* Someone cannot spend your Bitcoin without the necessary private-key authorization.
* The network can verify that a transaction was authorized without knowing your identity.

### 5. SPV Allows Lightweight Verification

Not every Bitcoin user needs to store the entire blockchain. **SPV**, or Simplified Payment Verification, allows a lightweight client to check that a transaction happened without downloading and validating the complete blockchain itself.

An SPV client downloads block headers, follows the chain with the greatest accumulated proof-of-work, and requests a Merkle proof that a transaction is included in a block. It does not independently validate the full consensus ruleset the way a full node does. In other words, it is a lightweight wallet that relies on information provided by the network rather than independently validating every transaction and block.

**Why does this matter?**

* Lightweight wallets can run on devices with limited storage.
* Users can check relevant payment information without maintaining a full node.
* Bitcoin can be used without running a full copy of the blockchain, at the cost of trusting the network's chain rather than verifying it independently.

Reference: [Bitcoin Whitepaper, Section 8](https://bitcoin.org/bitcoin.pdf)

### Why Verification Matters

That trust trade-off, relying on the network instead of validating everything yourself, matters when looking at how Bitcoin has evolved.

One of the problems that became important for second-layer protocols was **transaction malleability**. A transaction's ID is calculated by hashing the transaction's data, and before SegWit that data included the signatures. Because a signature could be re-encoded in a different but still valid way without changing what the transaction actually spent, the transaction ID could change even though the payment itself remained the same.

That created problems for systems that needed to refer to transactions by their IDs.

The **Lightning Network** is a good example. Lightning uses Bitcoin transactions as part of its channel protocol, so being able to rely on stable transaction IDs was important. If a transaction ID could change after the transaction was created, protocols built around that transaction could be disrupted.

SegWit addressed this by moving the signature data outside the part of the transaction used to calculate the transaction ID.

This is why SegWit was more than a simple increase in transaction capacity. It fixed a property of Bitcoin transactions that was important for building protocols such as Lightning on top of Bitcoin.

It also shows an important part of Bitcoin development: changes to the base protocol can affect what kinds of systems can safely be built on top of it.

Reference: [Mastering Bitcoin](https://github.com/bitcoinbook/bitcoinbook) by Andreas M. Antonopoulos.

## How the Main Pieces Work Together

Bitcoin uses several mechanisms to solve different parts of the payment problem.

| Participant      | Concern                                                          | Bitcoin mechanism                                                        |
| ---------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------------ |
| Merchant         | Can the payment be reversed?                                     | Transaction confirmation and the absence of a central reversal mechanism |
| Buyer            | Who can authorize a payment?                                     | Digital signatures                                                       |
| Network          | Which transactions should be accepted?                           | Consensus                                                                |
| Attacker         | Can old transaction history be changed?                          | Proof-of-work                                                            |
| Lightweight user | How can I verify payments without storing the entire blockchain? | SPV                                                                      |

These mechanisms work together. Digital signatures authorize spending, nodes verify transactions, consensus determines which blocks are accepted, and proof-of-work makes it expensive to rewrite the history.

## What Bitcoin Does Not Do

Bitcoin does not solve every problem related to money or payments.

**It does not hide your identity.** Bitcoin transactions are public. The blockchain records addresses and transaction amounts, although an address does not directly reveal the real-world identity of its owner.

**It does not prevent mistakes.** If you send Bitcoin to the wrong address, there is no central customer-service department that can reverse the transaction.

**It does not work without network connectivity.** A Bitcoin transaction needs to reach the network before it can be confirmed.

**It does not provide instant final settlement.** Transactions are normally included in blocks roughly every ten minutes on average, although the actual confirmation time varies.

**It is not free.** Users may need to pay transaction fees when sending Bitcoin.

## Key Takeaway

The main idea behind Bitcoin is that participants can verify transactions without relying on a central institution to maintain the ledger.

The system combines cryptography, digital signatures, proof-of-work, and a decentralized network of nodes.

You do not have to trust a bank or another central payment provider to verify the Bitcoin transaction history. You can use the protocol and network rules to verify the information yourself.

Traditional payment systems usually rely on institutions to maintain the ledger and settle disputes.

Bitcoin moves much of that verification into the protocol and the network.

## Where to Learn More

If you want to go deeper, these are useful starting points:

1. [**Bitcoin Whitepaper**](https://bitcoin.org/bitcoin.pdf)
   Start with the abstract and introduction, then read through the sections on transactions, proof-of-work, and network operation.

2. [**Bitcoin's Academic Pedigree**](https://queue.acm.org/detail.cfm?id=3136559)
   This explains some of the ideas that existed before Bitcoin, including proof-of-work and digital signatures.

3. **Mastering Bitcoin by Andreas M. Antonopoulos**
   A more detailed technical introduction to Bitcoin. The book is available free on [GitHub](https://github.com/bitcoinbook/bitcoinbook).

4. [**Chaincode Labs Seminar Curriculum**](https://chaincode.gitbook.io/seminars/bitcoin-protocol-development)
   A useful resource for studying Bitcoin protocol development.

## What I'm Learning Next

This week I focused on the ideas behind Bitcoin and why the system works.

Next, I'm moving into the implementation details: how these concepts are represented in code and how the Bitcoin protocol works at the software level.

Bitcoin has a lot of technical detail, so the next step is to study each part more closely and understand how the pieces are implemented.

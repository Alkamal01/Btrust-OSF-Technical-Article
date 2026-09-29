# The Mempool Is Not Just a Waiting Room

When I first learned about Bitcoin's mempool, the explanation was simple:

> The mempool is where unconfirmed transactions wait before miners include them in a block.

That's a useful explanation when you're starting out.

But it leaves out a lot.

Who decides whether a transaction is allowed into the mempool? What happens when the mempool gets full? Why can a transaction exist in one node's mempool but not another? And if a transaction is valid, does that automatically mean every Bitcoin node has to accept and relay it?

I started looking through Bitcoin Core to understand what actually happens.

The first thing I realized is that it is better to stop thinking about **the mempool** as one global place.

## There Isn't One Global Bitcoin Mempool

Every Bitcoin node maintains its own mempool.

So instead of imagining this:

```text
                 Bitcoin Mempool
                /       |       \
             Node A   Node B   Node C
```

a better picture is:

```text
Node A              Node B              Node C
┌─────────┐         ┌─────────┐         ┌─────────┐
│ Mempool │         │ Mempool │         │ Mempool │
└─────────┘         └─────────┘         └─────────┘
```

These mempools can overlap heavily, but they do not have to contain exactly the same transactions.

To understand why, we first need to look at what happens when a transaction reaches a Bitcoin Core node.

## A Transaction Doesn't Just Walk Into the Mempool

In Bitcoin Core, one of the main pieces responsible for deciding whether a transaction gets accepted is `MemPoolAccept`.

For a normal single transaction, the path eventually reaches:

```cpp
MemPoolAccept::AcceptSingleTransactionInternal(...)
```

Before the transaction is added, Bitcoin Core performs a number of checks.

The interesting thing is that these checks are not all asking the same question.

Some are about whether the transaction follows Bitcoin's consensus rules.

Others are about whether the node wants that transaction in its mempool.

That difference is important.

## Consensus Rules and Mempool Policy Are Not the Same Thing

This was probably the most important thing I learned while looking through the acceptance code.

Bitcoin Core performs basic transaction checks and rejects things that cannot be valid transactions. For example, a coinbase transaction cannot simply arrive over the network and enter the mempool. Coinbase transactions have a special role inside blocks.

But Bitcoin Core also checks things such as:

- whether the transaction is standard;
- whether it is final enough to be mined in the next block;
- whether its inputs are available;
- whether it conflicts with another mempool transaction;
- whether its fee rate is sufficient;
- whether its scripts and witnesses satisfy relevant checks;
- and whether it violates other mempool limits.

Some of these are **policy decisions**, not Bitcoin consensus rules.

That gives us an important distinction:

```text
Consensus:
"Could this transaction be valid under Bitcoin's rules?"

Mempool policy:
"Am I willing to keep and relay this transaction?"
```

Those questions are related, but they are not identical.

A transaction being rejected from a node's mempool does not automatically mean that the transaction could never appear in a valid Bitcoin block.

That initially felt strange to me.

If nodes enforce Bitcoin's rules, why would they deliberately have another set of rules for their mempools?

The answer becomes clearer when you remember that a mempool consumes resources.

## Your Node Has Limited Memory

A node cannot keep an unlimited number of unconfirmed transactions forever.

Bitcoin Core therefore has to manage its mempool.

Two functions made this especially clear when I looked through the source:

```cpp
Expire(...)
```

and:

```cpp
TrimToSize(...)
```

They solve two different problems.

`Expire()` deals with transactions that have been sitting around for too long.

`TrimToSize()` deals with the mempool growing beyond its configured memory limit.

So transactions do not necessarily stay in the mempool until they are mined.

They can disappear for other reasons.

## Expiration Can Affect More Than One Transaction

Transactions in the mempool are not always independent.

Imagine:

```text
Transaction A
     │
     ▼
Transaction B
     │
     ▼
Transaction C
```

B spends an output created by A, and C spends an output created by B.

Bitcoin Core understands these relationships.

When an old transaction is expired, Bitcoin Core calculates its descendants as part of the removal process.

That makes sense.

If A disappears from the mempool, B and C depend on a transaction that is no longer there.

This was one of the points where the "waiting room" analogy really started breaking down for me.

The mempool isn't just a list.

There are relationships between transactions.

## What Happens When the Mempool Gets Full?

This gets even more interesting.

Bitcoin Core's `TrimToSize()` keeps removing transactions while the mempool's memory usage is above its configured limit.

But it doesn't simply remove whichever transaction arrived first.

It looks for a low-value part of the transaction graph to remove.

That transaction graph matters because one transaction can depend on another.

Consider this:

```text
Parent
Low fee
   │
   ▼
Child
High fee
```

Looking only at the parent might make it seem like an obvious transaction to remove.

But the child cannot be mined without the parent.

The fees therefore need to be understood in the context of their relationship.

This connects to **Child Pays for Parent**, or CPFP.

A child transaction can pay a sufficiently high fee that including the parent and child together becomes attractive.

So when Bitcoin Core manages limited mempool space, transaction relationships matter too.

## Getting Evicted Changes the Cost of Getting Back In

There was another detail in `TrimToSize()` that I found interesting.

When Bitcoin Core removes transactions because of memory pressure, it updates a rolling minimum fee rate.

Why?

Imagine the mempool is full and Bitcoin Core removes a low-fee transaction.

If another transaction paying essentially the same fee could immediately enter, the node could end up repeatedly accepting and evicting transactions without improving the mempool.

Instead, eviction can raise the minimum fee needed for new transactions to enter.

Conceptually:

```text
Mempool fills up
       ↓
Lower-value transactions are removed
       ↓
Rolling minimum fee increases
       ↓
Some new low-fee transactions are rejected
```

So the mempool reacts to demand for its limited space.

Again, that is much more than simply waiting for miners.

## Why Two Nodes Can Have Different Mempools

Now the earlier question becomes easier to answer.

Suppose we have two nodes:

```text
Node A                 Node B
```

A transaction reaches Node A.

Node A checks it and accepts it.

Later, Node B learns about the same transaction.

But Node B has its own mempool, its own current state and its own policy conditions.

Maybe Node B's mempool has recently been under greater pressure and its current minimum fee is higher.

Node B may reject the transaction while Node A keeps it.

Now we have:

```text
Node A                     Node B

┌────────────────┐         ┌────────────────┐
│ Transaction X  │         │                │
└────────────────┘         └────────────────┘
```

Neither node necessarily has a problem.

Their local mempools simply differ.

Other things can cause differences too.

Nodes may receive transactions at different times. One node may have seen a parent transaction that another hasn't seen yet. Transactions may have been evicted or expired. Node policy and configuration can also differ.

This is why thinking about **a node's mempool** is more accurate than imagining one globally synchronized Bitcoin mempool.

## So How Does a Transaction Spread?

Bitcoin's peer-to-peer network handles transaction relay.

One thing I found interesting in Bitcoin Core is that nodes commonly announce transaction inventory to peers rather than blindly pushing every complete transaction to everyone.

At a simplified level:

```text
Node A
  │
  │ announces transaction
  ▼
Node B
  │
  │ requests data it needs
  ▼
receives transaction
  │
  ▼
runs its own checks
  │
  ├── Reject
  │
  └── Accept
         │
         ▼
   Node B's mempool
```

Bitcoin Core can announce transaction inventory using a transaction's `txid`, or its `wtxid` when the peer supports wtxid relay.

It also takes the peer's fee filter into account when deciding what transaction inventory to announce.

That means transaction relay isn't simply:

> "I accepted this, so you should accept it too."

Each node makes its own decision.

Node B does not trust Node A's mempool policy.

It validates the transaction for itself.

That is an important part of Bitcoin's design.

## Then a Block Arrives

Suppose three nodes currently look like this:

```text
Node A mempool: Transaction X ✓

Node B mempool: Transaction X ✗

Node C mempool: Transaction X ✓
```

Then a miner produces a block containing Transaction X.

All three nodes can receive that block.

At that point, the important question is no longer:

> "Would I accept Transaction X into my mempool?"

The node now has to determine whether the **block itself satisfies Bitcoin's consensus rules**.

If the block is valid and becomes part of the active chain, the nodes can agree on it even though their mempools were different before the block arrived.

That gives us a useful distinction:

```text
Mempool
Local and policy-driven
        │
        │ transaction gets mined
        ▼
Blockchain
Consensus-driven
```

Nodes can disagree about what belongs in their mempools without necessarily disagreeing about what belongs in the blockchain.

## So What Is the Mempool?

After going through Bitcoin Core's transaction acceptance, expiration, eviction and relay code, I don't think of the mempool simply as a waiting room anymore.

A more useful description is:

> **A mempool is a node's locally managed set of unconfirmed transactions that it currently considers acceptable for potential inclusion in future blocks.**

The word **local** matters.

So does **managed**.

A transaction has to satisfy the node's acceptance rules to get in. It may compete for limited memory. Its relationship with other transactions can matter. It may be replaced, expire or get evicted. And other nodes do not have to make exactly the same decision about it.

Eventually, mining moves the discussion from local mempool policy to network consensus.

So the next time someone says:

> "The transaction is in the mempool."

there is a useful follow-up question:

**Whose mempool?**

*AI Usage Disclosure: I wrote and researched this article myself. I used AI only to check grammar and language, not for research, structure, or content.*
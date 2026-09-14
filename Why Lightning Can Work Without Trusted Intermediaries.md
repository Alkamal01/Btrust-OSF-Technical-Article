# Why Lightning Can Work Without Trusted Intermediaries

One of the things I wanted to understand while studying the Lightning Network was how payments can move through people you do not know without giving those people control over your money.

With Bitcoin, transactions are published to the blockchain and independently validated by nodes. Lightning works differently. Most channel activity happens off-chain, so the protocol needs a way for participants to update balances and settle disputes without relying on a bank or another trusted intermediary.

*AI Usage Disclosure: I wrote and researched this article myself. I used AI only to check grammar and language, not for research, structure, or content.*

## Payment Channels

A payment channel allows two people to make multiple payments between themselves without publishing every payment to the Bitcoin blockchain.

Suppose Alice and Bob open a channel. They create a funding transaction that locks bitcoin into an output they control together. Once the channel is open, they can update the distribution of those funds by creating new channel states.

For example, they might start with:

```text
Alice: 1 BTC
Bob:   1 BTC
```

After Alice pays Bob 0.1 BTC, the new state becomes:

```text
Alice: 0.9 BTC
Bob:   1.1 BTC
```

They can continue updating the channel without publishing each new state on-chain.

Eventually, they can close the channel and settle the final balance on Bitcoin.

This reduces the number of transactions that need to be recorded on the blockchain.

## The Problem With Old States

The part I initially found more interesting was what happens when one participant tries to use an old channel state.

Suppose the current balance is:

```text
Alice: 0.85 BTC
Bob:   1.15 BTC
```

An earlier state might have given Alice 0.9 BTC and Bob 1.1 BTC.

If Alice could simply broadcast that earlier commitment transaction and receive the old balance, she would have an incentive to do so.

Lightning therefore needs a way to make previously revoked states unsafe to publish.

## Revocation

When participants update a Lightning channel, they exchange information that allows the previous commitment state to be revoked.

If one party later broadcasts a revoked commitment transaction, the other party can use the revocation information associated with that state to claim the relevant funds through the penalty path.

This changes the incentive to publish an old state. A participant attempting to recover an outdated balance risks losing funds instead.

Timelocks are also important here. Certain outputs from a commitment transaction cannot be spent immediately by the party who broadcast it. This gives the other participant time to detect a revoked state and respond.

This was where I started to understand why monitoring matters in Lightning.

## Going Offline

A Lightning user does not have to watch the blockchain every second, but they cannot ignore their channels indefinitely.

If another participant broadcasts a revoked commitment transaction, the honest participant has a limited period to respond. The exact delay depends on how the channel was configured.

This creates a problem for someone who remains offline for a long time.

Watchtowers provide one way to address this. A watchtower can monitor the blockchain for a user and react when it detects a revoked commitment transaction.

## Paying Someone Without a Direct Channel

Payment channels become more useful when payments can move beyond two people.

Suppose Alice has a channel with Bob, and Bob has a channel with Charlie. Alice can potentially pay Charlie through Bob without opening a new channel directly with Charlie.

HTLCs help make this possible.

Charlie provides a payment hash. The corresponding preimage is needed to complete the payment. The payment can then travel through the route under conditions that allow each participant to claim the incoming payment when the outgoing payment succeeds.

Timelocks provide a way for participants to recover their funds if the payment does not complete.

Bob therefore does not need to receive Alice's money and then promise to forward it to Charlie. The payment conditions are enforced by the transactions and scripts used by the protocol.

Bob also only sees his own hop. He knows the previous node and the next node in the path, but not where the payment originally started or where it ends up. This is done through onion routing, where each node in the path can only unwrap the layer meant for it. So the trust problem is not just about the money, it is also about not having to give any single node the full picture of who is paying whom.

## There Are Still Tradeoffs

Studying Lightning also made it clear to me that moving payments off-chain introduces other problems.

**Liquidity matters.** A channel needs enough outbound liquidity in the required direction for a payment to pass through it.

**Routing matters.** The sender needs to find a path of channels capable of carrying the payment.

**Availability matters.** Nodes involved in forwarding a payment need to be reachable while the payment is being attempted.

**Monitoring matters.** Channel participants need a way to detect revoked commitment transactions before the relevant delay expires.

These are different problems from the ones I had been studying on Bitcoin's base layer.

## What I Learned

Before studying Lightning, I mostly understood it as a way to make Bitcoin payments faster. Payment channels explained how transactions could happen off-chain, but I had not understood how the protocol dealt with old channel states or payments through intermediate nodes.

Commitment transactions give channel participants an on-chain settlement option. Revocation handles outdated channel states. HTLCs allow payments to move across multiple channels without requiring the sender to trust each intermediate node.

The part I still want to understand better is how these mechanisms work in current Lightning implementations, especially channel backups, watchtowers, routing, and liquidity management.

That is where I plan to continue from here.

## References

- [Lightning Network Whitepaper (Poon & Dryja)](https://lightning.network/lightning-network-paper.pdf)
- [BOLT #4: Onion Routing Protocol](https://github.com/lightning/bolts/blob/master/04-onion-routing.md)

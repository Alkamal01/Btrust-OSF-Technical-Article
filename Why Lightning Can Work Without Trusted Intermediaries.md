# Why Lightning Can Work Without Trusted Intermediaries

One of the things I wanted to understand while studying the Lightning Network was how payments can move through people you do not know without giving those people control over your money.

Lightning still relies on other people's nodes to route a payment when you do not have a direct channel with the person you are paying. What changes is not whether intermediaries exist, but whether you have to trust them with custody of your funds while they do it.

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

Charlie provides a payment hash. The corresponding preimage is needed to complete the payment.

The detail that made this click for me is that every hop in the route uses the same hash. Charlie generates a secret, the preimage, and gives Alice its hash as part of an invoice. Alice sends an HTLC to Bob locked to that hash, and Bob sends his own HTLC to Charlie locked to the same hash. Charlie is the only one who knows the preimage, so he is the only one who can claim the payment Bob sent him, and claiming it means revealing the preimage to do so.

Once Bob sees that preimage, he can use it to claim the HTLC Alice sent him. But if Bob never forwards the payment to Charlie, he never learns the preimage, and he has no way to claim Alice's HTLC either. That is what stops Bob from simply keeping Alice's payment without completing his part of the route.

Timelocks provide a way for participants to recover their funds if the payment does not complete. If Bob never forwards it, or Charlie never claims it, the HTLC expires and Alice gets her funds back.

Bob therefore does not need to receive Alice's money and then promise to forward it to Charlie. The payment conditions are enforced by the transactions and scripts used by the protocol.

Bob also does not learn the complete route from the onion packet Alice constructs; he only receives the information needed to forward his part of the payment, meaning which node to send it to next and under what conditions. He does not learn where the payment originally started or where it ultimately ends up, since each node in the path can only unwrap the layer meant for it. So the trust problem is not just about the money, it is also about not having to give any single node the full picture of who is paying whom.

## There Are Still Tradeoffs

Studying Lightning also made it clear to me that moving payments off-chain introduces other problems.

**Liquidity matters, and it is not the same as capacity.** A channel's capacity is the total amount locked in the funding transaction. The example earlier in this article had a capacity of 2 BTC. But at any given moment, only one side's balance is available to send in a given direction. If Alice's balance drops to 0.1 BTC, she only has 0.1 BTC of outbound liquidity left, even though the channel's capacity has not changed. A payment needs enough liquidity on the sending side, in the direction it needs to move, and capacity alone does not guarantee that.

**Routing matters.** The sender needs to find a path of channels capable of carrying the payment.

**Availability matters.** Nodes involved in forwarding a payment need to be reachable while the payment is being attempted.

**Monitoring matters.** Channel participants need a way to detect revoked commitment transactions before the relevant delay expires.

These are different problems from the ones I had been studying on Bitcoin's base layer.

## What I Learned

Before studying Lightning, I mostly understood it as a way to make Bitcoin payments faster. Payment channels explained how transactions could happen off-chain, but I had not understood how the protocol dealt with old channel states or payments through intermediate nodes.

Commitment transactions give channel participants an on-chain settlement option. Revocation handles outdated channel states. HTLCs allow payments to move across multiple channels without requiring the sender to trust each intermediate node.

The part I still want to understand better is how these mechanisms work in current Lightning implementations, especially channel backups, watchtowers, routing, and liquidity management.

That is where I plan to continue from here.

## References & Further Reading

**Official Lightning Documentation:**
- [Lightning Network Whitepaper (Poon & Dryja)](https://lightning.network/lightning-network-paper.pdf)
- [BOLT Specifications](https://github.com/lightning/bolts) — the official technical specs, including [BOLT #4: Onion Routing Protocol](https://github.com/lightning/bolts/blob/master/04-onion-routing.md)
- [Lightning Network Explained](https://www.bitcoin.com/get-started/blockchain-tech/layer-2s-scaling/what-is-lightning-network/) — Bitcoin.com overview

**Deep Dives:**
- [History of the Lightning Network (Christian Decker)](https://btctranscripts.com/chaincode-labs/chaincode-residency/2018-10-22-christian-decker-history-of-lightning/) — how Lightning was invented
- [LN Things Part 4: HTLC Overview (Elle Mouton)](https://ellemouton.com/posts/htlc/) — technical explanation of the hash-and-timelock mechanism
- [Revocable Transactions with LN-Penalty](https://www.derpturkey.com/revocable-transactions-with-ln-penalty/) — how revocation keys work

**To Use Lightning (Beginner-Friendly Wallets):**
- [Strike](https://strike.me/) — send Bitcoin instantly
- [Breez](https://breez.technology/) — mobile Lightning wallet
- [Blue Wallet](https://bluewallet.io/) — Bitcoin + Lightning mobile wallet

**To Run a Node:**
- [LND (Lightning Network Daemon)](https://github.com/lightningnetwork/lnd) — most popular implementation
- [Core Lightning](https://github.com/ElementsProject/lightning) — alternative implementation
- [Eclair](https://github.com/ACINQ/eclair) — Kotlin implementation

**For More Technical Understanding:**
- [Explaining Bitcoin's Payment Channels (Lightspark)](https://www.lightspark.com/glossary/channel)
- [Timelocks (Lightning Engineering Builder's Guide)](https://docs.lightning.engineering/the-lightning-network/multihop-payments/timelocks)
- [Visualizing HTLCs and the Lightning Network's Dirty Little Secret (Peter R. Rizun)](https://medium.com/@peter_r/visualizing-htlcs-and-the-lightning-networks-dirty-little-secret-cb9b5773a0)

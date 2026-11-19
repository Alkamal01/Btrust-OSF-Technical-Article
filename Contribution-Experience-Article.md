# Beyond the Diff: Lessons from My Btrust Open-Source Fellowship in LDK

When I joined the Btrust Open-Source Fellowship, I already knew how to find my way around a repository and put together a Rust patch. I had not yet learned how much more is involved in changing a protocol implementation safely. In the Lightning Dev Kit (LDK), a small change can affect payment state, persistence, concurrency, or what a peer can infer from a message.

The fellowship helped me move from asking, “Does this patch work?” to asking, “What assumptions does it rely on, and what happens in the surrounding system?” Weekly sessions with experienced contributors and hands-on work helped me build that habit. During the fellowship, I investigated five issues, opened two pull requests, reviewed thirteen PRs, and wrote ten technical articles about Bitcoin and Lightning. The numbers show what I completed, but the bigger change was learning to reason about the code around the diff.

One of the biggest changes was in how I responded when a test or review exposed a gap in my understanding. Earlier, I was more likely to focus on making the immediate failure disappear. During the fellowship, I became more willing to pause, trace the surrounding state, and ask what the failure was revealing about the design. That shift often took longer, but it produced better questions and more useful changes.

## Code contributions

During the fellowship, I worked on two rust-lightning changes. One was closed after an automated PR addressed the same issue; the other is my trampoline amount-validation contribution.

### Refund invoice expiry

In [PR #5484](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/pulls/5484), I worked on an issue where a BOLT 12 refund invoice could arrive after its expiry but before the next timer tick. The pending payment still existed during that interval, so it could be accepted and paid even though the payer's deadline had passed.

I added an expiry check at the shared transition from AwaitingInvoice to InvoiceReceived, using the same clock rules as stale-payment cleanup. I also added a unit test and an end-to-end test to check that the expired request is removed and produces one InvoiceRequestExpired failure event.

The issue had asked for an automated PR, but I missed that instruction and started implementing the fix. The bot later opened a PR for the same issue, and mine was closed. I linked my work in the bot PR discussion. I learned to read the issue description and check existing PRs before starting, and to ask maintainers if it is unclear whether an issue is available to work on.

### Trampoline receive amount validation

In [PR #5585](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/pulls/5585), I addressed a gap in trampoline receive validation. The receiver checked that the authenticated outer onion amount covered the inner request, but also needed to compare the actual incoming HTLC with that outer amount.

The change covers regular and blinded trampoline receives and preserves the explicitly allowed skimmed-fee amount. I added a regression test for underpayment, exact payment, and the permitted skim boundary. The focused test passed and failed when I removed the production check. Formatting and the no-default-features check passed. The full test suite was not completely green: two splice and anchor-reserve tests failed outside the changed trampoline path, while the focused checks for this contribution passed.

This work reinforced that validating two values carried in authenticated payloads is not enough by itself. The receiver also has to compare the actual value arriving at the boundary.

## Reviewing other contributors' work

I reviewed thirteen rust-lightning PRs during the fellowship. Rather than treating each diff as a self-contained change, I tried to follow important values and state through the code paths that use, store, or recover them. Four reviews especially changed how I think about edge cases.

In [PR #5428](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/pulls/5428), which rejects intercept SCIDs already assigned to another LSPS2 flow, I noticed that the duplicate check and map insertion used separate locks. Two callers could both see an SCID as free and then overwrite each other's mapping. I raised the race in the review, and the author updated the code to hold the write lock across both operations. It was a concrete reminder that a sequential test does not prove a check-and-update sequence is safe under concurrency.

[PR #5515](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/pulls/5515) groups wallet coins by claim to reduce how many claims a forced selection displaces. I followed the aggregate ranking into the later weight checks and found that an oversized UTXO could improve a claim's rank even though selection would skip that coin. I posted a comment asking whether weight-ineligible coins should be excluded from the aggregate.

In [PR #5525](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/pulls/5525), I looked at how the new ResourceManager saves and rebuilds in-flight HTLC state. I traced what happens when a channel closes and the node later restarts, and questioned whether a pending HTLC could lose its association with the peer whose reputation should be updated when it resolves. I also checked when the enable flag is read, since the configuration can be changed after construction.

[PR #5584](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/pulls/5584) changes gossip backfill to honor timestamp filters. Following the backfill cursor alongside the relay path led me to a possible gap: if a channel is skipped because its stored updates are outside the filter, a later update might be relayed without the channel announcement the receiving peer needs. This review taught me to check not only what a filter excludes, but also what later messages depend on.

Other reviews gave me a wider view of the codebase. I checked fee and CLTV accounting for a blinded-path fix in [#5127](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/pulls/5127) and ran its focused rounding test. I traced lock order in [#5427](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/pulls/5427), shared-state persistence in [#5454](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/pulls/5454), and final shutdown persistence in [#5409](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/pulls/5409). I also followed splice contribution ownership in [#5475](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/pulls/5475), checked documentation and serialization coverage in [#5499](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/pulls/5499), and checked that [#5500](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/pulls/5500) retained the important cancellation and replay cases. Finally, I looked at deterministic pruning in [#5579](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/pulls/5579) and type-level secret redaction in [#5474](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/pulls/5474). Together, these reviews taught me to ask what happens on retry, shutdown, reload, or concurrent execution—not only on the expected path.

## Issues I investigated

In [Issue #5431](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/issues/5431), I traced why a refund invoice could arrive after its deadline and still be accepted before stale-payment cleanup ran on the next timer tick. That investigation led to the work described in [PR #5484](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/pulls/5484). The coordination lesson from that contribution is discussed above and below.

[Issue #5446](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/issues/5446) described a wallet selection failure: adding one heavier confirmed coin could make the selector discard smaller candidates and fail even though a valid set of inputs remained. I followed the coin ordering and weight-budget logic and considered a fallback selection pass that skips coins that do not fit. The investigation helped me understand how wallet selection can affect time-sensitive anchor claims.

For [Issue #5447](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/issues/5447), I examined the retry overpayment ceiling, where multiplying a large `u64` payment amount by 110 can overflow before division. A wider intermediate calculation avoids that edge. It is a defensive arithmetic issue rather than a likely problem for ordinary channel payments, but the check is intended to bound router output and should remain safe across the supported value range.

In [Issue #4898](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/issues/4898), I traced how BOLT 11 fields are parsed and then re-encoded when the invoice signature hash is computed. Some non-minimal wire encodings can be normalized during re-encoding, so the bytes represented by parsed values may differ from what was received. The discussion clarified that the reader requirements should be resolved at the specification level as well as in the parser.

[Issue #5416](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/issues/5416) became my second code contribution. I reproduced the mismatch between the authenticated outer trampoline amount, the inner amount, and the actual incoming HTLC. The fix in [PR #5585](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/pulls/5585) checks the actual amount for regular and blinded receives while preserving the allowed skimmed-fee amount. Its regression test covers underpayment and the accepted boundaries.

Across these reviews and investigations, I learned to follow a value or state record through the whole path: how it enters, what checks it passes, how it is stored, and what happens when the system retries, restarts, or handles an edge case. That habit helped me move from understanding a report to identifying the code path and test that could demonstrate the behavior.

## Technical writing

I contributed ten articles to the [Btrust OSF Technical Article repository](https://github.com/Alkamal01/Btrust-OSF-Technical-Article) during the fellowship. They helped me study protocol questions closely and explain them to readers with different levels of technical experience.

- [How Money Works Without Banks](https://github.com/Alkamal01/Btrust-OSF-Technical-Article/blob/main/How%20Money%20Works%20Without%20Banks.md) introduces Bitcoin's approach to digital payments, verification, and proof of work.
- [Why Lightning Can Work Without Trusted Intermediaries](https://github.com/Alkamal01/Btrust-OSF-Technical-Article/blob/main/Why%20Lightning%20Can%20Work%20Without%20Trusted%20Intermediaries.md) explains payment channels, revoked states, and the difference between routing through intermediaries and giving them custody.
- [The Mempool Is Not Just a Waiting Room](https://github.com/Alkamal01/Btrust-OSF-Technical-Article/blob/main/The%20Mempool%20Is%20Not%20Just%20a%20Waiting%20Room.md) explains local mempools and the difference between consensus rules and relay policy.
- [Why Lightning Payments Fail Even When the Route Looks Good](https://github.com/Alkamal01/Btrust-OSF-Technical-Article/blob/main/Why%20Lightning%20Payments%20Fail%20Even%20When%20the%20Route%20Looks%20Good.md) covers liquidity, routing information, and why a route is not a reservation.
- [What Actually Happens When a Lightning Channel Closes](https://github.com/Alkamal01/Btrust-OSF-Technical-Article/blob/main/What%20Actually%20Happens%20When%20a%20Lightning%20Channel%20Closes.md) describes cooperative and force closes and the on-chain steps that follow.
- [Can Lightning Nodes Track Your Payments?](https://github.com/Alkamal01/Btrust-OSF-Technical-Article/blob/main/Can%20Lightning%20Nodes%20Track%20Your%20Payments.md) examines what onion routing hides and how shared HTLC payment hashes can help nodes correlate payments.
- [How Lightning Fees Are Calculated Across Multiple Hops](https://github.com/Alkamal01/Btrust-OSF-Technical-Article/blob/main/How%20Lightning%20Fees%20Are%20Calculated%20Across%20Multiple%20Hops.md) explains base and proportional fees along a route.
- [Why Payment Secrets Matter in Lightning Invoices](https://github.com/Alkamal01/Btrust-OSF-Technical-Article/blob/main/Why%20Payment%20Secrets%20Matter%20in%20Lightning%20Invoices.md) distinguishes payment hashes from payment secrets and explains their roles.
- [How Multi-Part Lightning Payments Split and Reassemble](https://github.com/Alkamal01/Btrust-OSF-Technical-Article/blob/main/How%20Multi-Part%20Lightning%20Payments%20Split%20and%20Reassemble.md) explains how separate HTLCs make up one MPP.
- [The Hidden Amount in a Lightning Payment](https://github.com/Alkamal01/Btrust-OSF-Technical-Article/blob/main/The%20Hidden%20Amount%20in%20a%20Lightning%20Payment.md) examines why trampoline receivers need to validate the outer amount as well as the invoice and inner payload.

The articles changed through review. I clarified the SPV trust model in [one revision](https://github.com/Alkamal01/Btrust-OSF-Technical-Article/commit/f34a468631bde133e913eff005cc7cf08c95cd38), and revised the Lightning trust article after feedback on custody, HTLC atomicity, onion routing, and capacity versus liquidity in [another revision](https://github.com/Alkamal01/Btrust-OSF-Technical-Article/commit/af2626b96936a9c205922061a881a01672ddca1d). Review also helped me make the mempool article clearer about transaction relationships and policy checks, and tighten the privacy article's explanation of HTLC timeouts and PTLCs. Some of the linked review discussions may require access to the fellowship's Git service.

Writing alongside code helped me notice when I understood a concept only at a surface level. To explain it simply, I had to go back to the relevant BOLTs, implementation, or review feedback.

## Feedback and what I would do differently

The most specific feedback I received came from reviews of my technical articles. On “Why Lightning Can Work Without Trusted Intermediaries,” Daniel asked me to distinguish routing through intermediaries from giving them custody, and to be more precise about HTLC atomicity, onion routing, and the difference between channel capacity and available liquidity. I revised the article in [this update](https://github.com/Alkamal01/Btrust-OSF-Technical-Article/commit/af2626b96936a9c205922061a881a01672ddca1d). That feedback showed me that a simple explanation still has to preserve the protocol's distinctions; otherwise, a reader can leave with the wrong idea even if the broad point sounds right.

For “How Money Works Without Banks,” reviewer feedback led me to clarify the SPV trust model in [a separate revision](https://github.com/Alkamal01/Btrust-OSF-Technical-Article/commit/f34a468631bde133e913eff005cc7cf08c95cd38). On the mempool article, feedback about transaction relationships and policy checks led to revisions and diagrams. I also tightened the privacy article's explanation of HTLC timeouts and PTLC terms after review. Across these edits, I learned to treat questions from reviewers as a signal to check my explanation against the source material, not just to rephrase a sentence.

Review discussions changed how I look for code bugs too. In [#5428](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/pulls/5428), I asked whether the duplicate-SCID check could race with insertion. The fix was to make both steps atomic under one write lock. That exchange reminded me to examine concurrent interleavings separately from the sequential behavior a unit test demonstrates. In [#5515](https://git.rust-bitcoin.org/lightningdevkit/rust-lightning/pulls/5515), following the claim ranking into the weight checks helped me make my comment specific: a coin that cannot be selected can still affect which claim is chosen.

The lesson from #5484 was about coordination. The issue had asked for an automated PR, and I missed that before I started coding. If I were beginning again, I would check the issue instructions and existing PRs first, then confirm ownership with maintainers before starting when it is unclear. I would also write the smallest regression test that captures the failure before expanding the implementation, and check my branch against the current base before interpreting unrelated test failures.

## What comes next

The fellowship has given me a stronger working knowledge of payment handling, channel state, persistence, gossip, and routing in LDK. After the fellowship, I plan to keep contributing to open source with a more deliberate approach: choosing issues with maintainers, testing changes against the behavior they are meant to fix, and staying engaged through review. I also want to carry my trampoline contribution through review and continue writing technical articles.

The most useful result has not been a count of PRs. It has been learning to follow a technical problem from the report, through the code and tests, and into the collaboration that makes a change useful to the project.

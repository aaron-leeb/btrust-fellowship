
The purpose of this article is to introduce readers to Compact Block Filters. It will discuss what they are, how they improve on previous solutions, where the implementation is found in Bitcoin Core, and more.

First, it's important to understand the problem Compact Block Filters are meant to solve. In Bitcoin, not every node is a full node. To be considered a full node, it's required to download the entire blockchain and store the set of Unspent Transaction Outputs. Full nodes are able to verify transactions AND consensus rules at the cost of increased resource utilization. 

Although Bitcoin was designed for anyone to be able to run a full node, running one comes with the responsibility of maintaining hardware and some technical overhead. These tradeoffs might not be appealing to users who care more about convenience. 

Light clients are the alternative to running a full node and often power wallets. They allow users to run a node on resource constrained hardware like a phone. Satoshi mentioned light clients in the [whitepaper](https://bitcoin.org/bitcoin.pdf) when he outlined Simplified Payment Verification (SPV), a method for verifying the inclusion of a transaction in a block by obtaining a Merkle proof. 

SPV works great as a verification mechanism in Bitcoin, but doesn't provide a way to discover transactions efficiently. In other words, unless a user knows the block they're interested in, they're still required to download, at a minimum, ever block header and possibly the block. This is the problem Compact Block Filters are meant to solve, but first let's look at the previous solution.

### Bloom Filters (BIP 37)

Bloom filters were introduced in Bitcoin Core v0.8.0 in 2013 via [PR #1795](https://github.com/bitcoin/bitcoin/pull/1795). They were meant to solve the issues that light clients using SPV face, such as synchronization process slow downs, user bandwidth waste, and increased memory usage. These optimizations were accomplished by allowing light clients to build custom filters that hash relevant transaction data. 

These filters were designed to include "noise" to help with privacy. This "noise" is basically just transaction data that is not actually of interest to the light client. In the case of wallets powered by light clients, Bloom Filters provide a more secure mechanism for requesting transaction data containing their addresses and scriptPubKeys than sharing them directly with a full node.

Despite the benefits Bloom Filters provide for light clients, they have some major problems that eventually led to their replacement in Bitcoin Core. One of these problems was that over time a full node could begin to identify patterns in the transaction data being requested, essentially making null and void the privacy benefits they were meant to provide. 

Another problem was that bloom filters made full nodes susceptible to denial of service (DOS) attacks because a light client (or many) could hypothetically send infinite filters to a full node which was required to load the relevant blocks from disk to determine the transactions that match the filters. 

Making these filters is rather easy, allowing for each filter to be different, but the burden of fulfilling these filter requests falls to the full node and requires memory intensive work to complete. You can see how this could impact a full node's performance rather quickly. 

Bitcoin Core has since disabled BIP 37 filtering by default as of 2019 and Core v0.19. There replacement was Compact Block Filters (BIP 158).

### What Are Compact Block Filters?

Compact block filters provide a privacy-preserving mechanism for light clients to find relevant transactions in the blockchain without leaking their addresses. BIP 158 defines the *basic* block filter type called the Compact Block Filter as a probabilistic data structure that utilizes Golomb-Rice encoding. For those interested in learning more about the technical workings of Compact Block Filters, I would encourage you to read [BIP 158](https://github.com/bitcoin/bips/blob/master/bip-0158.mediawiki) 

Although this article is about BIP 158, it is unavoidable to mention BIP 157. [BIP 157](https://github.com/bitcoin/bips/blob/master/bip-0157.mediawiki) describes the light client protocol that transmits Compact Block Filters. To make this possible, there were new peer to peer (P2P) messages added, along with a new chain of filter headers all the way back to the genesis block similar to block headers.

The PRs to implement BIP 157 and 158 are: [#12254](https://github.com/bitcoin/bitcoin/pull/12254) [#14121](https://github.com/bitcoin/bitcoin/pull/14121), and [#18876](https://github.com/bitcoin/bitcoin/pull/18876)

### Some of Improvements Compact Block Filters Offer

Compact Block Filters can be seen as the opposite of Bloom Filters in many ways. The burden of creating custom filters for each light client is no longer the responsibility of the full node. Instead, the full node creates a filter containing the input and output [scriptPubKeys](https://learnmeabitcoin.com/technical/transaction/output/scriptpubkey/) from each transaction within a block. 

The filters are deterministic meaning that all full nodes end up with the same binary payload for a block. This single block filter can then be served to many light clients. In other words, Compact Block Filters become client-side discovery mechanisms since the light client is required to do the work to identify their addresses.

Since all the filters are deterministic, a light client can ask multiple full nodes for a Compact Block Filter to verify which nodes are honest. Also, like mentioned previously, the filters have headers that point to the previous filter header similar to the block headers ensuring the integrity of the filters.

These are examples of how Compact Block Filters protect the privacy and security of the light clients. Whereas Bloom Filters force the light client to place blind trust in the full node to deliver a complete and honest set of transactions. That doesn't align very well with the Bitcoin motto: Don't trust, verify. 

If the light client finds that one of the address it tracks was included in a block, it asks the full node to deliver the full block for it to download. This preserves more privacy for the light client because although a full node might know which block is requested, it has no idea what addresses within the block the light client is interested in.
### Filter Creation Simplified

To compress the list of scriptPubKeys in a block, they undergo a series of steps. The functions that are used can be found in [`src/blockfilter.cpp`](https://github.com/bitcoin/bitcoin/blob/master/src/blockfilter.cpp) and [`src/blockfilter.h`](https://github.com/bitcoin/bitcoin/blob/master/src/blockfilter.h) in the Bitcoin Core codebase.

Here are the steps put more simply:

1. Hash: convert each item into a 64 bit number using SipHash
2. Map and sort: map the numbers into a limited range, then sort them in ascending order.
3. Calculate gaps: compute the difference between each value and the previous one
4. Encode: use Golomb-Rice encoding to get binary. Each gap is represented as two parts: a quotient and a remainder.
   
A light client that receives a filter can then do this series of steps in reverse to identify whether one of their addresses was in the block.

If you're interested in learning more about Compact Block Filters there are many other articles that inspired this one including: [BIP158: Compact Block Filters Deep Dive](https://www.ellemouton.com/posts/bip158/), [Book of BDK](https://bookofbdk.com/design/cbf/#distribution-of-peers), and [Bitcoin Optech](https://bitcoinops.org/en/topics/compact-block-filters/).












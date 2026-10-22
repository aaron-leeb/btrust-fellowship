
The intention of this article is to introduce readers to compact block filters. It will discuss what they are, how they compare to previous solutions, where the implementation is found in Bitcoin Core, and more.

First, it's important to understand the problem compact block filters are meant to solve. In Bitcoin, not every node is a full node. To be considered a full node, it's required to download the entire blockchain and store the set of Unspent Transaction Outputs. Full nodes are able to verify transactions AND consensus rules at the cost of increased resource utilization. 

Although Bitcoin was designed for anyone to be able to run a full node, running one comes with the responsibility of maintaining hardware and some technical overhead. These tradeoffs might not be appealing to users who care more about convenience. Light clients are the alterative to running a full node and often power wallets. They allow users to run a node on resource constrained hardware like a phone. Satoshi mentioned light clients in the [whitepaper](https://bitcoin.org/bitcoin.pdf) when he outlined Simplified Payment Verification (SPV), a method for verifying the inclusion of a transaction in a block by obtaining a Merkle proof. 

SPV works great as a verification mechanism in Bitcoin, but doesn't provide a way to discover transactions efficiently. In other words, unless a user knows the block they're interested in, they're still required to download every block. This is the problem compact block filters are meant to solve, but first let's look at the previous solution.

### Bloom Filters (BIP 37)

Bloom filters were introduced in Bitcoin Core v0.8.0 in 2013 via [PR #1795](https://github.com/bitcoin/bitcoin/pull/1795). They were meant to solve the issues that light clients using SPV face, such as synchronization process slow downs, user bandwidth waste, and increased memory usage. These optimizations were accomplished by allowing light clients to build filters 




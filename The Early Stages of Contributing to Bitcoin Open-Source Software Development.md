
As I begin the 12th and final week of the BTrust Bitcoin Open-Source Fellowship, I find myself reflecting on the transformational experience it has been. From the opening call until now, it has challenged me to grow in ways I hadn't anticipated. Not only pertaining to my new knowledge of Bitcoin technology and the process of contributing to open-source software, but so much more. It has pushed me to become better at learning, communicating, problem solving, and time management. 

Bitcoin is one of the main reasons I returned to school to complete my bachelor's degree in computer science. Although my academic experience has provided me with knowledge of software development fundamentals and project-based learning, this fellowship has introduced me to an entirely different learning experience. Unlike the structured assignments and clearly defined expectations of school, contributing to open-source software requires a much greater degree of independence and self-direction. Often, there is nobody telling you what to work on, how to approach a problem, or when it needs to be completed. Instead, it requires the curiosity to explore unfamiliar codebases, the initiative to identify meaningful problems, and the persistence to work through challenges without always knowing the solution.

The third month of the fellowship is meant to focus on contributing to open-source projects. When I first began exploring opportunities to contribute, I underestimated how much time it would take to to choose a project. Since we spent a considerable amount of time studying the Bitcoin Core codebase, I figured it would make sense to continue. I also decided to  explore contributing to Sparrow Wallet, a project I was personally interested in as a user of the software.

## What I Built & the Feedback I Received

### Bitcoin Core

I started small after finding a [comment](https://github.com/bitcoin/bitcoin/pull/36176#discussion_r3944837660) on a Bitcoin Core pull request (PR) suggesting additional test coverage for an edge case. The original [PR #36176](https://github.com/bitcoin/bitcoin/pull/36176), opened by Rob1Ham, fixed a crash that occurred when creating a wallet in Bitcoin-Qt 31.1 with the `-nosettings` flag enabled. The issue stemmed from attempting to write wallet startup settings when dynamic settings were disabled.

Reviewing this PR was a valuable learning opportunity. I reverted the fix locally to reproduce the original issue and spent time understanding the changes that resolved it. Since the PR had already been merged without the suggested additional test coverage, I seized the opportunity to open my first [Bitcoin Core PR #36378](https://github.com/bitcoin/bitcoin/pull/36378).

The assertions I added to `wallet_startup.py` verify that reloading a wallet with `load_on_startup=True`, when it is already included in the startup list, correctly evaluates to `SKIP_WRITE` and does not emit a warning under `-nosettings`.

More specifically, the test covers the short-circuit evaluation of a condition in `src/node/interfaces.cpp`. When the action is `SKIP_WRITE`, the first condition evaluates to true, avoiding the need to check for a settings path or attempt to write to the settings file. This protects against a regression where changing the order of the conditions could cause an unnecessary warning.

Although this was a relatively small contribution, it helped me become more familiar with Bitcoin Core's functional testing framework and demonstrated how even a few additional assertions can protect important behavior.

I then moved on to implementing test coverage for an error raised by the `getblockfilter` RPC command when a node has received a block header but the corresponding block has never been connected. I identified this gap using the [Bitcoin Core coverage reports](https://corecheck.dev/master-coverage).

My interest in `getblockfilter` came from a presentation I participated in during week eight of the fellowship on BIP 158: Compact Block Filters. Following the advice of one of our mentors, I decided to continue exploring how compact block filters were implemented and tested in Bitcoin Core.

I opened [PR #36613](https://github.com/bitcoin/bitcoin/pull/36613) to add functional test coverage to `rpc_getblockfilter.py` for a previously untested condition in `src/rpc/blockchain.cpp`:

`if (!block_was_connected)`

To reproduce this scenario, the test disconnects two nodes, has one node mine a new block, and sends only the block header to the other node using `submitheader()`. At this point, the receiving node recognizes the block header but has not received or connected the complete block, meaning its compact block filter is unavailable.

The test verifies that `getblockfilter` returns error code `-5` with the expected message indicating that the block was never connected to the active chain. This distinction matters because Bitcoin Core should distinguish between an entirely unknown block and a known block header for which a compact block filter has never been constructed.

Both of my Bitcoin Core PRs were eventually merged, but the review process taught me something about contributing to open-source projects: patience is essential. While I received relatively little feedback, the ACKs and reviews came over time rather than immediately after submission.

A valuable [suggestion](https://github.com/bitcoin/bitcoin/pull/36613#pullrequestreview-5800773314) came on my `getblockfilter` PR. A reviewer pointed out that I could simplify the test by generating a block without submitting it, eliminating the need to disconnect, synchronize, and reconnect the nodes. Although the original approach worked, the suggested alternative was more streamlined and helped me recognize the importance of writing tests that are not only correct but also efficient and easy to maintain.
### Sparrow Wallet

My final major contribution was to [Sparrow Wallet](https://github.com/sparrowwallet/sparrow/pull/2120). My original design document focused on adding test coverage to the project, but after creating an issue on the repository and receiving valuable feedback from the main maintainer, I shifted my focus toward wallet interoperability.

While exploring the project, I found a function called `getUROutputDescriptor()` that converts wallet configurations into a standardized format used to export wallet descriptors through Uniform Resources (UR) QR codes. I noticed that this function had no existing test coverage, so I decided to write JUnit tests to verify that important wallet information was preserved during the conversion.

Specifically, I focused on the following branch of the function, which handles multisignature wallets:

``` java
else if(wallet.getPolicyType() == PolicyType.MULTI_HD) {
    WalletNode firstReceive = new WalletNode(wallet, KeyPurpose.RECEIVE, 0);
    Utils.LexicographicByteArrayComparator lexicographicByteArrayComparator = new Utils.LexicographicByteArrayComparator();
    List<CryptoHDKey> cryptoHDKeys = wallet.getKeystores().stream().sorted((keystore1, keystore2) -> {
        return lexicographicByteArrayComparator.compare(keystore1.getPubKey(firstReceive).getPubKey(), keystore2.getPubKey(firstReceive).getPubKey());
    }).map(keystore -> {
        KeyDerivation keyDerivation = keystore.getKeyDerivation();
        return getCryptoHDKey(keyDerivation.getMasterFingerprint(), keyDerivation.getDerivation(), keystore.getExtendedPublicKey(), keystore.getLabel());
    }).collect(Collectors.toList());
    MultiKey multiKey = new MultiKey(wallet.getDefaultPolicy().getNumSignaturesRequired(), null, cryptoHDKeys);
    List<ScriptExpression> multiScriptExpressions = new ArrayList<>(scriptExpressions);
    multiScriptExpressions.add(ScriptExpression.SORTED_MULTISIG);
    registryItem = new CryptoOutput(multiScriptExpressions, multiKey);
}
```

This code takes the individual keystores associated with a multisignature wallet, sorts them by their public keys, and converts each into a `CryptoHDKey` representation containing information such as the extended public key, master fingerprint, and derivation path. It then combines these keys with the required signing threshold into a `MultiKey` object before constructing the final `CryptoOutput`.

My tests use existing Coldcard multisignature wallet fixtures to verify that information such as script expressions, signing thresholds, cosigner public key information, fingerprints, and derivation paths are correctly preserved during this process.

A valuable test I added uses a multisignature wallet where cosigners have different derivation paths. This is important because the conversion must preserve each cosigner's individual derivation information rather than assuming they all use the same path. Losing or incorrectly associating this information could result in compatibility issues when importing the exported wallet configuration into another application or hardware wallet.

Through this contribution, I gained a better understanding of Bitcoin wallet interoperability and the importance of testing how data is converted between different representations. I also experienced the open-source review process firsthand, receiving feedback from the maintainer about avoiding tests that enforce behavior likely to change, following existing coding conventions, and focusing on meaningful regression coverage.

One particularly valuable piece of [feedback](https://github.com/sparrowwallet/sparrow/pull/2120#issuecomment-6914720671) was to remove a test that asserted the ordering of cosigners, as this behavior was being reconsidered in the project. I was also encouraged to add an assertion verifying the parent fingerprint, strengthening the test's ability to detect incorrect key information during export. After incorporating the feedback, the PR was merged.

## What I Would Do Differently

Looking back, one of the biggest things I would change is how I approached identifying contribution opportunities. I initially underestimated the time required to become familiar with an established codebase and understand where meaningful improvements could be made. I spent considerable time exploring different areas without always having a clear direction.

If I were to approach the fellowship again, I would narrow my focus earlier and spend more time understanding the architecture, existing tests, and development conventions of a particular project before deciding what to implement.

I would also engage with maintainers earlier. My experience with Sparrow Wallet demonstrated how valuable their guidance can be. What I initially believed would be useful test coverage turned out to overlap with tests already implemented in the underlying Drongo library. That feedback ultimately helped me identify a more meaningful contribution.

Perhaps most importantly, I would approach the learning process with more patience. Throughout the fellowship, I sometimes measured my progress by how much code I had written or how many tasks I had completed. However, I came to realize that reading code, researching unfamiliar concepts, asking questions, reviewing pull requests and investigating potential solutions are all essential parts of the development process.
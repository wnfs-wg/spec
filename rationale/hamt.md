# Need

For the private WNFS we need a data structure that efficiently encodes a hash-map in IPLD.
We're optimizing for
- byte diffs that need to be transferred to sync two separate versions of a HAMT and
- key/value pair lookup performance.

We are *not* optimizing for bitswap sync round-trips. We recommend other methods of syncing huge HAMTs than over bitswap, for example by sending CAR files containing new blocks in a single round trip.

Since we're looking for efficient IPLD encodings and content-addressed data is inherently immutable, we're looking for efficient immutable data structures.


# Choice

We've chosen a 16-degree HAMT (hash array mapped trie) with commutative different-key-insertion semantics: In the original HAMT implementation, key removal was implemented as setting the value to a null pointer. Instead, we remove the key/value pair from the array. This means that for a certain set of key/value pairs, the root hash of our HAMT is independent from the order of insertions of these key/value pairs.


# Alternatives

## HAMTs with different Node Degrees

We've both considered a binary merkle tree (degree 2 HAMT), since they have byte-minimal merkle proofs as well as very high degree HAMTs such as 1024 or 256-degree HAMTs, since they minimize the internal CID-link byte overheads.

We've experimentally evaluated various degrees. This chart shows the trade-off between total byte storage overhead and merkle proof size clearly:

![total bytes vs. proof bytes](/images/hamt_total_bytes_vs_proof_bytes.png)

Although binary trees have the smallest merkle proofs (red bar on the very left), they have the highest total byte overhead (blue bar on the very left).
Merkle proof size is also almost exactly the byte diff needed to transfer to sync a single-key update.

On the other hand a 256-degree HAMT has huge merkle proof byte overheads (red bar on the very right), but small total byte overheads (blue bar on the very right).

Degree 16 seems to be a compromise on both of these dimensions. Their merkle proofs are less than twice as big as binary tree diffs and their internal byte node overhead seems only slightly above high-degree HAMTs.

If you look at the growth curves on bytes needed to transfer for changing `n` of 100,000 keys, you can see that the 16-degree HAMT has reasonable growth compared to other options:

![diff sizes](/images/hamt_diff_sizes.png)


## Modified Merkle Patricia Tree (MMPT)

The MMPT is known for being the data structure that the Ethereum state tree is based on.
We found it be very similar to a HAMT: The actual topology of the tree seems to be almost the same for the same set of keys. They're both prefix tries.

However, for degree 16, the HAMT has a more byte-efficient encoding for full internal nodes, which will usually be about half of all total tree nodes, as it doesn't store the prefix for each key, but instead only an array of links with a bitmap that encodes the populated prefixes.


## [Merkle Search Tree (MST)](https://hal.inria.fr/hal-02303490/document)/[Pattern-Oriented-Split Tree (POS-Tree)](https://arxiv.org/abs/2003.02090)

There exist various lookup-efficient, immutable tree data structures. We took a look at Merkle Search Trees and Patter-Oriented-Split Trees.

What makes these trees useful is their ability to handle non-uniform key distributions. This doesn't apply to our use-case in private WNFS, because we're using `NameAccumulator` hashes as the HAMT keys.


## [Splay Trees](https://en.wikipedia.org/wiki/Splay_tree)

Splay trees have the unique property that recently-inserted items can be accessed faster than older items. While it's true that recently inserted private WNFS blocks are more likely to be read, we also want to hide the fact that these items were written more recently in the HAMT to prevent leaking metadata.

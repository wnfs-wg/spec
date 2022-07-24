# Need

We need a cryptographic accumulator to prove to others we've got signature-attested write access to a set of keys in the private file system, without revealing the hierarchy of these keys.


# Choice

We've chosen a [Generalized Combinatoric Accumulator](https://www.jstage.jst.go.jp/article/transinf/E91.D/5/E91.D_5_1489/_pdf/-char/en) (GCA), which is a tunable version of [Nyberg's hash accumulator](https://link.springer.com/content/pdf/10.1007%2F3-540-60865-6_45.pdf), which is in turn a [Bloom filter](https://en.wikipedia.org/wiki/Bloom_filter).

The underlying bloom filter is a 2048-bit bloom filter with 30 hash functions consisting of XXHash3 with seeds ranging from 0-29. This bloom filter is tuned to be used for 47 elements, averaging out at a popcount of 1019 with 47 elements.

# Alternatives

## Parameters

We've considered other bloom filter sizes. We want to have the bloom filter bit-size be a power-of-two, as that allows easily creating unbiased hash indices.

We arbitrarily deemed 1024-bit bloom filters as too small, as they'd only support half as many path segments.

Every private file system block will be accompanied by a namefilter, so we don't want namefilters to be too big, in order to reduce time-to-sync.


## Other Design Considerations

GCAs were chosen over other [arguably more sophisticated](https://www.fim.uni-passau.de/fileadmin/dokumente/fakultaeten/fim/forschung/mip-berichte/MIP_1210.pdf) options for three main reasons: witness side, raw performance, and ease of implementation for web browsers. For example, we were unable to find a widely-used RSA accumulator library on NPM or Crates, but implementing a GCA is very straightforward.

Due to distinguishability, GCAs potentially leak some information about related, deeply-nested sibling nodes as the Hamming approaches zero. Our namefilter GCAs have a cardinality of 47 by default, which is much deeper than most file paths.

We considered using XOR or Cuckoo filters instead of class Bloom filters. XOR is very close to the theoretic efficiency limit, but is very new and the library untested. Cuckoo filters would provide around an additional 4 path segments with the same false-positive rate, but we lose the single-bit-collision of Bloom filters which is actually an advantage for obfuscation.


## Zero-Knowledge Succinct, Non-Interactive Arguments of Knowledge (zkSNARKs)

(Or zkSTARKs, Bulletproofs, etc.)

We've briefly considered zkSNARKs, but initially decided against them due to longer proof times (state 2021). In the [private WNFS](/private-wnfs.md) a client may have to generate a proof that they're allowed to write to a private node under a certain key for every directory or file they're modifying. Proof times of above 1 second on mobile devices are thus impractical.

Another reason for deciding against zkSNARKs is a lack of expertise. However, given the speed of advancement in zkSNARK technology recently, the above paragraph may be outdated already! If *you* have expertise in working with zkSNARKs, please contact us!

# Need

We need a cryptographic accumulator to prove to others we've got signature-attested write access to a set of keys in the private file system, without revealing the hierarchy of these keys.

# Choice

We've chosen 2048-bit RSA accumulators with a per-WNFS trusted setup by the root owner on creation, 256-bit inumbers and proofs `PoKE*` and `PoKCR` described [in this paper][IOP Batching Boneh] for verifying membership.

# Alternatives

## Generalized Combinatoric Accumulators (Namefilters)

[Generalized Combinatoric Accumulators][GCAs] (GCAs) are a tunable version of [Nyberg's hash accumulator][Nyberg Accumulators], which is in turn a [Bloom filter][Wikipedia Bloom Filters].

A previous version of this specification used a 2048-bit bloom filter with 30 hash functions consisting of XXHash3 with seeds ranging from 0-29. This bloom filter was tuned to be used for 47 elements, averaging out at a popcount of 1019 with 47 elements.

The downside of bloom filters over RSA accumulators is that you have an element limit, and providing a proof for a certain element will both reveal the element and reveal any other bloom filters it's contained in.

## Zero-Knowledge Succinct, Non-Interactive Arguments of Knowledge (zkSNARKs)

(Or zkSTARKs, Bulletproofs, etc.)

We've briefly considered zkSNARKs, but initially decided against them due to longer proof times (state 2021). In the [private WNFS][Private WNFS] a client may have to generate a proof that they're allowed to write to a private node under a certain key for every directory or file they're modifying. Proof times of above 1 second on mobile devices are thus impractical.

Another reason for deciding against zkSNARKs is a lack of expertise. However, given the speed of advancement in zkSNARK technology recently, the above paragraph may be outdated already! If *you* have expertise in working with zkSNARKs, please contact us!

[IOP Batching Boneh]: https://eprint.iacr.org/2018/1188.pdf
[GCAs]: https://www.jstage.jst.go.jp/article/transinf/E91.D/5/E91.D_5_1489/_pdf/-char/en
[Nyberg Accumulators]: https://link.springer.com/content/pdf/10.1007%2F3-540-60865-6_45.pdf
[Wikipedia Bloom Filters]: https://en.wikipedia.org/wiki/Bloom_filter
[Private WNFS]: /spec/private-wnfs.md

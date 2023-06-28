# Name Accumulator Specification

Name accumulators in WNFS represent commitments to "paths" in the private file system in a way that allows a third party to verify that certain path segments are part of a committed path without revealing all path segments and without leaking correlatable information to other committed paths.

They are built on 2048-bit RSA accumulators. They were originally described in 1993 in [this paper][RSA acc og paper].

The element membership proofs in this specification are based on the 2018 paper ["Batching Techniques for Accumulators with Applications to IOPs and Stateless Blockchains"][batching acc paper].

## 1 Encoding

### 1.1 Setup Parameters

The accumulator setup consists of a 2048 bit RSA modulus $N$ and a quadratic residue generator $g$.

They are encoded as 256 byte big-endian byte arrays.

### 1.2 Commitments

All name accumulator commitments are 2048 bit unsigned integers smaller than the RSA modulus from the setup parameters.

They are encoded as 256 byte big-endian byte arrays.

### 1.3 Elements

Name accumulator elements are essentially "path segments", but not represented as strings. Instead, each element is assigned a 256-bit random prime number.

They are encoded as 32 byte big-endian byte arrays.

### 1.4 Proofs

Proofs and witnesses of relationships between name accumulators come in various forms. Components in these proofs consist of elements summarized in the table below:

| Proof component                        | Representation |
|----------------------------------------|----------------|
| Element in the quadratic residue group | 256 byte big-endian byte array |
| Prime hash $l$                         | 16 byte big-endian byte array and big-endian unsigned varint counter |
| Residue $r$                            | 16 byte big-endian byte array |

Collecting these proof components into their own structures is up to implementations for now. In WNFS, the residue $r$ is stored alongside each entry in the private forest, but the prime hash $l$ is computed on the fly and proof witnesses are only found in protocols that combine WNFS with certificates. A future version of this specification may describe such protocols.

## 2 Algorithms

---

TODO

The concrete protocol we use is based on the "proof of knowledge of exponent" (PoKE*) from [this paper](https://eprint.iacr.org/2018/1188.pdf), made non-interactive via the fiat-shamir heuristic. The resulting protocol generates a 128-bit prime $l$ by hashing the path that the current node has access to and the path that they delegate further.

Again extending the example from before, say an peer that has gotten delegated write access to `/Docs/University/` further delegates write access to the file `/Docs/University/Notes.md` to yet another peer.

To do this, they'd generate a certificate containing
- that peer's public key
- their own certificate (or a hash thereof)
- the name accumulator $u_1$ `rsa_accumulator([inum(docs), inum(uni), inum(notes)])`
- the proof $Q = u_0^q$ where $qx + r = l$ where $l = \mathsf{Hash}_\mathsf{Prime}(u_0, u_1)$ where $u_0 = g^{\mathbb{inum}(\mathbb{docs}) \cdot \mathbb{inum}(\mathbb{uni})}\ mod\ N$
- the residue $r$.

A verify would then verify this delegation by checking $Q^l u_0^r = u_1$.



TODO

They will thus accumulate all writes they did by multiplying the $Q_i$ they generate together by taking their product.

For each key they write to the forest, they also include the residue $r$ from the $PoKE*$ in the leaf as well as a 4-byte indicator $l_{inc}$ that helps the verifier to compute the prime hash $l$ faster.

They then finally provide the accumulated $Q = \prod_{i = 0}^n{Q_i}$ as well as the certificate chain and the current and previous private forest hash in a certificate proving write access.

[RSA acc og paper]: https://link.springer.com/content/pdf/10.1007/3-540-48285-7_24.pdf
[batching acc paper]: https://eprint.iacr.org/2018/1188.pdf



TODO specify wnfs/nameaccum/revisions/segment for domain separation of revision part of name accumulator

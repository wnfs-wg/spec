# Name Accumulator Specification

Name accumulators in WNFS represent commitments to "paths" in the private file system in a way that allows a third party to verify that certain path segments are part of a committed path without revealing all path segments and without leaking correlatable information to other committed paths.

They are built on 2048-bit RSA accumulators. They were originally described in 1993 in the paper ["One-way Accumulators: A Decentralized Alternative to Digital Signatures"][RSA acc og paper].

The element membership proofs in this specification are based on the 2018 paper ["Batching Techniques for Accumulators with Applications to IOPs and Stateless Blockchains"][IOP Batching Boneh].

## 1 Encoding

### 1.1 Setup Parameters

The accumulator setup consists of a 2048 bit RSA modulus $N$ and a quadratic residue generator $g$. The generator element represents the empty name accumulator.

They are encoded as 256 byte big-endian byte arrays.

### 1.2 Commitments

All name accumulator commitments (`NameAccumulator`s) are 2048 bit unsigned integers smaller than the RSA modulus from the setup parameters.

They are encoded as 256 byte big-endian byte arrays.

### 1.3 Elements

Name accumulator elements are essentially "path segments", but not represented as strings. Instead, each element is assigned a 256-bit random prime number.

They are encoded as 32 byte big-endian byte arrays.

### 1.4 Proofs

Proofs and witnesses of relationships between name accumulators come in various forms. Components in these proofs consist of elements summarized in the table below:

| Proof component                        | Representation |
|----------------------------------------|----------------|
| Element in the quadratic residue group | 256 byte big-endian byte array |
| Prime hash $l$                         | 16 byte little-endian byte array and [unsigned varint] counter |
| Residue $r$                            | 16 byte little-endian byte array |

Collecting these proof components into their own structures is up to implementations. Future versions of this specification may describe an encoding.

## 2 Algorithms

### 2.1 Accumulation

`add_element : (SetupParameters, NameAccumulator, Element) -> NameAccumulator`

To add an element, exponentiate the current state of the name accumulator to the element added and take the setup's modulo. With the current accumulator state denoted as $u$, the element denoted as $e$ and the RSA modulus denoted as $N$, the new accumulator state $u'$ is

$$u' = u^e \mod\ N$$

### 2.2 Empty Accumulator

`empty : SetupParameters -> NameAccumulator`

The empty accumulator can simply be read out of the setup parameters, it's the generator element $g$.

### 2.3 Hashing to Prime

`hashToPrime : (String, ByteArray, Uint) -> (BigUint, Uint)`

Multiple parts of the name accumulator proof protocols need the capacity to hash a byte array to a prime number.

The input to this function is a byte array to hash as well as the desired output length $n <= 32$ in bytes.
The output is a number $h$ with $1 <= h < 2^n$ and a 32-bit unsigned integer counter that can be used to verify a hash produced by `hashToPrime` in constant time.

```ts
function hashToPrime(
  domainSeparationString: String,
  toDigest: Uint8Array,
  outputLength: Uint
): [Uint8Array, Uint] {
  let counter = 0
  let primeCandidate

  do {
    const counterEncoded = encode32BitBigEndian(counter)
    const hash = blake3.deriveKey(domainSeparationString, concat([toDigest, counterEncoded]))
    const truncatedHash = new Uint8Array(hash.buffer, 0, outputLength)
    primeCandidate = decodeBigUintBigEndian(truncatedHash)
  } while (!isPrime(primeCandidate))

  return [primeCandidate, counter]
}
```

### 2.4 Batched Elements Proof

`batchProofElements : (SetupParameters, NameAccumulator, Array<Element>) -> (NameAccumulator, ElementsProof)`

This algorithm updates an accumulator state by adding an arbitrary number of prime-sized elements to it and produces a succinct proof that verifies that the proving party knows a set of elements to go from the past state of the accumulator to the current.

The algorithm for this proof is the "proof of knowledge of exponent" (PoKE*) from [this paper][IOP Batching Boneh], made non-interactive via the Fiat-Shamir heuristic.

The number $l$ from the protocol is derived via SHA-3-hashing the big-endian bytes of the modulus, base, commitment and a counter. The base is the accumulator state before the elements were added and the commitment is the state after.
The counter is the lowest 32-bit number that makes the first 128-bit truncated hash prime, if interpreted as a little-endian unsigned 128-bit integer.

```ts
function deriveLHash(
  modulusBigEndian: Uint8Array,
  baseBigEndian: Uint8Array,
  commitmentBigEndian: Uint8Array
): Uint8Array {
  return hashToPrime(
    "wnfs/PoKE*/l 128-bit hash derivation",
    concat([
      modulusBigEndian,
      baseBigEndian,
      commitmentBigEndian,
    ]),
    16
  )
}
```

When generating the proof, we additionally output the counter that was used for making the truncated hash prime. Although this counter is not necessary for verification, it's useful by making it constant-time.

This method of hashing to a prime number is described in Section 7 of [this paper][IOP Batching Boneh].

The final `ElementsProof` output consists of the residue $r$, $l_{nonce}$ hash counter and the number $Q$. They are derived as described in the PoKE* protocol from section 3.2 of [this paper][IOP Batching Boneh].

### 2.5 Multi-Batch Elements Proof

`multiBatchElementsProof : (SetupParameters, Array<(NameAccumulator, NameAccumulator, ElementsProof)>) -> MultiBatchProof`

This algorithm batches multiple batch elements proofs together. It combines the PoKCR (Proof of Knowledge of Co-prime Roots) from Section 3.3 of [this paper][IOP Batching Boneh] with the previously described PoKE* (see the [Batched Elements Proof algorithm]).

The resulting proof is not fully succinct. It only manages to compress the number $Q$ from each `ElementsProof`, but still requires that you keep around all residues $r$ and $l_{nonce}$ hash counters as well as which accumulator states they are related to.

The output consists of $Q^*$, the product of all numbers $Q$ from the `ElementsProof`s as well as all residues $r$ and all $l_{nonce}$ hash counters, mapped to the accumulator states they prove.

### 2.6 Multi-Batch Elements Proof Verification

`verifyMultiBatchElementsProof : (SetupParameters, MultiBatchProof) -> Boolean`

This algorithm verifies the multi-batch proof's mapping of accumulator states. The multi-batch proof contains a single accumulator value $Q$ as well as a mapping of accumulator before to after states together with a residue $r$ and hash counter $l_{nonce}$.

It then does the following steps:
1. Initialize an empty array `basesAndExponents`.
2. Go through each mapping of accumulator states, for each:
   1. Compute the $l$ hash by using `deriveLHash` from the [Batched Elements Proof algorithm]. It's RECOMMENDED to speed the prime hash computation up by making use of the given hash counter $l_{nonce}$.
   2. Verify that $l < r$. If not, return `false`.
   3. Given the accumulator before state $b$ and after state $c$, compute $c * b^{-r}\ mod\ N$.
   4. Push this result together with the $l$ hash as a tuple into `basesAndExponents`.
3. Compute $l^*$ as the product of all $l$ hashes in `basesAndExponents`.
4. Verify that $Q^{l^*} \equiv \mathtt{multiExp}(\mathtt{basesAndExponents})\ mod\ N$, where `multiExp` is multi-exponentiation as described in [section 3.3 of this paper][IOP Batching Boneh]. Otherwise return `false`.
5. Return `true`.

[RSA acc og paper]: https://link.springer.com/content/pdf/10.1007/3-540-48285-7_24.pdf
[IOP Batching Boneh]: https://eprint.iacr.org/2018/1188.pdf
[unsigned varint]: https://github.com/multiformats/unsigned-varint

[Batched Elements Proof algorithm]: #24-Batched-Elements-Proof

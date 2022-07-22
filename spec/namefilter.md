# Namefilters

A namefilter in WNFS is a 256 bytes (2048 bits) bloom filter. Its bloom filter parameters are tuned for 47 elements resulting in an optimal 30 hash functions used.

Formally the construction we are using is known a *symmetric cryptographic accumulator* or a *Nyberg accumulator*.

The namefilter is used as an obfuscated naming scheme for private blocks. You should think of it similar to the name of a public directory or file consisting of all the directory or file's ancestors and the name of the file or directory itself. For example, the file path "/public/Documents/thesis.pdf" contains its own name "thesis.pdf" and its ancestor's names "public" and "Documents".
Similarily, namefilters are bloom filter that contain the identity numbers (`inumber`s) of what they're addressing and their ancestors.
Given only such a namefilter, it's impossible to extract the original `inumber`s in the path of that namefilter.
However, given a single `inumber` that is contained in the namefilter, it's possible to verify its membership in the namefilter (with a false positive rate of 1 in a billion) without revealing information about any other members of that namefilter.
This is useful for proving that you have right access to a particular file or directory and all of its decendants to a third party that doesn't need to be able to read the private files that are written to.

## Initialization

A namefilter starts out as an empty bloom filter, i.e. a byte string of length 256 with all bytes zeroed out.

Such a bloom filter is considered a "bare" namefilter until it is saturated, i.e. until at least 1019 bits are set to one.

Only bare namefilters can have items added to them. Once a namefilter is saturated, it must not have items added to it.

Bare namefilters should be treated as confidential.

If a bare namefilter is not saturated after processing, it must undergo a saturation operation before being shared publicly.

## Operation `add`

A bare namefilter can have an item added to it by setting 30 indices of its internal bitarray to one.
These 30 indices are computed like this:

```ts
function indexFor(element: ByteArray, n: uint64): uint64 {
  return xxhash3_64withSeed(sha3(element), n)
}
```

## Operation `has`

A namefilter can be queried for elements it may contain by going through the indices for an element as described in [Operation `add`](#Operation-add) and testing that every bit at these indices has been set.

If that is the case, then the element is part of the namefilter, except for false positives that should occur for less than 1 in a billion elements on saturated namefilters.

If that is not the case, then the element is definitely not part of the namefilter.


## Operation `saturate`

```ts
let namefilter: ByteArray<256> = // ...
let index = 0
let xof: (index: uint64) => ByteArray<32> = shake256XOF(namefilter)

while (true) {
  // addBare is a version of 'add' that skips the 'sha3' step on its input.
  const nextNamefilter = addBare(namefilter, xof(index))

  if (countOnes(nextNamefilter) > 1019) {
    return namefilter
  }

  namefilter = nextNamefilter
  index++
}
```

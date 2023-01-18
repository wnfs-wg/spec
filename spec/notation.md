
# Notation

We're using typescript-similar type annotations as notation for describing WNFS data encoding. These type annotations are not supposed to be interpreted "verbatim" as they include some unconventional ways of conveying encoding. These special encoding markers meaning's are listed in this section.

## `Cbor<Data>`

This marks that the data type that it wraps is supposed to be encoded as a DAG-CBOR block and always decodes to a value of type `Data`.

For example: `Cbor<{ name: string; age: Int }>` is supposed to be a DAG-CBOR-encoded map with "name" and "age" fields of corresponding types.

You can think of the *actual* representation of `Cbor` as being a `Uint8Array`/basic byte string, with a phantom type parameter marking what type of data this byte string should decode to.

## `Cid<Block>`

This means at this point in the data structure there's a single IPLD CID addressing data that decodes to the provided phantom type parameter `Block`.

The CID's multicodec is meant to be inferred automatically from its type parameter. For example, `Cid<Cbor<...>>` will always be a CID marked as `dag-cbor`, and `Cid<Encrypted<...>>` will always be a CID marked as `raw`.

The *actual* representation of `Cid` is meant to be a byte representation for CIDs, and depends on the context. E.g. if `Cid` is found somewhere within a `Cbor<>` block, then it uses the DAG-CBOR-native way of encoding CIDs as CBOR objects with a `42` tag and a binary representation of a CID.

## `AesGcm<Data>`

This represents a ciphertext that's marked with the data type `Data` that it needs to decrypt to. For example `AesGcm<Ratchet>` represents a byte string that would decrypt to a value of type `Ratchet` using [AES-GCM](https://csrc.nist.gov/publications/detail/sp/800-38d/final) with 256-bit keys.
The ciphertexts produced by `AesGcm<>` have a 12 byte initialization vector prepended and the 16 byte authentication tag appended.

## `AesKw<Data>` and `AesKwp<Data>`

This represents a ciphertext that decrypts to given data type `Data`.
The encryption/decryption algorithm used is [AES-KW](https://csrc.nist.gov/publications/detail/sp/800-38f/final) with 256-bit keys.
AES-KW and AES-KWP are can be thought of as keyed permutation functions where observing the permutation doesn't reveal information about the key used. It doesn't provide [IND-CCA2](https://en.wikipedia.org/wiki/Ciphertext_indistinguishability), so must only be only be used in cases where detecting two same-key and same-message ciphertexts is not a security concern.

In contrast to AES-KW, AES-KWP allows encrypting messages that are not multiples of 8 bytes by adding padding.

In this specification we use AES-KW for encrypting random AES keys and AES-KWP for encrypting private node headers containing the inumber, namefilter and skip ratchet.

## `ByteArray<length>`

This represents a `ByteArray` of `length` bytes.

If `length` isn't provided, this represents a `ByteArray` that additionally stores its length.

## `Hash<Data>`

This means at this point in the data structure there's a 32-byte array representing the SHA3 hash of `Data`.

## Algorithm Type Signatures

For algorithms, we often specify their type signature, e.g. `: (Namefilter, RevisionKey) -> Namefilter`. This is read as "has the type signature 'function of namefilter and revision key to namefilter'".

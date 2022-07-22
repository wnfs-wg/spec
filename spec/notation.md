
# Notation

We're using typescript-similar type annotations as notation for describing WNFS data encoding. These type annotations are not supposed to be interpreted "verbatim" as they include some unconventional ways of conveying encoding. These special encoding markers meaning's are listed in this section.

## `CBOR<Data>`

This marks that the data type that it wraps is supposed to be encoded as a DAG-CBOR block and always decodes to a value of type `Data`.

For example: `CBOR<{ name: string; age: Int }>` is supposed to be a DAG-CBOR-encoded map with "name" and "age" fields of corresponding types.

You can think of the *actual* representation of `CBOR` as being a `Uint8Array`/basic byte string, with a phantom type parameter marking what type of data this byte string should decode to.

## `CID<Block>`

This means at this point in the data structure there's a single IPLD CID adressing data that decodes to the provided phantom type parameter `Block`.

The CID's multicodec is meant to be inferred automatically from its type parameter. For example, `CID<CBOR<...>>` will always be a CID marked as `dag-cbor`, and `CID<Encrypted<...>>` will always be a CID marked as `raw`.

The *actual* representation of `CID` is meant to be a byte representation for CIDs, and depends on the context. E.g. if `CID` is found somewhere within a `CBOR<>` block, then it uses the DAG-CBOR-native way of encoding CIDs as CBOR objects with a `42` tag and a binary representation of a CID.

## `Encrypted<Data>`

This represents a ciphertext that's marked with the data type `Data` that it needs to decrypt to. For example `Encrypted<Ratchet>` represents a byte string that would decrypt to a value of type `Ratchet`.

## `ByteArray<length>`

This represents a `ByteArray` of `length` bytes.

If `length` isn't provided, this represents a `ByteArray` that additionally stores its length.

## `Hash<Data>`

This means at this point in the data structure there's a 32-byte array representing the SHA3 hash of `Data`.

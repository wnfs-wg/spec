# Private WNFS

It makes sense to split the private partition into two layers:
- The "decrypted" layer that defines the type of data you can decrypt given the correct keys. Links between blocks in this layer are references in the HAMT data structure at the "encrypted" layer.
- The "encrypted" layer that defines how all of the encrypted data blocks are organized as IPLD data. Links in this layer are CID-links.

## Key Structure

![hierarchical key structure](/images/hierarchical_key_structure.png)

![key structure](/images/key_structure.png)

## The Encrypted Layer

The private partition consists of lots of private data blocks encrypted with different keys. These blocks are supposed to be of size smaller than 256 kilobytes in order to comply with block size restrictions from IPFS. However, keeping block size small is also useful for reducing metadata leakage - it's less obvious what the file size distribution in the private file system is like, if these files are split into blocks.

These blocks of encrypted data are put into a HAMT that encodes a multi-valued hash-map. The HAMT has a node-degree of 16. See [`rationale/hamt.md`](/rationale/hamt.md) for more information about that.

The keys in the HAMT are saturated [namefilter](/namefilter.md)s.

SHA3 hashes of namefilters are the linking scheme used in the decrypted layer.


```typescript
type PrivateRoot =
  CBOR<HAMT<
    // Map Key: The namefilter ("name") for that private node
    Namefilter,
    // Map Value: A tuple of an encrypted private node header and a set of links to encrypted node contents
    [ByteArray, Array<CID<ByteArray>>]
  >>

type HAMT<K, V> = {
  structure: "hamt"
  version: "0.1.0"
  root: Node<K, V>
}

type Node<K, V> =
  [ ByteArray<2> // bitmask
  , Array<Entry<K, V>> // Entries
  ]

type Entry<K, V>
  = CID<CBOR<Node<K, V>>>
  | Array<[K, V]> // bucket of values
```

## The Decrypted Layer

The private WNFS borrows the same metadata structure as the [public WNFS](/public-wnfs.md#metadata).

Encryption keys are derived from a [skip ratchet](/skip.ratchet.md).

```typescript
type Namefilter = ByteArray<256>
type Key = ByteArray<32>
type Inumber = ByteArray<32>

type PrivateNode
  = PrivateDirectory
  | PrivateFile

// encrypted using deriveKey(ratchet)
type PrivateNodeHeader = {
  ratchet: SkipRatchet
  bareName: Namefilter
  inumber: Inumber // TODO(matheus23): Inside or outside the revision section?
}

type PrivateDirectory = {
  metadata: Metadata<"wnfs-private-dir">
  entries: Record<string, {
    contentKey: Key // hash(deriveKey(entryRatchet))
    revisionKey: Encrypted<Key> // encrypt(deriveKey(ratchet), deriveKey)(entryRatchet))
    name: Hash<Namefilter> // hash(saturated(entryBareName))
    // and can be used as the key in the private partition HAMT to lookup
    // a (set of) PrivateNode(s) with an entryBareName and entryRatchet from above
  }>
}

type PrivateFile = {
  metadata: Metadata<true>
  content: ByteArray | {
    ratchet: SkipRatchet
    count: Uint64
  }
}
```


Example:

![block encryption example](/images/encrypted_blocks.png)


## Algorithms

### File Content Access

Private file content may be inlined or externalized. Inlined content is decrypted along with the header.

Since external content is separate from the header, it needs a unique namefilter derived from a ratchet (to avoid forcing lookups to go through the header). If the key were derived from the header's key, then the file would be re-encrypted e.g. every time the metadata changed.

External content namefilters are defined thus:

```typescript
const segmentNames = (file) => {
  const { bareNamefilter, content: { ratchet, count } } = file.header
  const key = ratchet.toBytes()
  
  let names = []
  for (i = 0; i < count; i++) {  
    names[i] = bareNamefilter
                 .addBare(sha3(key))
                 .addBare(sha3(`${key}${i}`))
                 .saturate()
  }

  return contentNames
}
```


### Merge


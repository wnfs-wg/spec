
# Private WNFS

It makes sense to split the private partition into two layers:
- The "decrypted" layer that defines the type of data you can decrypt given the correct keys. Links between blocks in this layer are references in the HAMT data structure at the "encrypted" layer.
- The "encrypted" layer that defines how all of the encrypted data blocks are organized as IPLD data. Links in this layer are CID-links.

## The Encrypted Layer

The private partition consists of lots of private data blocks encrypted with different keys. These blocks are supposed to be of size smaller than 256 kBs in order to comply with block size restrictions from IPFS. However, keeping block size small is also useful for reducing metadata leakage - it's less obvious what the file size distribution in the private file system is like, if these files are split into blocks.

These blocks of encrypted data are put into a HAMT that encodes a multi-valued hash-map. 

The keys in the HAMT are saturated [namefilter](/namefilter.md)s.

SHA3 hashes of namefilters are the linking scheme used in the decrypted layer.


```typescript
type PrivateRoot =
  CBOR<HAMT<
    Namefilter, // Map Key: The namefilter ("name") for that private node
    Array<CID<ByteArray>> // Map Value: A set of links to encrypted blocks of data
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

```typescript
type Namefilter = ByteArray<256>
type Key = ByteArray<32>
type Inumber = ByteArray<32>

type SkipRatchet = {
  large: Key
  mediumCount: Uint8
  medium: Key
  smallCount: Uint8
  small: Key
}

type PrivateNode
  = PrivateDirectory
  | PrivateFile

type PrivateDirectory = {
  // encrypted using deriveKey(ratchet)
  revision: Encrypted<CBOR<{
    ratchet: SkipRatchet
    bareName: Namefilter
    inumber: Inumber // TODO(matheus23): Inside or outside the revision section?
  }>>
  metadata: Metadata<false>
  entries: Record<string, {
    snapshotKey: Key // hash(deriveKey(entryRatchet))
    revisionKey: Encrypted<Key> // encrypt(deriveKey(ratchet), deriveKey(entryRatchet))
    name: Hash<Namefilter> // hash(saturated(entryBareName))
    // and can be used as the key in the private parition HAMT to lookup
    // a (set of) PrivateNode(s) with an entryBareName and entryRatchet from above
  }>
}

type PrivateFile = {
  // encrypted using deriveKey(ratchet)
  revision: Encrypted<CBOR<{
    ratchet: SkipRatchet
    bareName: Namefilter
    inumber: Inumber
  }>>
  metadata: Metadata<true>
  content: ByteArray // TODO(matheus23) non-inline files
}
```
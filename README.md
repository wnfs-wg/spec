# Webnative File System (WNFS) Specification v0.2.0

# Abstract

The Web Native File System (WNFS) is a file system written for the web. It is versioned, logged, programmable, has strong-yet-flexible security, and is fully controlled by the end user. Service providers can validate writes without reading the contents of the file system, and minimal metadata is leaked.

WNFS relies heavily on the ”space” side of the space/time tradeoff to deliver performance. As a consequence of this and Merklization, WNFS can provide advanced functionality such as versioning and delegated or collaborative write access.

WNFS can be used for offline collaboration, because WNFS forms a state-based conflict-free replicated data type (CRDT): There exists a commutative and associative merge function that combines any two (versions of) WNFS roots.

WNFS is content-addressed and thus extremely portable. It may be stored on the edge, on the end user's device or in the cloud. Devices may also only partially load a WNFS and still write files and directories.

# Data Structure

## Notation

We're using typescript-similar type annotations as notation for describing WNFS data encoding. These type annotations are not supposed to be interpreted "verbatim" as they include some unconventional ways of conveying encoding. These special encoding markers meaning's are listed in this section.

### `CBOR<Data>`

This marks that the data type that it wraps is supposed to be encoded as a DAG-CBOR block and always decodes to a value of type `Data`.

For example: `CBOR<{ name: string; age: Int }>` is supposed to be a DAG-CBOR-encoded map with "name" and "age" fields of corresponding types.

You can think of the *actual* representation of `CBOR` as being a `Uint8Array`/basic byte string, with a phantom type parameter marking what type of data this byte string should decode to.

### `CID<Block>`

This means at this point in the data structure there's a single IPLD CID adressing data that decodes to the provided phantom type parameter `Block`.

The CID's multicodec is meant to be inferred automatically from its type parameter. For example, `CID<CBOR<...>>` will always be a CID marked as `dag-cbor`, and `CID<Encrypted<...>>` will always be a CID marked as `raw`.

The *actual* representation of `CID` is meant to be a byte representation for CIDs, and depends on the context. E.g. if `CID` is found somewhere within a `CBOR<>` block, then it uses the DAG-CBOR-native way of encoding CIDs as CBOR objects with a `42` tag and a binary representation of a CID.

### `Encrypted<key, Data>`

This represents a ciphertext that's marked with the data type `Data` that it needs to decrypt to, if provided with the correct key `key`. The `key` parameter is supposed to be a value-level expression. For example `Encrypted<sha3(derive_key(ratchet)), ratchet>` represents a byte string that would decrypt to the value `ratchet` given the key computed by `sha3(derive_key(ratchet))`.

### `ByteArray<length>`

This represents a `ByteArray` of `length` bytes.

### `hash()`

### `deriveKey()`

## WNFS Root

The WNFS root is a [UnixFS](https://github.com/ipfs/specs/blob/main/UNIXFS.md) directory with links to WNFS's partitions:

- `public`: A link to the root directory of all public data
- `private`: A link to all private data blocks organized in a HAMT
- `pretty`: A reduction of the public partition into UnixFS to make it easy to serve over IPFS http gateways.

It also includes a single UnixFS file at the link `version` that encodes in UTF8 the WNFS version `0.2.0`.


```typescript
type DataRoot = CID<UnixFSDirectory<Root>>

type Root = {
  version: "0.2.0"
  public: CID<PublicRoot>
  private: CID<PrivateRoot>
}
```


## `public` Partition

There are three types of nodes in the public partition:
- Directory Nodes
- File Nodes
- Symlink Nodes

Nodes need to be encoded as dag-cbor.

A directory node contains a map of entry names to CIDs that resolve to either more directory nodes, file nodes or symlink nodes.

The root node of the public partition must be a directory node.

```typescript
type PublicRoot = CBOR<PublicDirectory>

type PublicNode
  = PublicDirectory
  | PublicFile

type PublicDirectory = {
  previous?: CID<CBOR<PublicDirectory>>
  merge?: CID<CBOR<PublicDirectory>>
  metadata: Metadata<false> // must always mark this as "not a file"
  entries: Record<string, CID<CBOR<PublicNode>> | PublicSymlink>
}

type PublicSymlink = {
  ipns: string // e.g. alice.files.fission.name/public/ABCD
}

type PublicFile = {
  previous?: CID<CBOR<PublicFile>>
  merge?: CID<CBOR<PublicFile>>
  metadata: Metadata<true> // must always mark this as a file
  content: CID<CBOR<IPFSUnixFSFile>>
}

type Metadata<IsFile> = {
  version: "0.2.0"
  isFile: IsFile
  unixMeta: UnixMeta
}

type UnixMeta = {
  mtime: Int
  ctime: Int
  mode: Int // POSIX file mode, e.g. (octal) 666 or 777
}
```

## `private` Partition

It makes sense to split the private partition into two layers:
- The "decrypted" layer that defines the type of data you can decrypt given the correct keys. Links between blocks in this layer are references in the HAMT data structure at the "encrypted" layer.
- The "encrypted" layer that defines how all of the encrypted data blocks are organized as IPLD data. Links in this layer are CID-links.

### The Encrypted Layer

The private partition consists of lots of private data blocks encrypted with different keys. These blocks are supposed to be of size smaller than 256 kBs in order to comply with block size restrictions from IPFS. However, keeping block size small is also useful for reducing metadata leakage - it's less obvious what the file size distribution in the private file system is like, if these files are split into blocks.

These blocks of encrypted data are put into a HAMT that encodes a multi-valued hash-map. 

The keys in the HAMT are namefilters.

SHA3 hashes of namefilters are the linking scheme used in the decrypted layer.


```typescript
type PrivateRoot =
  CBOR<HAMT<
    hash(saturate(bareName)), // map key
    Array<CID<EncryptedPrivateBlock>> // map value
  >>

type HAMT<K, V> = {
  structure: "hamt"
  version: "0.1.0"
  root: Node<K, V>
}

type Node<K, V> =
  [ [UInt8, UInt8] // bitmask
  , Array<Entry<K, V>> // Entries
  ]

type Entry<K, V>
  = CID<CBOR<Node<K, V>>>
  | [K, V][] // bucket of values
```

### The Decrypted Layer

```typescript
type EncryptedPrivateBlock = PrivateNode<Namefilter, SkipRatchet>

type Namefilter = ByteArray<256>

type SkipRatchet = {
  large: ByteArray<32>
  mediumCount: Uint8
  medium: ByteArray<32>
  smallCount: Uint8
  small: ByteArray<32>
}

type PrivateNode
  = PrivateDirectory
  | PrivateFile

type PrivateDirectory = {
  revision: Encrypted<deriveKey(ratchet), CBOR<{
    ratchet: SkipRatchet
    bareName: Namefilter
    inumber: ByteArray<32> // TODO(matheus23): Inside or outside the revision section?
  }>>
  metadata: Metadata<false>
  entries: Record<string, {
    snapshotKey: hash(deriveKey(entryRatchet))
    revisionKey: Encrypted<deriveKey(ratchet), deriveKey(entryRatchet)>
    name: hash(saturated(entryBareName)) // links to a PrivateNode<entryBareName, entryRatchet>
  }>
}

type PrivateFile = {
  revision: Encrypted<deriveKey(ratchet), CBOR<{
    ratchet: SkipRatchet
    bareName: Namefilter
    inumber: ByteArray<32>
  }>>
  metadata: Metadata<true>
  content: ByteArray // TODO(matheus23) non-inline files
}
```

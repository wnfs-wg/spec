# WNFS Private Partition Specification

# 0 Abstract

The private file system provides granular control over read access along two dimensions: file hierarchy and time. Access is granted with a backward secret mechanism, where being grated access to a subgraph at a point in time is either a single snapshot point in time, or from that time forward -- but never access to the past of a point of history. Similarly, access to a directory implies access to all children nodes, but not to its parents or sibling nodes. Key sharing is an orthognal concern, and is accomplished out of band, or via the WNFS shared segment.

# 1 Terminology

Encryption adds another dimension to a file system: visibility. The data and file layers are each augmented with cleartext and ciphertext components. While namefilters and multivalues do encode a concept of an encrypted file, we generally only speak of the data layer. 

```
┌───Visibility───┐ ┌──────Noun──────┐

┌─────────────────┬─────────────────┐
│                 │                 │
│                 │       File      │
│                 │                 │
│    Decrypted    ├─────────────────┤
│                 │                 │
│                 │                 │
│                 │                 │
├─────────────────┤       Data      │
│                 │                 │
│    Encrypted    │                 │
│                 │                 │
└─────────────────┴─────────────────┘
```

Broadly speaking, there is a "decrypted" layer and a "encrypted" levels

- The "decrypted" level defines the type of data you can decrypt given the correct keys. Links between blocks in this layer are references in the HAMT data structure at the "encrypted" layer.
- The "encrypted" level defines how all of the encrypted data blocks are organized as IPLD data. Links in this layer are CID-links.

These all form graphs, where the nodes and links have different meanings per layer. The 

| Visibility | Layer | Node        | Link             |
|------------|-------|-------------|------------------|
| Decrypted  | File  | WNFS File   | File Path        |
| Decrypted  | Data  | CBOR Object | Namefilter + Key |
| Encrypted  | Data  | IPLD Block  | CID              |

# 2 Encrypted Layer

The encrypted layer hides the structure of the file system that it contains. The data MUST be placed into a flat namespace — in this case a [Merklized](https://en.wikipedia.org/wiki/Merkle_tree) [hash array mapped tire (HAMT)](https://en.wikipedia.org/wiki/Hash_array_mapped_trie). The root node of the resulting HAMT plays a very different role from a fie system root: it "merely" anchors this flat namespace, and is otherwise unrelated to the filesystem. The file system structure will be ["rediscovered" in the decrypted layer (§3)](#3-decrypted).

The encrypted layer is intended to hide as much information as possible, while still permitting write access validation by untrusted nodes. A single file system's encrypted root MAY represent a whole forest of decrypted file system trees. The roots of these trees MAY be completely unrelated. These are referred to as the `PrivateForest`. Since a reader may not know what else there is in the forest — and that it is safer to not reveal this information — we sometimes refer to the it as a ["dark forest"](https://en.wikipedia.org/wiki/The_Dark_Forest).

## 2.1 Ciphertext Blocks

At the encrypted data layer, the private forest is a collection of ciphertext blocks. These blocks SHOULD be smaller than 256 kilobytes in order to comply with the default IPFS block size. Keeping block size small is also useful for reducing metadata leakage - it's less obvious what the file size distribution in the private file system is like if these files are split into blocks.

Ciphertext blocks MUST be stored as the leaves of the HAMT that encodes a [multimap](https://en.wikipedia.org/wiki/Multimap). The HAMT MUST have a node-degree of 16[^1], and MUST used saturared  saturated [namefilter](/spec/namefilter.md)s as keys.

### 2.1.1 Data Types

The multimap container MUST be represented as a CBOR-encoded Merkle HAMT. The values MUST be a set of IPLD-fomatted binary blobs.

All values in the Merkle HAMT MUST be sorted in lexicographic ascending order by CID.

```typescript
type PrivateForest = CBOR<HAMT<Namefilter, Array<CID<ByteArray>>>>
  
type HAMT<K, V> = {
  structure: "hamt"
  version: "0.1.0"
  root: Node<K, V>
}

type Node<K, V> = [
  ByteArray<2>, // Bitmask Key
  Array<Entry<K, V>> // Entries
]

type Entry<K, V>
  = CID<CBOR<Node<K, V>>> // Subentry sharding
  | Array<[K, V]> // Bucket of values
```

In the above, note that `Node<K, V>` and `Entry<K, V>` are mutually recursive types. This permits the sharding of entries beyond the default IPLD link limit. <!-- FIXME @matheus plz confirm -->

## 2.2 Ciphertext Files

The encrypted file layer is a very thin enrichment of the data layer. In particular, it knows about about namefilters as labels, and ciphertext blobs as being separate from the expanded namefilter inside the bucket.

```
                               ┌───Index────┐
                               │            │
          ╭╶╶╶╶╶╶╶╶╶╶╶╶╶╶╶╶╶╶╶►│   Hashed   │
          ╵                    │ Namefilter │
          ╵                    │            │
          ╵                    └─────┬──────┘
          ╵                          │
          ╵                          ▼
┌─────────┼────────────────────────Bucket───────────────────────────────────────┐
│         ╷                                                                     │
│  ┌──────┴───────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │              │  │              │  │              │  │              │       │
│  │   Expanded   │  │  Ciphertext  │  │  Ciphertext  │  │  Ciphertext  │  ...  │
│  │  Namefilter  │  │    Node A    │  │    Node B    │  │    Node C    │       │
│  │              │  │              │  │              │  │              │       │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
```

# 3 Decrypted

The decrypted (or "cleartext") layer is where the actual structure of the file system is rediscovered out of the encrypted layer.

The decrypted layer has two sublayers: a cleartext data layer, and a cleartext file layer.

## 3.1 Cleartext Data 

The cleartext data layer makes use of the pointer machine from the encrypted layer to rediscover the semantically meaningful links in the file system. The private WNFS shares the same metadata structure as the [public WNFS](/spec/public-wnfs.md#metadata). Encryption keys and revision secrets are derived from a [skip ratchet](/spec/skip.ratchet.md).

```typescript
type Namefilter = ByteArray<256>
type Key = ByteArray<32>
type Inumber = ByteArray<32>

type PrivateNode
  = PrivateDirectory
  | PrivateFile

type PrivateNodeHeader = {
  ratchet: SkipRatchet
  bareName: Namefilter
  inumber: Inumber // Invariant: The `inumber` is included in `bareName`
}

type PrivateDirectory = {
  type: "wnfs/priv/dir"
  version: "0.2.0"
  // encrypted using deriveKey(ratchet)
  header: Encrypted<CBOR<PrivateNodeHeader>>
  // userland:
  metadata: Metadata
  entries: Record<string, {
    contentKey: Key // hash(deriveKey(entryRatchet))
    revisionKey: Encrypted<Key> // encrypt(deriveKey(ratchet), deriveKey(entryRatchet))
    name: Hash<Namefilter> // hash(saturated(add(deriveKey(ratchet), entryBareName)))
    // and can be used as the key in the private partition HAMT to lookup
    // a (set of) PrivateNode(s) with an entryBareName and entryRatchet from above
  }>
}

type PrivateFile = {
  type: "wnfs/priv/file"
  version: "0.2.0"
  // encrypted using deriveKey(ratchet)
  header: Encrypted<CBOR<PrivateNodeHeader>>

  // USERLAND
  metadata: Metadata
  content: ByteArray | { // FIXME what is the SR here for? Is this a tombstone? If so, is that not a different node type?
    ratchet: SkipRatchet 
    count: Uint64
  }
}
```

### 3.2.1 Node Headers

Node headers MUST be encrypted with the key derived from the node's skip ratchet: the "content key". Headers MUST NOT grant access to other versions of the associated node. Node headers are in kernelspace and MUST NOT be user writable. Refer to the key structure (FIXME) section for more detail.

### 3.2.2 Node Metadata

Node metadata is the userland equivalent of the node's header.

### 3.2.1 Private File

A private file MUST contain the actual bytes that represent the file. Files MAY also contain userland metadata.

### 3.2.1 Private Directory

A private directory MUST contain links to zero or more further nodes. Private directories MAY include userland metadata.

See the FIXME section for more information about the link structure.

### 3.2.3 Pointers & Keys

Keys are always attached to pointers to some data.

```
┌──Decryption Pointer──┐
│                      │
│     Namefilter &     │
│     Content Key      │
│                      │
└──────────────────────┘
```

#### 3.2.3.1 Node Key

Node keys MUST be deirved from the skip ratchet for that node, incremented to the relevant revision number. This limits the reader to reading from a their earliest ratchet and forward, but never earlier revisions than that.

#### 3.2.3.2 Content Key

Content keys MUST be dervied from the [Node Key](#3231-node-key) by hashing it with SHA3. The content key grants access to a single revision snapshot of that node and its children, but no other revisions forward or backward.

### 3.2.4 Read Hierarchy

Access in WNFS is fundamentally heirarchical. Access granted to a single node in a DAG implies access to all of its child nodes (and no others). Decryption pointers provide a way to "discover" the structure of the portion of the file system accessible to the viewer. This process is always statred from a pointer held by the viewer outside of the file system.

For example, having a decryption pointer to a directory with the `Documents/` and `Images/` directories could look something like this:

```
                                       ┌────External────┐
                                       │                │
                                       │  Namefilter &  │
                                       │  Content Key   │
                                       │                │
                                       └────────┬───────┘
                                                │
                                                ▼
                       ┌─────────────────Private Directory────────────────┐
                       │                                                  │
                       │  ┌───Documents/───┐          ┌─────Images/────┐  │
                       │  │                │          │                │  │
                       │  │  Namefilter &  │          │  Namefilter &  │  │
                       │  │  Content Key   │          │  Content Key   │  │
                       │  │                │          │                │  │
                       │  └────────┬───────┘          └────────┬───────┘  │
                       │           │                           │          │
                       └───────────┼───────────────────────────┼──────────┘
                                   │                           │
                                   ▼                           ▼ 
┌──────────────────Private Directory────────────────┐  ┌───Private Directory───┐
│                                                   │  │                       │
│  ┌────Thesis.pdf───┐         ┌─────Notes.md────┐  │  │  ┌───/Hawaii.png───┐  │
│  │                 │         │                 │  │  │  │                 │  │
│  │   Namefilter &  │         │   Namefilter &  │  │  │  │   Namefilter &  │  │
│  │   Content Key   │         │   Content Key   │  │  │  │   Content Key   │  │
│  │                 │         │                 │  │  │  │                 │  │
│  └───────┬─────────┘         └────────┬────────┘  │  │  └────────┬────────┘  │
│          │                            │           │  │           │           │
└──────────┼────────────────────────────┼───────────┘  └───────────┼───────────┘
           │                            │                          │
           ▼                            ▼                          ▼
   ┌──Private File──┐           ┌──Private File──┐         ┌──Private File──┐
   │                │           │                │         │                │
   │    Content     │           │    Content     │         │    Content     │
   │                │           │                │         │                │
   └────────────────┘           └────────────────┘         └────────────────┘
```

Each link in the above picture is looked up in the [encrypted HAMT](FIXME), decrypted, and transformed into further decrypted file system nodes. It is sometimes helpful to analogize this process as being similar to lazy loading remote content, though of course with content addressed encrypted-at-rest data the encrypted data could very well be stored locally.


## 3.2 

A key structure diagram exploring how hierarchical read access works:
Given the root content key, you can decrypt the root directory that contains the content keys of all subdirectories, which allow you to decrypt the subdirectories.
It's possible to share the content key of a subdirectory which allows you to decrypt everything below that directory, but not siblings or anything above.

| Line   | Meaning           |
|--------|-------------------| 
| `╴╴╴►` | Derive via SHA    |
| `───►` | Contains Key      |
| `═══►` | Increment Ratchet | 

```
                                                      Revisions
        ╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴►

        ┌  ╔══Skip Ratchet══╗                   ╔══Skip Ratchet══╗                   ╔══Skip Ratchet══╗
        │  ║                ║                   ║                ║                   ║                ║
     ═════►║  Root          ╠══════════════════►║  Root          ╠══════════════════►║  Root          ║
        │  ║  Revision: 4   ║                   ║  Revision: 5   ║                   ║  Revision: 6   ║
        │  ║                ╟╴╴╴╴╴╮             ║                ╟╴╴╴╴╴╮             ║                ╟╴╴╴╴╴╮
        │  ╚═══════╤════════╝     ╷             ╚════════╤═══════╝     ╷             ╚════════╤═══════╝     ╷
   Root │          │              ╷                      │             ╷                      │             ╷
        │          │              ▼                      │             ▼                      │             ▼
        │          │          ┌──Content Key──┐          │         ┌──Content Key──┐          │         ┌──Content Key──┐
        │          │          │               │          │         │               │          │         │               │
        │          │          │  Root         │          │         │  Root         │          │         │  Root         │
        │          │          │  Revision: 4  │          │         │  Revision: 5  │          │         │  Revision: 6  │
        │          │          │               │          │         │               │          │         │               │
        └          │          └───────┬───────┘          │         └───────┬───────┘          │         └───────┬───────┘
                   │                  │                  │                 │                  │                 │
                   │                  │                  │                 │                  │                 │
                   │                  │                  │                 │                  │                 │
                   │                  │                  │                 │                  │                 │
                   ▼                  │                  ▼                 │                  ▼                 │
        ┌  ╔══Skip Ratchet══╗         │         ╔══Skip Ratchet══╗         │         ╔══Skip Ratchet══╗         │
        │  ║                ║         │         ║                ║         │         ║                ║         │
        │  ║  Docs/         ╠═════════╪════════►║  Docs/         ╠═════════╪════════►║  Docs/         ║         │
        │  ║  Revision: 0   ║         │         ║  Revision: 1   ║         │         ║  Revision: 2   ║         │
        │  ║                ╟╴╴╴╴╴╮   │         ║                ╟╴╴╴╴╴╮   │         ║                ╟╴╴╴╴╴╮   │
        │  ╚════════════════╝     ╷   │         ╚════════╤═══════╝     ╷   │         ╚════════╤═══════╝     ╷   │
   Docs │                         ╷   │                  │             ╷   │                  │             ╷   │
        │                         ▼   ▼                  │             ▼   ▼                  │             ▼   ▼
        │                    ┌──Content Key──┐           │         ┌──Content Key──┐          │        ┌──Content Key──┐
        │                    │               │           │         │               │          │        │               │
        │                    │  Docs/        │           │         │  Docs/        │          │        │  Docs/        │
        │                    │  Revision: 0  │           │         │  Revision: 1  │          │        │  Revision: 2  │
        │                    │               │           │         │               │          │        │               │
        └                    └───────────────┘           │         └───────┬───────┘          │        └────────┬──────┘
                                                         │                 │                  │                 │
                                                         │                 │                  │                 │
                                                         │                 │                  │                 │
                                                         │                 │                  │                 │
                                                         ▼                 │                  ▼                 │
        ┌                                        ╔══Skip Ratchet══╗        │          ╔══Skip Ratchet══╗        │
        │                                        ║                ║        │          ║                ║        │
        │                                        ║  Note.md       ╠════════╪═════════►║  Note.md       ║        │
        │                                        ║  Revision: 0   ║        │          ║  Revision: 1   ║        │
        │                                        ║                ╟╴╴╴╴╴╮  │          ║                ╟╴╴╴╴─╮  │
        │                                        ╚════════════════╝     ╷  │          ╚════════════════╝     ╷  │
Note.md │                                                               ╷  │                                 ╷  │
        │                                                               ▼  ▼                                 ▼  ▼
        │                                                         ┌──Content Key──┐                    ┌──Content Key──┐
        │                                                         │               │                    │               │
        │                                                         │  Note.md      │                    │  Note.md      │
        │                                                         │  Revision: 0  │                    │  Revision: 1  │
        │                                                         │               │                    │               │
        └                                                         └───────────────┘                    └───────────────┘
        
```

A diagram exploring the revision key structure. Newer versions of files and directories are to the right of their older versions. As in the diagram above, hierarchy still goes from top to bottom, so subdirectories are below the directory that contains them. Given any of these boxes, follow the lines to see what data you can decrypt or derive.

Knowing the root content key of a directory will only give you access to a single revision of that file or directory, as the next revision will be derived from a separate skip ratchet that can't be derived from the current content key.

Knowing the root skip ratchet of a directory will give you access to that revision by deriving the content key of from the skip ratchet, and future revisions by stepping the ratchet forward and deriving content keys for those revisions.

It's impossible to read previous revisions given a skip ratchet, because it's computationally infeasible to compute the previous revision's skip ratchet.

### 3.2.3 Node Key Structure

A viewing agent may be able to view more than a single version of a file.

Below is a diagram representing the inner structure of a single cleartext node and its keys.

```
                                   ┌────────────┐
                                   │            │
╭╶╶╶╶╶╶╶╶╶╶╶╶Derive╶╶╶╶╶╶╶╶╶╶╶╶╶╶╶►│  Node Key  │
╷                                  │            │
╷                                  └─┬────┬───┬─┘
╷                                    │    │   │
╷              ┌─────────────────────┘    │   └──────────────SHA3───────────────┐
╷              │                          │                                     │
╷              │                          │                                     ▼
╷              │                          │                              ┌─────────────┐
╷              │                          │                              │             │
╷              │                          │                              │ Content Key │
╷              │                          │                              │             │
╷              │                          │                              └──────┬──────┘
╷              │                          │                                     │
╷              ▼                          ▼                                     ▼
╷  ┌────Private Header─────┐   ┌─────Temporal Header─────┐   ┌──────────Private Content───────────┐
╷  │                       │   │                         │   │                                    │
╷  │                       │   │   ┌─────────────────────┼───┼───────Documents/────┐ ┌──────────┐ │
╷  │  ┌──────────────┐     │   │   │                     ╷   ╷                     │ │          │ │
╷  │  │              │     │   │   │  ┌──────────────┐   ╷   ╷    ┌─────────────┐  │ │ Metadata │ │
╰╶╶┼╶╶┤ Skip Ratchet │     │   │   │  │              │   ╷   ╷    │             │  │ │          │ │
   │  │              │     │   │   │  │  Documents   │   ╷   ╷    │  Documents  │  │ └──────────┘ │
   │  └──────────────┘     │   │   │  │ Skip Ratchet │   ╷   ╷    │ Content Key │  │              │
   │                       │   │   │  │              │   ╷   ╷    │             │  │              │
   │  ┌─────────────────┐  │   │   │  └──────────────┘   ╷   ╷    └─────────────┘  │              │
   │  │                 │  │   │   │                     ╷   ╷                     │              │
   │  │ Bare Namefilter │  │   │   └─────────────────────┼───┼─────────────────────┘              │
   │  │                 │  │   │                         │   │                                    │
   │  └─────────────────┘  │   │                         │   │                                    │
   │                       │   │   ┌─────────────────────┼───┼─────────Apps/───────┐              │
   │  ┌──────────┐         │   │   │                     ╷   ╷                     │              │
   │  │          │         │   │   │  ┌──────────────┐   ╷   ╷    ┌─────────────┐  │              │
   │  │ i-number │         │   │   │  │              │   ╷   ╷    │             │  │              │
   │  │          │         │   │   │  │     Apps     │   ╷   ╷    │  Documents  │  │              │
   │  └──────────┘         │   │   │  │ Skip Ratchet │   ╷   ╷    │ Content Key │  │              │
   │                       │   │   │  │              │   ╷   ╷    │             │  │              │
   └───────────────────────┘   │   │  └──────────────┘   ╷   ╷    └─────────────┘  │              │
                               │   │                     ╷   ╷                     │              │
                               │   └─────────────────────┼───┼─────────────────────┘              │
                               │                         │   │                                    │
                               └─────────────────────────┘   └────────────────────────────────────┘
```

FIXME more storytelling abouyt the diagram here would be helpful

FIXME single source of truth for keys

FIXME explain how to walk the two parallel paths (headers & content) using above diagram


## 3.2 Cleartext Files

The decrypted (cleartext) file layer is very straightforward: it follows the exact interface as public WNFS files and directories. The primary difference is that while the public file system MUST form a DAG by its hash-linked structure, special care MUST be taken so that the private file system does not form pointer cycles.

### 3.2.1 Backwards Secrecy

FIXME add access control semantics.

# 4 Algorithms

All algorithms assume to have access to a `PrivateForest` in their context.

## 4.1 Namefilter Hash Resolving

`resolveHashedKey: Hash<Namefilter> -> (Namefilter, Encrypted<PrivateNodeHeader>, Array<Encrypted<PrivateNode>>)`

The private file system is a pointer machine, where pointers are hashes of namefilters.

To resolve a namefilter hash, look up the hash in the HAMT. The resulting key-value pair will give you the full namefilter as well as the private node header and a list of at least one private directory.

Looking up a namefilter hash in the HAMT works by splitting a hash into its nibbles. For example: a hash `0xf199a877d0...` gets split into the nibbles `0xf`, `0x1`, `0x9`, etc.

The nibble order is first taking the 4 most significant bits of and *then* the 4 least significant bits for each hash byte from byte index 0 to 31. This way this matches the usual hex encoding of byte-strings and reading the hex digits off one-by-one.

Each nibble is used as an identifier for a `Node`s child `Node`. Starting at the root, find the root `Node`s child by taking the nibble and computing the index of the child node as

$$\textsf{index} = \textsf{popcount}(bitmask \land ((1 \ll nibble) - 1))$$

If the child is a Node, repeat the process of with the next nibble.

If the child is a HAMT bucket of values, iterate that bucket to find one that has a namefilter that matches the hash of the namefilter. The associated values then contains the ciphertexts and the algorithm is done.


## 4.2 Private Versioning

`toVersioned : (Namefilter, RevisionKey) -> Namefilter`

Every private file or directory implicitly links to the name (namefilter) of its next version.

These implicit links can only be resolved when you have the revision key that allows you to decrypt the `PrivateNodeHeader`.

Given a `PrivateNodeHeader` it is possible to construct namefilters for newer versions of this private file or directory by stepping the ratchet forward as far as you want to look ahead.

Then, the new namefilter is:

$$saturate(add(deriveKey(inc^n(ratchet)), bareName))$$

Where
- $add$ refers to the [namefilter `add` operation](/spec/namefilter.md#Operation-add)
- $saturate$ refers to the [namefilter `saturate` operation](/spec/namefilter.md#Operation-saturate)
- $deriveKey$ refers to the [skip ratchet `deriveKey` operation](/spec/skip-ratchet.md#Key-Derivation) and
- $inc$ refers to the [skip ratchet increase operation](/spec/skip-ratchet.md#Increasing)

It is possible to choose $n$ in $inc^n(ratchet)$ and due to the properties of the skip ratchet skip ahead versions instead of having to skip one-by-one. When looking for the most recent version of a file, we recommend first skipping to the next small epoch, then the next medium, then large epochs, as long as these revisions exit, and then backtracking once an unpopulated revision is found.


## 4.3 Path Resolution

`resolvePath : (PrivateDirectory, Array<string>) -> Hash<Namefilter>`

Paths in the private file system need to be resolved relative to a directory to resolve the path from.

Path resolving can happen in three modes:
- Resolve the current snapshot: You only resolve the current snapshot of a version. This only requires a content key for decryption.
- Seeking: For each path segment, you look up the most recent version you can find (as described in the [private versioning algorithm](#Private-Versioning)). This requires access to the directory's revision key
- Seeking & repairing: Similar to seeking, this mode will look for the most recent version of a file and once it found it and in case it's not the same as what the parent refers to, it'll repair the parent's link to link to the most recent revision.

In all of these cases the next path segment's directory or file's hash of the namefilter can be retrieved by accessing the current directory's `directory.entries[segmentName].name`, looking the private node up as described in [Namefilter Hash Resolving](#Namefilter-Hash-Resolving) and then decrypting the content node(s) using `directory.entries[segmentName].contentKey`.

If this mode is seeking, the `directory.entries[segmentName].revisionKey` needs to be decrypted using the revision key for the current directory.


## 4.4 Sharded File Content Access

`getShards : PrivateFile -> Array<Namefilter>`

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

## 4.5 Merge

`merge : Array<PrivateForest> -> PrivateForest`

The private forest forms a join-semilattice with the `merge` ($\land$) function as the semilattice operation:
The merge operation is
- associative: $(a \land b) \land c = a \land (b \land c)$
- commutative: $a \land b = b \land a$
- idempotent: $a \land a = a$

The merge operation has an identity element which is the empty HAMT.

It is sufficient to describe a two-way merge function, as it can be extended to an $n$-ary algorithm by reducing the array of forests using the two-way merge. However, an implementation may decide to implement a more efficient $n$-ary merge.

When the two `PrivateForest`s to merge have the same CID, they're equal and thus nothing has to be done. This gives the merge algorithm the idempotence property.

Otherwise, merge the HAMT `Node`s of each `PrivateForest` together recursively. At each level:
- Find the difference between the `Node` bitmasks. The resulting bitmask is simply the binary-or of the input bitmasks.
- Figure out which children one `Node` adds over the other. Keep the superset of both sets of children.
- Recursively apply this algorithm on matching pairs of children nodes, unless they have the same CID.
- When merging buckets of values, merge them by matching keys in the bucket entries. Merge the values of matching keys by merging the CID lists using set semantics and keep the CID list sorted. Key-value pairs that only appear in one of the inputs are kept as-is. If the bucket gets bigger than the normal bucket size, split the bucket into its own node, as per its normal splitting semantics.

The private forest merge algorithm thus works completely on the encrypted layer and can be done by a third party that doesn't have read access to the private file system at all. However, there is some complexity involved when reading files. It's possible multiple "conflicting" file writes exist at a single revision. In these cases, we need to do some simple tie-breaking and may just choose the smallest CID.

 [^1]: See [`rationale/hamt.md`](/rationale/hamt.md) for more information.

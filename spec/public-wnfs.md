# Public WNFS

There are two types of nodes in the public partition:
- Directory Nodes
- File Nodes

Nodes need to be encoded as dag-cbor.

A directory node contains a map of entry names to either symlinks or CIDs that resolve to another directory or to a file.

See the [`notation.md`](/spec/notation.md) for information on the notation used and the [validation specification](/spec/validation.md) on details of verifying these data structures during deserialization.

```typescript
type PublicRoot = Cbor<PublicDirectory>

type PublicNode
  = PublicDirectory
  | PublicFile

type PublicDirectory = {
  "wnfs/pub/dir": {
    version: "0.2.0"
    previous: Array<Cid<Cbor<PublicDirectory>>>
    // userland:
    metadata: Metadata
    entries: Record<string, Cid<Cbor<PublicNode>> | PublicSymlink>
  }
}

type PublicSymlink = {
  ipns: string // e.g. alice.files.fission.name/public/<public-key>
}

type PublicFile = {
  "wnfs/pub/file": {
    version: "0.2.0"
    previous: Array<Cid<Cbor<PublicFile>>>
    // userland:
    metadata: Metadata
    content: Cid<IPFSUnixFSFile>
  }
}
```

## Metadata

The metadata field MUST be a CBOR map. It is in userland, and contains arbitrary keys and values.

```ts
type Metadata = Cbor<Record<string, DagCbor>>
```

## Algorithms

### Merge

`: Array<PublicNode> -> PublicNode`

Public file system nodes forms a join-semilattice with the `merge` ($\land$) function as the semilattice operation:
The merge operation is
- associative: $(a \land b) \land c = a \land (b \land c)$
- commutative: $a \land b = b \land a$
- idempotent: $a \land a = a$

It is sufficient to describe a two-way merge function, as it can be extended to an $n$-ary algorithm by reducing a non-empty array of forests using the two-way merge. However, an implementation may decide to implement a more efficient $n$-ary merge.

When the two `PublicNode`s have the same CID, they are the same and merging will just return that `PublicNode`. From this it follows that `PublicNode` merging is idempotent.

Otherwise, check whether the CID of one of the nodes is one of the (transitive) `previous` links of the other node.
If this is the case, then we know that that node is "older", because it appears in the history of the "newer" node. Thus the result of the merge operation is just the newer node.

If the nodes don't contain each other in their histories in one way, proceed to merging the node contents themselves.
For files, we can't make any assumptions about the file contents, so we need to tie-break on the file content. Merge in the file content with the lower CID.
For directories, merge the entry maps key-wise. If a directory entry only exists in one directory, keep it in the output as-is. If a directory entry exists in both nodes, recursively apply the merge algorithm on that pair of sub-directories or files. If one of the nodes is a directory and one is a file, take the node with the lower CID.

Once a node's content (either the `content` or the `entries` field) has been merged, create a new node that links back to all merged nodes in the `previous` field.
For all nodes that are to-be-merged, if they are a merge node themselves, i.e. if they have more than one `previous` entry, merge that node into the root merge node by repeating all its `previous` entries instead of linking to the merge node itself.


# Public WNFS

There are two types of nodes in the public partition:
- Directory Nodes
- File Nodes

Nodes need to be encoded as dag-cbor.

A directory node contains a map of entry names to either symlinks or CIDs that resolve to another directory or to a file.

```typescript
type PublicRoot = CBOR<PublicDirectory>

type PublicNode
  = PublicDirectory
  | PublicFile

type PublicDirectory = {
  type: "wnfs/pub/dir"
  version: "0.2.0"
  previous: Array<CID<CBOR<PublicDirectory>>>
  // userland:
  metadata: Metadata
  entries: Record<string, CID<CBOR<PublicNode>> | PublicSymlink>
}

type PublicSymlink = {
  ipns: string // e.g. alice.files.fission.name/public/<public-key>
}

type PublicFile = {
  type: "wnfs/pub/file"
  version: "0.2.0"
  previous: Array<CID<CBOR<PublicFile>>>
  // userland:
  metadata: Metadata
  content: CID<IPFSUnixFSFile>
}
```

## Metadata

Metadata in WNFS is a CBOR record with arbitrary keys and values.

```ts
type Metadata = CBOR<Record<string, any>>
```

## Algorithms

### Merge


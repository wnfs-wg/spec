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
  previous?: CID<CBOR<PublicDirectory>>
  merge?: CID<CBOR<PublicDirectory>>
  metadata: Metadata<"wnfs-dir">
  entries: Record<string, CID<CBOR<PublicNode>> | PublicSymlink>
}

type PublicSymlink = {
  ipns: string // e.g. alice.files.fission.name/public/ABCD
}

type PublicFile = {
  previous?: CID<CBOR<PublicFile>>
  merge?: CID<CBOR<PublicFile>>
  metadata: Metadata<"wnfs-file">
  content: CID<IPFSUnixFSFile>
}
```

## Metadata

Metadata in public WNFS must be extensible. Additional fields in the metadata type must be parsed without errors and preserved between versions unless explicitly modified by an application.

```typescript
type Metadata<Type> = {
  version: "0.2.0"
  type: Type
  unixMeta: UnixMeta
}

type UnixMeta = {
  mtime: Int // POSIX timestamp
  ctime: Int // POSIX timestamp
  mode: Int // POSIX file mode, e.g. (octal) 666 or 777
}
```

## Algorithms

### Merge


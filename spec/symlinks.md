# Symlink Specification

Symlink is short for "Symbolic Link".

# 0 Abstract

Symlinks are a tool for users to organize their file system:
- They allow linking to the same directory from different places in their file system
- They enable linking out to other user's file systems from your own file system
- They enable linking between public and private parts of file systems

While hard links (`CID`s or private references) conceptually are supposed to work like 'copying' the linked data, a symlink is meant to work differently:
- It won't guarantee availability.
- It's mutable: Resolving the same link twice may result in different results.
- It doesn't provide integrity of the linked data on its own.

Symlinks shouldn't be confused with a key management mechanism: Knowing a symlink doesn't provide additional read or write access. This ensures the conceptual invariant that giving someone read or write access to a directory will only give them access to that directory and anything contained in it. Symlinks are not meant to 'contain' the data they link to, as that may change over time.

# 1 Layout

Because of differences in addressing between public and private WNFS, symlinks have different formats depending on whether you're linking into public or linking into private data.

```ts
type Symlink = PublicSymlink | PrivateSymlink
```

Symlinks are objects containing a `type` key set to either `"wnfs/sym/pub"`, if it's a public symlink or `"wnfs/sym/priv"`, if it's a private symlink.

We call symlinks that refer to some data that is external to the current WNFS 'foreign'. Foreign symlinks MUST include a `root` pointer to the root they're relative to.

## 1.1 Public Symlink

```ts
type PublicSymlink = {
  type: "wnfs/sym/pub"
  root?: string
  path: string[]
}
```

Public data is symlinked via a list of [UTF-8](https://www.rfc-editor.org/rfc/rfc3629) path segment strings.

If the symlink is not linking relative to the same file system, it can provide a `root` parameter: An [`ipns://` URI](https://github.com/ipfs/in-web-browsers/blob/master/ADDRESSING.md) to the public root that the path is relative to.

A foreign public symlink's `root` MUST point to a [public directory](/spec/public-wnfs.md).

### 1.1.1 Public Symlink Examples

```json
{
  "type": "wnfs/sym/pub",
  "root": "ipns://matheus23.files.fission.name/public/",
  "path": ["Profile", "Avatar.jpg"]
}
```

```json
{
  "type": "wnfs/sym/pub",
  "path": ["Documents", "Notes.md"]
}
```

```json
{
  "type": "wnfs/sym/pub",
  "root": "ipns://matheus23.files.fission.name/public/",
  "path": []
}
```

```json
{
  "type": "wnfs/sym/pub",
  "root": "ipns://k51qzi5uqu5dgutdk6i1ynyzgkqngpha5xpgia3a5qqp4jsh0u4csozksxel2r/public/",
  "path": []
}
```

## 1.2 Private Symlink

```ts
type PrivateSymlink = {
  type: "wnfs/sym/priv"
  root?: string
  inumber: Inumber
}
```

A private symlink identifies a private node via its `inumber`.

A foreign private symlink's `root` MUST point to a [private forest](/spec/private-wnfs.md#21-ciphertext-blocks).

### 1.2.1 Private Symlink Examples

```json
{
  "type": "wnfs/sym/priv",
  "root": "ipns://matheus23.files.fission.name/private",
  "inumber": <CBOR-encoded 32 byte array>
}
```

```json
{
  "type": "wnfs/sym/priv",
  "inumber": <CBOR-encoded 32 byte array>
}
```

```json
{
  "type": "wnfs/sym/priv",
  "root": "ipns://k51qzi5uqu5dgutdk6i1ynyzgkqngpha5xpgia3a5qqp4jsh0u4csozksxel2r/private/",
  "inumber": <CBOR-encoded 32 byte array>
}
```


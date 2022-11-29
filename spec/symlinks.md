# Symlink Specification

# 0 Abstract

Symlinks are a tool for users to organize their file system:
- They allow them to link to the same directory from different places in their file system
- They enable linking out to other user's file systems from your own file system
- They enable linking between public and private parts of file systems

This shouldn't be confused with a key management mechanism: Knowing a symlink shouldn't give someone more read or write access. This prevents e.g. accidentally giving someone access to another file system's files if a directory includes a symlink.

While hard links (`CID`s or private references) conceptually are supposed to work like 'copying' the linked data, a symlink is meant to work differently:
- It won't guarantee availability.
- It's mutable: Resolving the same link twice may result in different results.
- It doesn't provide integrity of the linked data on its own.

# 1 Layout

```ts
type Symlink = PublicSymlink | PrivateSymlink

type PublicSymlink = {
  type: "wnfs/sym/pub"
  root: string // URI-encoded link to a public file system root
  // examples:
  // ipfs://bafkreiabxjdrtsaln7urdmeru7afcjfwj3xm5fsobhafr34ptac5vssunm
  // ipns://matheus23.files.fission.name/public/
  path: string[]
  // e.g. ["Documents", "Notes.md"], []
}

type PrivateSymlink = {
  type: "wnfs/sym/priv"
  root: string // URI-encoded link to a private forest root
  // examples:
  // ipfs://bafkreiabxjdrtsaln7urdmeru7afcjfwj3xm5fsobhafr34ptac5vssunm
  // ipns://matheus23.files.fission.name/private/
  inumber: Inumber // 32 bytes
}
```

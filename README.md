# Webnative File System (WNFS) Specification v0.2.0

# Abstract

The Web Native File System (WNFS) is a file system written for the web. It is versioned, logged, programmable, has strong-yet-flexible security, and is fully controlled by the end user. Service providers can validate writes without reading the contents of the file system, and minimal metadata is leaked.

WNFS relies heavily on the ”space” side of the space/time tradeoff to deliver performance. As a consequence of this and Merklization, WNFS can provide advanced functionality such as versioning and delegated or collaborative write access.

WNFS can be used for offline collaboration, because WNFS forms a state-based conflict-free replicated data type (CRDT): There exists a commutative and associative merge function that combines any two (versions of) WNFS roots.

WNFS is content-addressed and thus extremely portable. It may be stored on the edge, on the end user's device or in the cloud. Devices may also only partially load a WNFS and still write files and directories.

# Data Layer

Every WNFS is meant to have a single owner (user). At the root it's a [UnixFS](https://github.com/ipfs/specs/blob/main/UNIXFS.md) directory with links to WNFS's partitions:

- `public`: A link to the root directory of all public data
- `private`: A link to all private data organized in a HAMT
- `pretty`: A reduction of the public partition into UnixFS to make it easy to serve over IPFS http gateways.

It also includes a single UnixFS file at the link `version` that encodes in UTF8 the WNFS version `0.2.0`.


## `public` Partition

There are three types of nodes in the public partition:
- Directory Nodes
- File Nodes
- Symlink Nodes

Nodes need to be encoded as dag-cbor.

A directory node contains a table of links 

The root node of the public partition must be a directory node.

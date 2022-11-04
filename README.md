⚠️ This spec suite is Work-in-Progress. It may be useful for reference & discussion, though ⚠️

# Webnative File System (WNFS) Specification v0.2.0-alpha

## Abstract

The Web Native File System (WNFS) is a distributed file system. It is versioned, logged, programmable, has strong-yet-flexible security, and is fully controlled by the end user. Service providers can validate writes without reading the contents of the file system, and minimal metadata is leaked.

WNFS relies heavily on the ”space” side of the space/time trade-off to deliver performance. As a consequence of this and Merklization, WNFS can provide advanced functionality such as versioning and delegated or collaborative write access.

WNFS can be used for offline collaboration, because WNFS forms a state-based conflict-free replicated data type (CRDT): There exists a commutative and associative merge function that combines any two (versions of) WNFS roots.

WNFS is content-addressed and thus extremely portable. It may be stored on the edge, on the end user's device or in the cloud. Devices may also only partially load a WNFS and still write files and directories.


## Suite of Specifications

The WNFS spec is a suite of specifications consisting of public WNFS, private WNFS and its parts. Some of these specifications use common notation described in [notation.md](/spec/notation.md).

The specifications are:
- [Public WNFS](/spec/public-wnfs.md)
- [Private WNFS](/spec/private-wnfs.md)
- [Namefilters](/spec/namefilter.md)
- [Skip Ratchet](/spec/skip-ratchet.md)
## Editors

* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)
* [Philipp Krüger](https://github.com/matheus23), [Fission](https://fission.codes)

## Authors

* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)
* [Philipp Krüger](https://github.com/matheus23), [Fission](https://fission.codes)
* [Daniel Holmgren](https://github.com/dholms), [Bluesky](https://blueskyweb.xyz/)
* [Brendan O'Brien](https://github.com/b5), [number 0](https://www.n0.computer)

## Acknowledgments

Thank you [Brendan O'Brien](https://github.com/b5) for invaluable feedback on specification changes and building a benchmarking golang implementation of WNFS.

Thank you [Steven Allen](https://github.com/stebalien) engaging with the specification, pointing out concerns with namefilters, coming up with salted namefilters and suggesting them as an improvement.

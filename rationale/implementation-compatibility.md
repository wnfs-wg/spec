# Implementation compatibility notes

These notes are non-normative. They compare this work-in-progress suite with
[rs-wnfs 0.3.0](https://github.com/wnfs-wg/rs-wnfs/tree/80d542827f742274aed2d7103c3385a104b299f6)
and `skip_ratchet` 0.3.0. An implementation release is not an automatic normative
override of this specification. In particular, the big-endian change in
[spec PR #76](https://github.com/wnfs-wg/spec/pull/76) should not be silently
reversed just to match an existing release.

| Boundary | Published suite | Pinned implementation | Interoperability consequence |
| --- | --- | --- | --- |
| Private node version | `0.2.0` | `1.0.0` | Select an explicit target; do not merely rewrite the version field. |
| Temporal key context | Separate temporal and revision-label purposes | `wnfs/1.0/temporal derivation from ratchet` | Using the revision-segment context produces a different key. |
| Header protection | Header contains the skip ratchet; content uses the snapshot key | AES-KWP with temporal key for the header | Snapshot readers must not obtain the header's ratchet. |
| Hash-to-prime counter | 32-bit big-endian counter | 32-bit little-endian counter, then candidate bit zero is set to make it odd | Different candidate sequences produce different name segments and labels. |
| Medium ratchet carry | New medium is the hash of the old medium; new small hashes salt and old medium | Intervening preimage is the hash of old medium; new medium hashes that preimage again; new small hashes salt and that preimage | Carrying past a small epoch can diverge even if earlier keys matched. |
| Large ratchet carry | Prose describes updating the old medium | Rebuild from old large: hash old large; derive a medium preimage from salt and old large; hash it for medium and with salt for small | Validate large-epoch transitions separately. |
| File content | `inline` byte array; external `blockSize` | `inline: {data: [...]}`; external `blockContentSize` | Primitive encryption agreement is not private-file codec agreement. |

Source anchors:

- [Key contexts and encryption primitives](https://github.com/wnfs-wg/rs-wnfs/blob/80d542827f742274aed2d7103c3385a104b299f6/wnfs/src/private/node/keys.rs)
- [Header storage and loading](https://github.com/wnfs-wg/rs-wnfs/blob/80d542827f742274aed2d7103c3385a104b299f6/wnfs/src/private/node/header.rs)
- [Hash-to-prime candidate generation](https://github.com/wnfs-wg/rs-wnfs/blob/80d542827f742274aed2d7103c3385a104b299f6/wnfs-nameaccumulator/src/fns.rs)
- [Skip-ratchet epoch transitions](https://docs.rs/skip_ratchet/0.3.0/src/skip_ratchet/ratchet.rs.html)
- [Node serialization](https://github.com/wnfs-wg/rs-wnfs/blob/80d542827f742274aed2d7103c3385a104b299f6/wnfs/src/private/node/serializable.rs)
- [File content serialization](https://github.com/wnfs-wg/rs-wnfs/blob/80d542827f742274aed2d7103c3385a104b299f6/wnfs/src/private/file.rs)

A useful cross-implementation corpus should pin source revisions, algorithms,
contexts and codec versions. Include epoch carry boundaries, rejection controls,
and full private-node read/write tests. Matching a cipher vector or a HAMT alone
does not establish private filesystem compatibility.

Resolving the counter endianness, ratchet carry descriptions and node encoding
version is a separate compatibility decision. These notes deliberately do not
change those encodings or announce a new suite version.

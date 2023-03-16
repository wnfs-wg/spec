In this document we list what we think would be sensible alternatives to picked encryption algorithms for use-cases in WNFS. For a good resource on encryption algorithms in general, we recommend [this post](https://soatok.blog/2020/07/12/comparison-of-symmetric-encryption-methods/) and its further links.


# File Encryption

## Need

A way to encrypt [`PrivateFile` and `PrivateDirectory` blocks](/spec/private-wnfs.md#31-cleartext-data) as well as externalized file content blocks.

Note that authentication is not strictly necessary since write access is managed via [PKI](https://en.wikipedia.org/wiki/Public_key_infrastructure)-signed certificates instead of knowledge of a secret (like a read key). However, it is useful to have another layer of authentication for principals with write (but not read) access.

## Choice

We chose [AES-GCM] with per-message randomly generated 12-byte initialization vectors, because it is widely supported and implemented in hardware as of 2023.
There are no known practical attacks on AES-GCM.


## Alternatives

### [AES-GCM-SIV]

Compared to AES-GCM, this mode additionally protects against nonce-misuse via a synthetic initialization vector (SIV). Introducing a deterministic nonce based on the content has a roughly 50% performance overhead ([2/3 throughput][AES-GCM-SIV: Specification and Analysis]).

Aside from nonce-misuse resistance, the primary motivation for deterministic encryption is file deduplication. Since each revision of a file in WNFS has a unique encryption key, deduplication can only occur when two concurrent writes of the same file are made to the same point in the file's history. This case is not very common, so the write overhead on every file outweighs the deduplication benefit.

### [AES-OCB] ([paper](https://link.springer.com/article/10.1007/s00145-021-09399-8))

This is probably the fastest AEAD cipher that can be implemented in today's hardware. However, due to patent restrictions in the past its use was not widespread, which to this day makes it hard to good support in various programming languages.

### ChaCha-based ciphers (e.g. [ChaCha20-poly1305])

These ciphers run at roughly 4 times the throughput in software compared to AES-GCM and are comparatively easier to implement as constant-time algorithms.
However, AES-GCM has wider hardware support at the moment.
In the future, it's likely we'll reconsider these ciphers.


# Key Wrapping

## Need

A way to wrap [`TemporalKey`s](/spec/private-wnfs.md#3161-temporal-key) of subdirectories within [`PrivateDirectory`](/spec/private-wnfs.md#31-cleartext-data) blocks.

## Choice

We chose [AES-KWP] with 256-bit keys.

## Alternatives

### [AES-SIV]

AES-SIV supports associated data. However, keys need to be twice the size. Associated data is not necessary to prevent attacks: We don't care about the wrapped key's integrity, as it'll be checked either when the snapshot key derived from it is used with AES-GCM or the private node header is decrypted and the namefilter it implies is checked against the HAMT label.

### [AES-KW]

This mode works just like AES-KWP, except it's slightly simpler due to not including a padding algorithm. For the same reason it only supports cleartext lengths which are multiples of 8 bytes. Even though some of our key wrapping use cases are multiples of 8 bytes, other uses aren't and we need the extra complexity of AES-KWP. For consistency we thus prefer AES-KWP over AES-KW.


# File Header Encryption

## Need

Deterministic encryption for [`PrivateNodeHeader`](/spec/private-wnfs.md#31-cleartext-data)s. The advantage of deterministic encryption over probabilistic encryption for these headers is that replicas that write into the same revision concurrently also derive the same encrypted node headers and can thus skip exchanging them over the network, improving sync performance.

## Choice

We chose [AES-KWP] as it's the most widely supported authenticated deterministic encryption cipher.

## Alternatives

### [AES-SIV]

This mode's advantage is support for associated data. However, keys need to be twice the size and we don't need associated data, since we can just check that the HAMT label that is derived from the node header's contents matches its context.


[AES-SIV]: https://www.rfc-editor.org/rfc/rfc5297
[AES-KW]: https://www.rfc-editor.org/rfc/rfc3394
[AES-KWP]: https://www.rfc-editor.org/rfc/rfc5649
[AES-GCM-SIV]: https://www.rfc-editor.org/rfc/rfc8452.html
[AES-GCM]: https://csrc.nist.gov/publications/detail/sp/800-38d/final
[AES-OCB]: https://web.cs.ucdavis.edu/~rogaway/ocb/ocb-faq.htm
[ChaCha20-poly1305]: https://www.rfc-editor.org/rfc/rfc7539
[AES-GCM-SIV: Specification and Analysis]: https://eprint.iacr.org/2017/168.pdf

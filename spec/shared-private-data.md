# WNFS Shared Private Data Extension Specification

# 0 Abstract

There are several cases where we need to exchange private data asynchronously when the other party may be offline.
The shared private data extension allows a file system's owner to deposit messages encrypted with a recipient's public key on their private forest.
A recipient will then periodically - or after being prompted - scan the sender's private forest for payloads they can decrypt.
These encrypted payloads contain secrets giving read access and/or [UCANs](https://ucan.xyz) giving write access to parts of the owner's public or private file system.

# 1 Exchange Keys

To share information with a user that's offline we make use of asymmetric encryption. All WNFS users widely distribute a list of public 2048-bit RSA public keys - their non-exportable "exchange keys" - as `did:key:` DIDs at a well-known location: A list under `exchange` at the root of WNFS next to `public` and `private`. See [the rationale](/rationale/shared-private-data.md#exchange-key-location) for more information on this.

These RSA keys are used to encrypt a symmetric key for a recipient; the "share key". This symmetric key is then used to decrypt a pointer a private node.

Allowing multiple exchange keys per WNFS enables multiple recipient devices to receive a share without having to transfer exchange keys between devices.

# 2 Data Layout

Shared private data payloads are stored in the [Private Forest](/spec/private-wnfs.md#21-ciphertext-blocks).

## 2.1 Share Label

Shares are labeled in the private forest by a namefilter consisting of:
- The sender's root DID. In most cases this will be the file system owner's root DID.
- The recipient's exchange DID
- A counter, which is incremented each time the sender creates a share.

The counter MUST be a 0-based natural number.

The namefilter for a share is computed like this:

```ts
function computeShareLabel(senderRootDID: string, recipientExchangeDID: string, counter: number) {
  return new Namefilter()
    .add(senderRootDID)
    .add(recipientExchangeDID)
    .add(counter.toString())
    .saturate()
}
```

## 2.2. Share Payload

The share payload MUST be a non-empty list of CIDs to [RSAES-OAEP](https://datatracker.ietf.org/doc/html/rfc3447#section-7.1) encrypted ciphertexts.

When decrypted, this reveals a payload of the form:

`<unsigned varint prefix><32 byte private forest label><32 byte AES key>`

As exchange keys are 2048 bit and can thus encrypt only up to 190 bits of data, the unsigned varint prefix MUST be smaller than $190 - 32 \cdot 2 = 126$ bytes.

In the current specification, valid prefixes always encode into 8 bytes. This means *decrypted* share payloads will always be 72 bytes long.

Share payload *ciphertexts* will always be 256 bytes long, due to the way RSAES-OAEP works.

### 2.2.1 Unsigned Varint Prefix

WNFS (currently) uses the [multicodec private use range](https://github.com/multiformats/multicodec/#private-use-area) for prefixing and identifying share pointers.

| Prefix | Meaning | Decoded Unsigned Varint | Prefix Computation |
|-------:|---------|-------------------------|:-------------------|
| `0xceaec101` | Temporal Share Pointer | `0x30574e` | `varint_enc(0x300000 \| ascii_enc("WN"))` |
| `0xcfaec101` | Snapshot Share Pointer | `0x30574d` | `varint_enc(0x300000 \| ascii_enc("WN") + 1)` |

### 2.2.2. Temporal Share Pointer

After the unsigned varint prefix `0xceaec101`, the temporal share pointer consists of
- a 32 byte private forest label used to look up the private node to decrypt in the private forest
- a 32 byte [revision key](/spec/private-wnfs.md#3161-revision-key) to decrypt the private node and its temporal header.

### 2.2.3 Snapshot Share Pointer

The snapshot share pointer works exactly like the temporal share pointer, except it is prefixed with `0xcfaec101` and instead of encoding a revision key, it encodes a [content key](/spec/private-wnfs.md#3162-content-key).

This content key only gives access to a single revision, in case a user wanted to only share a single revision with someone else.

# 3 Share Lookup and Discovery

It's not assumed that recipients are notified over some channel about whether they recieved any new shares.

Thus it is possible to scan other user's file systems for files that were shared with a given exchange key. To do this, generate share labels as described in [section Share Label](#21-share-label) starting from 0 or the last counter used for lookup until the first missing label in the private forest is hit.

# 4 Concurrent Share Creation

If multiple shares are created concurrently on separate devices, then these can simply be merged according to the [private forest merge algorithm](/spec/private-wnfs.md#45-merge).

Multiple share payloads are allowed per share label.

# 5 Share Creation Permission

## 5.1 Share as Inbox






# TODO & thoughts
* think through write permissions. What would UCANs look like if this were done with RSA accumulators?
* may not need a shared counter
* put shared in the forest? Put stuff in the rsa accumulator
* extract .well-known/exchange/* into top level?
* make symlinks not contain the key! Only forest label (+ inumber?)

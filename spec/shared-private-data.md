# WNFS Shared Private Data Extension Specification

# 0 Abstract

There are several cases where users want to exchange private data asynchronously when the other party may be offline. In such cases we can make use of [store-and-forward networks](https://en.wikipedia.org/wiki/Store_and_forward) to deposit data for later retrieval.
The shared private data extension allows a file system's owner to deposit asymmetrically encrypted messages so they can be picked up by recipients when they come online.
Recipients can periodically - or after being prompted - scan the sender's private forest for payloads they can decrypt, for example by looking for updates in file systems from a contact list, or by being sent a message pointing to the share payload over an out-of-bounds secure channel.
These encrypted payloads contain secrets giving read access and/or [UCANs](https://ucan.xyz) giving write access to parts of the owner's public or private file system.

# 1 Protocol Versions

The shared private data extension is versioned and designed to support multiple versions living on the same file system concurrently, so devices can gradually upgrade to the new version and support multiple versions at the same time.

The protocol version determines the algorithm used, key size and other parameters.

## 1.1 Version 1

Version 1 of the shared private data protocol uses RSA public keys for asymmetric encryption. These RSA public keys MUST have an exponent of 65537 and MUST be 2048-bit.
The method for encryption MUST be RSAES-OAEP.

# 2 Exchange Keys Partition

To share information with a user who is offline we make use of asymmetric encryption. All WNFS users widely distribute a list of public keys - their non-exportable "exchange public keys" at a well-known location: A map under `exchange` at the root of WNFS next to `public` and `private`. See [the rationale](/rationale/shared-private-data.md#exchange-key-location) for more information.

These keys are used to encrypt a [share payload](#32-share-payload) for a recipient. This share payload contains a pointer to a private node and the symmetric key to decrypt it.

Allowing multiple exchange keys per WNFS enables multiple recipient devices to receive a share without having to transfer exchange keys between devices. This enables exchange keys to be stored as non-extractable keys in device enclaves for safety.

## 2.1 Layout

The exchange key partition is structured as a [public directory](/spec/public-wnfs.md) under its own top-level link. This directory contains further directories containing exchange keys.

Exchange keys are grouped by device, such that a sender only needs to choose *one* of the available keys for each device. For example, senders that support protocol version 1 and 2 look through all recipient's devices and select either a single version 2 exchange key or, if no such key is present, a version 1 key from each device.

These keys are grouped as a directory with a unique name per device. It is RECOMMENDED to use a [base64URL](https://www.rfc-editor.org/rfc/rfc4648#section-5)-encoded 32-byte nonce, as that does not leak any information about the device and is sufficiently collision-resistant.
In the future, implementations MAY choose to use a meaningful identifier instead, so implementations SHOULD be lenient with directory names.

See the [example exchange partition layout](#exchange-partition-layout) for examples.

### 2.1.1 Key Encoding

Individual device exchange keys are versioned. The protocol version is derived from the file name. Protocol version 1 exchange keys are named `v1.exchangekey`.

The file's content is 256 bytes of the exchange key's RSA public modulus encoded as low-endian unsigned integer.

See the [test vectors](#exchange-key-test-vectors) for examples.

### 2.2 Merging

The exchange partition is merged according to the rules for [public directory merging](/spec/public-wnfs.md#merge).

# 3 Shared Private Data

Shared private data payloads are stored in the [Private Forest](/spec/private-wnfs.md#21-ciphertext-blocks).

## 3.1 Share Label

Shares are labeled in the private forest by a [`NameAccumulator`](/spec/nameaccumulator.md) consisting of:
- The sender's root DID. In most cases this will be the file system owner's root DID.
- The recipient's exchange key
- A counter, which is incremented each time the sender creates a share.

The counter MUST be a 0-based natural number.

The `NameAccumulator` for a share MUST be computed as follows:

```ts
function computeShareLabel(senderRootDID: string, recipientExchangeKey: ByteString, counter: number) {
  return NameAccumulator.empty()
    .add(senderRootDID)
    .add(recipientExchangeKey)
    .add(encode(counter))
}
```

Here `encode` is a function that maps a share counter to a low-endian byte array encoding of a 64-bit unsigned integer.

The `recipientExchangeKey` are the recipient device's exchange key bytes, including the protocol versioning prefix.

## 3.2 Share Payload

The share payload MUST be a non-empty list of CIDs to [RSAES-OAEP](https://datatracker.ietf.org/doc/html/rfc3447#section-7.1)-encrypted ciphertexts.

The decrypted payload MUST be a CBOR-encoded object of following shape:

```typescript
type SharePayload = TemporalSharePointer | SnapshotSharePointer

type TemporalSharePointer = {
  "wnfs/share/temporal": {
    label: Hash<NameAccumulator> // 32 bytes SHA3 hash
    cid: Cid // content block CID
    temporalKey: Key // 32 bytes AES key
  }
}

type SnapshotSharePointer = {
  "wnfs/share/snapshot": {
    label: Hash<NameAccumulator> // 32 bytes SHA3 hash
    cid: Cid // content block CID
    snapshotKey: Key // 32 bytes AES key
  }
}
```

See the [validation specification](/spec/validation.md) on requirements for validating this data during deserialization.

Because v1 exchange keys are 2048 bit RSAES-OAEP keys, they can only encrypt up to 190 bits of data. Both payloads of the above format encode to 157, so fit within RSAES-OAEP limits.

NB: Share payload *ciphertexts* will always be 256 bytes long, due to the way RSAES-OAEP works.

### 3.2.2 Temporal Share Pointer

The temporal share pointer consists of
- a 32 byte private forest label used to look up the private node to decrypt in the private forest
- a 32 byte [temporal key](/spec/private-wnfs.md#3161-temporal-key) to decrypt the private node and its temporal header.

### 3.2.3 Snapshot Share Pointer

The snapshot share pointer works exactly like the temporal share pointer, except instead of encoding a temporal key, it encodes a [snapshot key](/spec/private-wnfs.md#3162-snapshot-key).

This snapshot key only gives access to a single revision, in case a user wanted to only share a single revision with someone else.

# 4 Share Lookup and Discovery

It's not assumed that recipients are notified over some secondary channel when they receive new shares.

Thus it is possible to scan other user's file systems for files that were shared with a given exchange key. To do this, generate share labels as described in [section Share Label](#31-share-label) starting from 0 or the last counter used for lookup until the first missing label in the private forest is hit.

# 5 Concurrent Share Creation

If multiple shares are created concurrently on separate devices, then these are merged according to the [private forest merge algorithm](/spec/private-wnfs.md#45-merge).

Multiple share payloads are allowed per share label.

# 6 Externally Shared Data

The shared private data extension also allows dropping off secrets on a file system with the owner as recipient instead of as a sender.

A sender with write access to the root owner's private forest would thus drop off share payloads in the private forest at the labels corresponding to all combinations of the sender's root DID and the file system owner's listed exchange keys.

The file system owner can later receive share payloads by looking through their own file system with combinations of the known sender's root DID and of their own exchange keys.

# 7 Sharing Multiple Private Nodes

If a sender wants to share multiple private nodes at the same time, it is RECOMMENDED to create a private directory containing all nodes to share and create a single share payload pointing to that directory.

This directory MAY be constructed on-the-fly. Its name then contains the sender's root DID and the node's i-number. This ensures the sender will always have permission to create this name in the private forest.
The directory's contents may then have names that aren't derived from the name of the sharing directory.

# Appendix

## Test Vectors

### Exchange Key Test Vectors

The hex encoded bytes for the [RSA-2048](https://en.wikipedia.org/wiki/RSA_numbers#RSA-2048) public modulus used as an exchange key of protocol version 1:

```
e5c71c36c6489d3916bcf71768eba533e724c85450f930ccbc6628171556f531311b0ffca3241f72d873d3d9a4166be523793b3c5bdc614cf2202964929572bc32adbd2595902c8765ad956aac109f6005cd80dcad1318cbb53453f8095813f659517da33e5f95eb6ce69d430927443f374dd689af79c402fc666cd2efdae8f74a52efbd92f535bebb30d7c4c291b98eba643167d1e41b78fd7b1fb5044af1b4359f03803ca30ed492855ecfc709eb46b68433c9ffb6b4448bff65770b5b1fa313288c64f00741a032deacc2acee1a72bd110065d1932fc79e18a11e8edbf07f5bbb5035466f72a8f1f590c781109173cd13a67a1a20904475b0c3dcee0c97c7
```

The RSA-2048 public modulus is this decimal-encoded number:

```plain
RSA-2048 = 2519590847565789349402718324004839857142928212620403202777713783604366202070
           7595556264018525880784406918290641249515082189298559149176184502808489120072
           8449926873928072877767359714183472702618963750149718246911650776133798590957
           0009733045974880842840179742910064245869181719511874612151517265463228221686
           9987549182422433637259085141865462043576798423387184774447920739934236584823
           8242811981638150106748104516603773060562016196762561338441436038339044149526
           3443219011465754445417842402092461651572335077870774981712577246796292638635
           6373289912154831438167899885040445364023527381951378636564391212010397122822
           120720357
```

### Exchange Partition Layout

An example exchange partition tree with a hypothetical protocol version 2 looks like this:

```
/exchange
├── aNXmyZ-kRI1-fsLcOol8wi0wfmxxY5_15bxdt2T4bs8
│   ├── v1.exchangekey
│   ├── v2.exchangekey
├── BEubij6fgCV51wyhYtITYCYhoGvk5Y5Slr59Kl4FOkQ
│   └── v1.exchangekey
└── zulwKufD7v71rJ7792cl2pi1LHmePS7d7a0vhuvaKNo
    └── v2.exchangekey
```

This represents an exchange partition with three devices.
The first device (with the label `aNXmyZ...`) has two exchange keys. It supports both protocol version 1 and 2.
The second device (with the label `BEubij...`) has not yet upgraded to protocol version 2. It only provides a single version 1 exchange key.
The third device (with the label `zulwKu...`) was initialized long after protocol version 2 support was implemented. It thus never generated a version 1 exchange key and operates completely on protocol version 2.

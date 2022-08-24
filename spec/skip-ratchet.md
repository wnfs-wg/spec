# Skip Ratchet

The skip ratchet is described in this paper: https://eprint.iacr.org/2022/1078

## Encoding

This specifies the field names for the skip ratchet when encoded as CBOR:

```typescript
type SkipRatchet = {
  large: ByteString<32>
  mediumCount: Uint8 // technically a Uint8, but encoded as a normal CBOR varint
  medium: ByteString<32>
  smallCount: Uint8
  small: ByteString<32>
}
```

## Algorithms

### Key Derivation

The skip ratchet cannot be used as-is for an AES-256 key, because it's bigger than 256 bits.
To derive a key suitable for AES-256 usage from a skip ratchet, XOR the large, medium and small parts of the skip ratchet:

```typescript
function deriveKey(ratchet: SkipRatchet): ByteArray<32> {
  // '^' is meant to be the XOR operator on ByteArrays of equal size
  return ratchet.large ^ ratchet.medium ^ ratchet.small
}
```

### Increasing

![A diagram of a skip ratchet stepping/skipping with three binary digits](/images/skip_ratchet.png)

> A diagram of a skip ratchet stepping/skipping with three binary digits
> The skip ratchet in WNFS also has three digits, but they're base 256, not base 2.

The skip ratchet can be stepped forward, but not backwards.

Forward stepping can happen at
- the small digit, in increments of one,
- the small digit, in increments of one
- the medium digit, in increments up to 256
- the large digit, in increments up to 65536

These intervals for each digit are often referred to as small, medium or large "epochs". The reason that it isn't exactly 256 and 65536 is because each ratchet can only be incremented to the beginning of their next epoch.

These intervals for each digit are often referred to as small, medium or large "epochs".

Skipping to the next small epoch will always be a step of size 1.

The next small epoch is computed by replacing `small` with its hash and increasing `smallCount`.

This kind of step is only valid if the `smallCount` is below 255. Once the `smallCount` would hit 256, you need to perform a skip to the next medium epoch.

Skipping to the next medium epoch can also be though of as a "carry-over" operation in the digit interpretation.

The next medium epoch is computed by
- replacing `small` with the hash of the ones-complement of the current `medium`
- replacing `medium` with its hash
- resetting `smallCount` to 0
- increasing `mediumCount` by one.

This kind of step is only valid if the `mediumCount` is below 255. Once the `mediumCount` would hit 256, you need to perform a skip to the next large epoch.

The next large epoch is computed by
- replacing `medium` with the hash of the ones-complement of the current `large`
- replacing `small` with the hash of the ones-complement of the previously computed `medium`
- replacing `large` with the hash of the current `large`
- resetting `smallCount` to 0
- resetting `mediumCount` to 0

Increasing a ratchet by one may thus involve not only a skip to the next small epoch, but a skip to the next medium or large epoch. Precisely, if the `mediumCount` and `smallCount` are 255, then it involves a skip to the next large epoch. If only the `smallCount` is 255, it involves a skip to the next medium epoch. In all other cases it only involves a skip to the next epoch.

# Skip Ratchet

The skip ratchet is described in this paper: https://github.com/fission-suite/skip-ratchet-paper/blob/main/spiral-ratchet.pdf

## Encoding

This specifies the field names for the skip ratchet when encoded as CBOR:

```typescript
type SkipRatchet = {
  large: ByteString<32>
  mediumCount: Uint8
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


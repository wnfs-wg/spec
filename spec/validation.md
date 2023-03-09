# Notes on Validation

This document contains specifications on how implementations should validate incoming WNFS data.

The goals with these extra validation steps are:
- Ensure consistency between different specification-adhering implementations. If two implementations parse the same data and are both spec-adherent, they should either show the same data to the user or at least one of them errors out.
- Keep enough extensibility in the specification format for it to eventually evolve independently from the spec.


## Enum Variant Validation

In WNFS data, enum variants are encoded via an object wrapper with a single key that identifies its variant, as in this simplified example:

```typescript
type Node = File | Dir

type File = {
  "file": {
    // file data
  }
}

type Dir = {
  "dir": {
    // dir data
  }
}
```

Reasons for preferring this format are described in [`rationale/encoding.md`](/rationale/encoding.md#enum-variant-signaling).

After parsing an enum variant from an object an implementation MUST check that the wrapper object contains only a single entry and error out otherwise.
If the variant identifier is unknown to the implementation, the implementation MUST error out.


## Duplicate Object Keys

The CBOR spec allows an object containing key-value pairs with the same key multiple times.
This is disallowed by DAG-CBOR. WNFS implementations MUST error out if they detect a key being assigned twice in the same object.


## Unexpected Object Keys

Although one version of WNFS has a pre-defined schema, with a well-defined function determining whether a key in an object is expected or not, implementations MAY allow additional unexpected keys without erroring out.

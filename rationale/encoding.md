This documents our reasoning for current choices of encoding and potential alternatives.

# Enum Variant Signalling

## Need

In various places, WNFS data structures are allowed to point to different types of data, e.g. a directory entry may point to yet another directory or a file instead. Encoding a datum that allows a deserialization procedure to find out which deserialization subprocedure to use is usually called signalling.

## Choice

Signalling works by wrapping the data DAG-CBOR with an object containing a single key like this:

```typescript
type Node = File | Dir

type File = {
  "file": { /* ... */ }
}

type Dir = {
  "dir": { /* ... */ }
}
```

The key then indicates which variant you're supposed to decode at the value.

Some advantages of doing this include:
- In encodings like DAG-CBOR, when an object wraps some data, the object's key always precedes the data value in the encoded byte sequence. This means that a procedure can prepare the appropriate subprocedure to decode the value in advance.
- It spares some bytes compared to alternatives: In DAG-CBOR, it only requires a single byte signalling the wrapping object, and additionally some bytes for the string identifier.
- Using a string key as signalling allows adding future variants or even independently developed extensions with minimal chance that these extensions accidentally reuse the same signalling bytes for different data.

## Alternatives

### `type` key on object

In previous versions of WNFS, we've encoded a string at a known key (`type`) in an object like this:

```typescript
type Node = File | Dir

type File = {
  type: "file"
  // ...
}

type Dir = {
  type: "dir"
  // ...
}
```

This seems natural to lots of developers and is used for signalling in lots of other places, however, it has some downsides:
- When encoded as DAG-CBOR, the keys are sorted, thus the `type` key may not appear first, or even last. A deserialization procedure now has to cache the values of all other keys before it knows what special deserialization procedure to use. This is not memory or performance efficient.
- Even though some programming languages such as javascript or python allow (and/or encourage) you to deserialize DAG-CBOR into dynamic objects that can be worked on directly, in other programming languages like rust, C++ or most statically type languages one usually doesn't work with dynamic objects and instead deserializes into memory-efficient data structures. Libraries like `serde` in rust can't efficiently support deserializing objects that use signalling like this, preventing performance techniques like zero-copy deserialization.

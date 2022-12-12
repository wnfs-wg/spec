# Exchange Key Location

## Need

We need to store a list of all exchange keys, publicly readable, attached to a user's file system.

## Choice

We've decided to put this list under `exchange` at the root of the file system.

## Alternatives Considered

We've previously put these exchange keys at `/public/.well-known/exchange/*`. However, it may not be clear to users that giving another person or app write access to the root of the file system allows them to receive private files shared with this user.

Extracting these keys into its own section avoids this & encourages implementations to use a special UCAN capability for managing write access to this list of keys.

# RSA Keys

## Need

We need a way to deposit secrets for a known user non-interactively.

## Choice

We've decided to use asymmetric RSA encryption ([RSAES-OAEP](https://datatracker.ietf.org/doc/html/rfc3447#section-7.1)) mainly because it is possible to use non-extractable RSA keys in the WebCrypto API, thus keeping these keys away from potentially malicious extensions or app updates.

## Alternatives Considered

### ECDH

The advantages ought to be:
- Authentication of the sender
- Smaller key sizes

We don't need authentication for shared private files, as that's already covered by file system modification authorization systems outside WNFS (e.g. through UCANs).

It also complicates looking up deposited messages for receivers, since they need to go through all combinations of sender & receiver public keys.

### [RSA KEM-DEM](https://www.rfc-editor.org/rfc/rfc5990.html#appendix-A)

In contrast to RSAES-OAEP, this is a padding-less encryption scheme, which makes implementations easier and is supposedly faster.

However, as browsers are one of our target environments and only RSA-OAEP is currently widely supported in WebCrypto, we can't use RSA KEM-DEM.

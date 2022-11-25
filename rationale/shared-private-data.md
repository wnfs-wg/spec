# Exchange Key Location

## Need

We need to store a list of all exchange keys, publicly readable, attached to a user's file system.

## Choice

We've decided to put this list under `exchange` at the root of the file system.

## Alternatives

We've previously put these exchange keys at `/public/.well-known/exchange/*`. However, it may not be clear to users that giving another person or app write access to the root of the file system allows them to receive private files shared with this user.

Extracting these keys into its own section avoids this & encourages implementations to use a special UCAN capability for managing write access to this list of keys.

# RSA Keys

TODO (Essentially 'Because WebCrypto & non-extractability')

# Fernet Spec

This document describes version 0x80 (currently the only
version) of the fernet format.

Conceptually, fernet takes a user-provided *message* (an arbitrary
sequence of bytes), a *key* (32 bytes of entropy), and the current
time, and produces a *token*, which contains the message in a form
that can't be read or altered without the key.

To facilitate convenient interoperability, this spec defines the
external format of both tokens and keys.

## Key Format

A fernet *key* is the URL-safe base-64 encoding of the following
fields:

    Signing-key + Encryption-key

- *Signing-key*, 16 bytes
- *Encryption-key*, 16 bytes

## Token Format

A fernet *token* is the URL-safe base-64 encoding of the
concatenation of the following fields:

    Version + Timestamp + IV + Ciphertext + HMAC

- *Version*, 1 byte
- *Timestamp*, 8 bytes
- *IV*, 16 bytes
- *Ciphertext*, variable length, multiple of 16 bytes
- *HMAC*, 32 bytes

Fernet tokens are not self-delimiting. It is assumed that the
transport will provide a means of finding the length of each
complete fernet token.

## Token Fields

### Version

This field denotes which version of the format is being used by
the token. Currently there is only one version defined, with the
value 128 (0x80).

### Timestamp

This field is a 64-bit unsigned big-endian integer. It records the
number of seconds elapsed between January 1, 1970 UTC and the time
the token was created.

### IV

The 16-byte Initialization Vector used in AES encryption and
decryption of the Ciphertext.

When generating new fernet tokens, the IV must be chosen uniquely
for every token. Random selection will do this with high
probability.

### Ciphertext

This field has variable size, but is always a multiple of 16
bytes, the AES block size. It contains the original input message,
padded and encrypted.

### HMAC

This field is the 32-byte SHA256 HMAC, under signing-key, of the
concatenation of the following fields:

    Version + Timestamp + IV + Ciphertext

Note that the HMAC input is the entire rest of the token verbatim,
and that this input is not encoded with base 64.

## Generating

Given a key and message, generate a fernet token with the
following steps, in order:

1. Record the current time for the timestamp field.
2. Choose a unique IV.
3. Construct the ciphertext:
   1. Pad the message to a multiple of 16 bytes using PKCS #7
   standard block padding.
   2. Encrypt the padded message using AES-128 with the chosen IV
   and user-supplied encryption-key.
4. Compute the HMAC field as described above using the
user-supplied signing-key.
5. Concatenate all fields together in the format above.
6. Base-64 encode the entire token.

## Verifying

Given a key and token, to verify that the token is valid and
recover the original message, perform the following steps, in
order:

1. Base-64 decode the token.
2. Recompute the HMAC from the other fields and the user-supplied
signing-key.
3. Ensure the recomputed HMAC matches the HMAC field stored in the
token, using a constant-time comparison function.
4. If the user has specified a maximum age (or "time-to-live") for
the token, ensure the recorded timestamp is not too far in the
past.
5. Decrypt the ciphertext field using the IV and user-supplied
encryption-key.
6. Unpad the decrypted plaintext, yielding the original message.

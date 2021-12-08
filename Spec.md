# Fernet Spec

This document describes version 0x80 (currently the only
version) of the fernet format.

Conceptually, fernet takes a user-provided *message* (an arbitrary
sequence of bytes), a *key* (256, 384 or 512 bits), and the current
time, and produces a *token*, which contains the message in a form
that can't be read or altered without the key.

To facilitate convenient interoperability, this spec defines the
external format of both tokens and keys.

Encryption in this version is done with AES 128, 192 or 256 in CBC mode
depending on selection of the key length. AES 128 requires a 256-bit (32 byte)
key, AES192 requires a 384-bit (48 byte) key and AES 256 requires a
512-bit (64 byte) key.

All base 64 encoding is done with the "URL and Filename Safe"
variant, defined in [RFC 4648](http://tools.ietf.org/html/rfc4648#section-5) as "base64url".

## Key Format

A fernet *key* is the base64url encoding of the following
fields:

    Signing-key ‖ Encryption-key

- *Signing-key*, 128, 192 or 256 bits
- *Encryption-key*, the same key length as *Signing-key*

## Token Format

A fernet *token* is the base64url encoding of the
concatenation of the following fields:

    Version ‖ Timestamp ‖ IV ‖ Ciphertext ‖ HMAC

- *Version*, 8 bits
- *Timestamp*, 64 bits
- *IV*, 128 bits
- *Ciphertext*, variable length, multiple of 128 bits
- *HMAC*, 256 bits

Fernet tokens are not self-delimiting. It is assumed that the
transport will provide a means of finding the length of each
complete fernet token.

## Token Fields

### Version

This field denotes which version of the format is being used by
the token. Currently there are three versions defined:

- value 128 (0x80) indicating AES 128 encryption is used
- value 160 (0xA0) indicating AES 192 encryption is used
- value 192 (0xC0) indicating AES 256 encryption is used

### Timestamp

This field is a 64-bit unsigned big-endian integer. It records the
number of seconds elapsed between January 1, 1970 UTC and the time
the token was created.

### IV

The 128-bit Initialization Vector used in AES encryption and
decryption of the Ciphertext.

When generating new fernet tokens, the IV must be chosen uniquely
for every token. With a high-quality source of entropy, random
selection will do this with high probability.

### Ciphertext

This field has variable size, but is always a multiple of 128
bits, the AES block size. It contains the original input message,
padded and encrypted.

### HMAC

This field is the 256-bit SHA256 HMAC, under signing-key, of the
concatenation of the following fields:

    Version ‖ Timestamp ‖ IV ‖ Ciphertext

Note that the HMAC input is the entire rest of the token verbatim,
and that this input is *not* base64url encoded.

## Generating

Given a key and message, generate a fernet token with the
following steps, in order:

1. Record the current time for the timestamp field.
2. Choose a unique IV.
3. Construct the ciphertext:
   1. Pad the message to a multiple of 16 bytes (128 bits) per [RFC
   5652, section 6.3](http://tools.ietf.org/html/rfc5652#section-6.3).
   This is the same padding technique used in [PKCS #7
   v1.5](http://tools.ietf.org/html/rfc2315#section-10.3) and all
   versions of SSL/TLS (cf. [RFC 5246, section
   6.2.3.2](http://tools.ietf.org/html/rfc5246#section-6.2.3.2) for
   TLS 1.2).
   2. Encrypt the padded message using AES in CBC mode with
   the chosen IV and user-supplied key-length and encryption-key.
4. Compute the HMAC field as described above using the
user-supplied key-length and signing-key.
5. Concatenate all fields together in the format above.
6. base64url encode the entire token.

## Verifying

Given a key and token, to verify that the token is valid and
recover the original message, perform the following steps, in
order:

1. base64url decode the token.
2. Ensure the first byte of the token is 0x80.
3. If the user has specified a maximum age (or "time-to-live") for
the token, ensure the recorded timestamp is not too far in the
past.
4. Recompute the HMAC from the other fields and the user-supplied
signing-key.
5. Ensure the recomputed HMAC matches the HMAC field stored in the
token, using a constant-time comparison function.
6. Decrypt the ciphertext field using AES in CBC mode with the
recorded IV and user-supplied key-length and encryption-key.
7. Unpad the decrypted plaintext, yielding the original message.

ursa
====

This Node module provides wrappers for the RSA public/private key
crypto functionality of OpenSSL.

This module was inspired by
[node-rsa](https://github.com/chrisa/node-rsa) by Chris Andrews. To
be clear, there are a few lines that I (Danfuzz) used from its
`wscript` build file, but other than that this code is new.

Installing
----------

```shell
npm install ursa
```

Or grab the source and

```shell
node-waf configure build
```

Testing
-------

```shell
node ./test/test.js
```

Usage
-----

This library aims to be convenient to use, allowing one to pass in and
get back regular string objects. However, it is also meant to be reasonably
easy to use efficiently, allowing one to pass in and get back Buffer
objects. Using Buffers is always the more efficient option.

All methods that can deal with strings take one or more arguments indicating
the encoding to use when interpreting an argument or generating a result.
These are limited to the usual encoding names that are valid for use with
Buffers: `base64` `binary` `hex` and `utf8`. If an encoding is left undefined
and the argument is a string, then the encoding is *always* assumed to be
`utf8`. If an argument is a Buffer, then the encoding (if defined at all)
is ignored. An undefined output encoding is *always* interpreted as a request
for a Buffer result.

The library knows how to read and output PEM format files for both
public and private keys, and it can generate new private keys (aka
keypairs).

The usual public-encryption / private-decryption operations are always
done using padding mode `RSA_PKCS1_OAEP_PADDING`, which is the recommended
mode for all new applications (as of this writing). Note that this mode
builds-in a random element into every encryption operation, making it
unnecessary to waste time or effort adding randomness in at a higher layer.

The less well-understood private-encryption / public-decryption operations
(used for building signature mechanisms) are always done using padding
mode `RSA_PKCS1_PADDING`. This doesn't build in any randomness (but that's
not usually a problem for applications that use these operations).

See the doc comments and tests for the excruciating details, but here's
a quick rundown of the available top-level exports and instance methods:

Top-Level Exports
-----------------

### ursa.createPrivateKey(pem, password, encoding)

Create and return a private key (aka a keypair) read in from the given
PEM-format file.  If defined, the given password is used to decrypt
the PEM file.

The encoding, if specified, applies to both other arguments.

See "Public Key Methods" below for more details.

### ursa.createPublicKey(pem, encoding)

Create and return a public key read in from the given PEM-format file.
See "Public Key Methods" below for more details.

### ursa.createSigner(algorithm)

Create and return a signer which can sign a hash generated with the named
algorithm (such as `"sha256"` or `"md5"`). See "Signer Methods" below
for more details.

This function is similar to `crypto.createSign()`, except this function
takes a hash algorithm name (e.g., `"sha256"`) and not a crypto+hash name
combination (e.g., `"RSA_SHA256"`).

### ursa.createVerifier(algorithm)

Create and return a verifier which can verify a hash generated with the
named algorithm (such as `"sha256"` or `"md5"`). See "Verifier Methods" below
for more details.

This function is similar to `crypto.createVerify()`, except this function
takes a hash algorithm name (e.g., `"sha256"`) and not a crypto+hash name
combination (e.g., `"RSA_SHA256"`).

### ursa.generatePrivateKey(modulusBits, exponent)

Create and return a freshly-generated private key (aka a keypair).
The first argument indicates the number of bits in the modulus (1024
or more is generally considered secure). The second argument indicates
the exponent value, which must be odd (65537 is the typical value; 3
and 17 are also common).  Both arguments are optional and default to
2048 and 65537 (respectively).

Using the command-line `openssl` tool, this operation is
equivalent to:

```shell
openssl genrsa -out key-name.pem <modulusBits>
```

for exponent 65537, or for exponent 3 with the additional option
`-3`. (That tool doesn't support other exponents.)

### ursa.isKey(obj)

Return `true` if the given object is a key object (public or private) that
was created by this module. Return `false` if not.

### ursa.isPrivateKey(obj)

Return `true` if the given object is a private key object that
was created by this module. Return `false` if not.

### ursa.isPublicKey(obj)

Return `true` if the given object is a public key object that
was created by this module. Return `false` if not.

Note that, even though all the public key operations work on private
keys, this function only returns true if the given object is a
public key, per se.

### ursa.createVerifier(algorithm)

Create and return a verifier which can verify a hash generated with the
named algorithm (such as `"sha256"` or `"md5"`). See "Verifier Methods" below
for more details.

### ursa.sshFingerprint(sshKey, sshEncoding, outEncoding)

Return the SSH-style public key fingerprint of the given SSH-format
public key (which was, perhaps, the result of a call to
`toPublicSsh()` on a key object).

This is no more and no less than an MD5 hash of the given SSH-format
public key. This function doesn't actually check to see if the given
key is valid (garbage in, garbage out).

Using the command-line `ssh-keygen` tool, this operation is
equivalent to:

```shell
ssh-keygen -l -f key-name.sshpub
```

This operation is also equivalent to this:

```shell
cat key-name.sshpub | awk '{print $2}' | base64 --decode | md5
```

Public Key Methods
------------------

These are all the methods available on public keys. These methods are
*also* available on private keys (since private keys have all the
underlying data necessary to perform the public-side operations).

### encrypt(buf, bufEncoding, outEncoding)

This performs the "public encrypt" operation on the given buffer. The
result is always a byte sequence that is the same size as the key
associated with the instance. (For example, if the key is 2048 bits,
then the result of this operation will be 2048 bits, aka 256 bytes.)

The input buffer is limited to be no larger than the key size
minus 41 bytes.

This operation is always performed using padding mode
`RSA_PKCS1_OAEP_PADDING`.

### getExponent(encoding)

Get the public exponent as an unsigned big-endian byte sequence.

### getModulus(encoding)

Get the public modulus as an unsigned big-endian byte sequence.

### toPublicPem(encoding)

This converts the public key data into a PEM-format file.

### toPublicSsh(encoding)

This converts the public key data into an SSH-format file. This is the
file format one finds in SSH's `authorized_keys` and `known_hosts` files.
When used in such files, the contents are base64-encoded and prefixed with
the label `ssh-rsa`. Depending on context, the line a key appears on may
also have a host name prefix (in `known_hosts`) or comment suffix
(in `authorized_keys`).

Using the command-line `ssh-keygen` tool, this operation is equivalent to:

```shell
ssh-keygen -y -f key-name.pem > key-name.sshpub
```

### toPublicSshFingerprint(encoding)

Return the SSH-style public key fingerprint of this key. See
`ursa.sshFingerprint()`, above, for more details.

### publicDecrypt(buf, bufEncoding, outEncoding)

This performs the "public decrypt" operation on the given buffer. The
result is always a byte sequence that is no more than the size of the
key associated with the instance. (For example, if the key is 2048
bits, then the result of this operation will be no more than 2048
bits, aka 256 bytes.)

This operation is always performed using padding mode
`RSA_PKCS1_PADDING`.

### verify(algorithm, hash, sig, encoding)

This performs an RSA public-verify on the given hash buffer, which
should be the result of performing the hash operation named by
the algorithm (such as `"sha256"` or `"md5"`) on some data. The
signature buffer is checked to see if it contains a private-signed
statement of the algorithm and hash. The method returns `true` if
the signature and hash match, or `false` if the signature and hash
don't match but the signature is at least a valid signature of
some sort. In any other situation, this throws an exception.

The encoding, if specified, applies to both buffer-like arguments. The
algorithm must always be a string.

This method is the underlying one used as part of the implementation
of the higher-level and much friendlier `ursa.createVerifier()`.

### unbox(unboxer)

This is an internal method that is used in the implementation of
`ursa.isKey()` `ursa.isPrivateKey()` and `ursa.isPublicKey()`. When
called externally, it should always return `undefined`.


Private Key Methods
-------------------

These are the methods available on private keys, above and beyond
what is available for public keys.

### decrypt(buf, bufEncoding, outEncoding)

This performs the "private decrypt" operation on the given buffer. The
result is always a byte sequence that is no more than the size of the
key associated with the instance. (For example, if the key is 2048
bits, then the result of this operation will be no more than 2048
bits, aka 256 bytes.)

This operation is always performed using padding mode
`RSA_PKCS1_OAEP_PADDING`.

### toPrivatePem(encoding)

This converts the private key data into a PEM-format file. The result
is not encrypted, so it behooves the user of this method to take care
with the result if the key is sensitive from a security standpoint,
which is often the case with such things. (YMMV of course.)

### privateEncrypt(buf, bufEncoding, outEncoding)

This performs the "private encrypt" operation on the given buffer. The
result is always a byte sequence that is the same size as the key
associated with the instance. (For example, if the key is 2048 bits,
then the result of this operation will be 2048 bits, aka 256 bytes.)

The input buffer is limited to be no larger than the key size
minus 12 bytes.

This operation is always performed using padding mode
`RSA_PKCS1_PADDING`.

### sign(algorithm, hash, hashEncoding, outEncoding)

This performs an RSA private-sign on the given hash buffer, which
should be the result of performing the hash operation named by
the algorithm (such as `"sha256"` or `"md5"`) on some data. The
result of this operation may later be passed to `verify()` on the
corresponding public key.

This method is the underlying one used as part of the implementation
of the higher-level and much friendlier `ursa.createSigner()`.

Signer Methods
--------------

These are the methods available on signer objects, which are returned
by `ursa.createSigner()`. These are similar to the objects returned
from `crypto.createSign()`.

### update(buf, bufEncoding)

Update the hash in-progress with the given data.

### sign(privateKey, outEncoding)

Get the final hash of the data, and sign it using the private key. The
return value is the signature, suitable for later verification.


Verifier Methods
----------------

These are the methods available on verifier objects, which are returned
by `ursa.createVerifier()`. These are similar to the objects returned
from `crypto.createVerify()`.

### update(buf, bufEncoding)

Update the hash in-progress with the given data.

### verify(publicKey, sig, sigEncoding)

Get the final hash of the data, and verify that the given signature
both matches it and was produced by the private key corresponding to
the given public key.

This returns `true` if the signature and hash match appropriately,
or `false` if the signature appears to be generally valid (e.g.
structurally) yet doesn't match. This throws an exception in all
other cases.


Contributing
------------

Questions, comments, bug reports, and pull requests are all welcome.
Submit them at [the project on GitHub](https://github.com/Obvious/ursa/).

Bug reports that include steps-to-reproduce (including code) are the
best. Even better, make them in the form of pull requests that update
the test suite. Thanks!

Author
------

[Dan Bornstein](https://github.com/danfuzz)
([personal website](http://www.milk.com/)), supported by
[The Obvious Corporation](http://obvious.com/).

License
-------

Copyright 2012 [The Obvious Corporation](http://obvious.com/).

Licensed under the Apache License, Version 2.0. 
See the top-level file `LICENSE.txt` and
(http://www.apache.org/licenses/LICENSE-2.0).


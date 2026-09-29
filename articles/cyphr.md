# Introduction

This package tries to smooth over some of the differences in encryption
approaches (symmetric vs. asymmetric, sodium vs. openssl) to provide a
simple interface for users who just want to encrypt or decrypt things.

The scope of the package is to protect data that has been saved to disk.
It is not designed to stop an attacker targeting the R process itself to
determine the contents of sensitive data. The package does try to
prevent you accidentally saving to disk the contents of sensitive
information, including the keys that could decrypt such information.

This vignette works through the basic functionality of the package. It
does not offer much in the way of an introduction to encryption itself;
for that see the excellent vignettes in the `openssl` and `sodium`
packages (see `vignette("crypto101")` and `vignette("bignum")` for
information about how encryption works). This package is a wrapper
around those packages in order to make them more accessible.

## Keys and the like

To encrypt anything we need a key. There are two sorts of key “types” we
will concern ourselves with here “symmetric” and “asymmetric”.

- “symmetric” keys are used for storing secrets that multiple people
  need to access. Everyone has the same key (which is just a bunch of
  bytes) and with that we can either encrypt data or decrypt it.

- a “key pair” is a public and a private key; this is used in
  communication. You hold a private key that nobody else ever sees and a
  public key that you can copy around all over the show. These can be
  used for a couple of different patterns of communication (see below).

We support symmetric keys and asymmetric key pairs from the `openssl`
and `sodium` packages (which wrap around industry-standard cryptographic
libraries) - this vignette will show how to create and load keys of
different types as they’re used.

The `openssl` keys have the advantage of a standard key format, and that
many people (especially on Linux and macOS) have a keypair already (see
below if you’re not sure if you do). The `sodium` keys have the
advantage of being a new library, starting from a clean slate rather
than carrying with it accumulated ideas from the last 20 years of
development.

The idea in `cyphr` is that we can abstract away some differences in the
types of keys and the functions that go with them to create a
standardised interface to encrypting and decrypting strings, R objects,
files and raw vectors. With that, we can then create wrappers around
functions that create files and simplify the process of adding
encryption into a data workflow.

Below, I’ll describe the sorts of keys that `cyphr` supports and in the
sections following describe how these can be used to actually do some
encryption.

### Symmetric encryption

![Illustration of Symmetric Encryption](symmetric.png)

Illustration of Symmetric Encryption

This is the simplest form of encryption because everyone has the same
key (like a key to your house or a single password). This raises issues
(like how do you *store* the key without other people reading it) but we
can deal with that below.

#### `openssl`

To generate a key with `openssl`, you can use:

``` r

k <- openssl::aes_keygen()
```

which generates a raw vector

``` r

k
```

    ## aes 05:7d:89:66:07:f1:29:21:71:ff:a2:22:7e:98:d7:0b

(this prints nicely but it really is stored as a 16 byte raw vector).

The encryption functions that this key supports are
[`openssl::aes_cbc_encrypt`](https://jeroen.r-universe.dev/openssl/reference/aes_cbc.html),
[`openssl::aes_ctr_encrypt`](https://jeroen.r-universe.dev/openssl/reference/aes_cbc.html)
and
[`openssl::aes_gcm_encrypt`](https://jeroen.r-universe.dev/openssl/reference/aes_cbc.html)
(along with the corresponding decryption functions). The `cyphr` package
tries to abstract this away by using a wrapper \`cyphr::key_openssl

``` r

key <- cyphr::key_openssl(k)
key
```

    ## <cyphr_key: openssl>

With this key, one can encrypt a string with
[`cyphr::encrypt_string`](https://docs.ropensci.org/cyphr/reference/encrypt_data.md):

``` r

secret <- cyphr::encrypt_string("my secret string", key)
```

and decrypt it again with
[`cyphr::decrypt_string`](https://docs.ropensci.org/cyphr/reference/encrypt_data.md):

``` r

cyphr::decrypt_string(secret, key)
```

    ## [1] "my secret string"

See below for more functions that use these key objects.

#### `sodium`

The interface is almost identical using sodium symmetric keys. To
generate a symmetric key with libsodium you would use
[`sodium::keygen`](https://docs.ropensci.org/sodium/reference/keygen.html)

``` r

k <- sodium::keygen()
```

This is really just a raw vector of length 32, without even any class
attribute!

The encryption functions that this key supports are
[`sodium::data_encrypt`](https://docs.ropensci.org/sodium/reference/symmetric.html)
and
[`sodium::data_decrypt`](https://docs.ropensci.org/sodium/reference/symmetric.html).
To create a key for use with `cyphr` that knows this, use:

``` r

key <- cyphr::key_sodium(k)
key
```

    ## <cyphr_key: sodium>

This key can then be used with the high-level cyphr encryption functions
described below.

### Asymmetric encryption (“key pairs”)

![Illustration of Asymmetric Encryption](asymmetric.png)

Illustration of Asymmetric Encryption

With asymmetric encryption everybody has two keys that differ from
everyone else’s key. One key is public and can be shared freely with
anyone you would like to communicate with and the other is private and
must never be disclosed.

In the `sodium` package there is a vignette (`vignette("crypto101")`)
that gives a gentle introduction to how this all works. In practice, you
end up creating a pair of keys for yourself. Then to encrypt or decrypt
something you encrypt messages with the recipient’s *public key* and
they (and only they) can decrypt it with their *private key*.

One use for asymmetric encryption is to encrypt a shared secret (such as
a symmetric key) - with this you can then safely store or communicate a
symmetric key without disclosing it.

#### `openssl`

Let’s suppose that we have two parties “Alice” and “Bob” who want to
talk with one another. For demonstration purposes we need to generate
SSH keys (with no password) in temporary directories (to comply with
CRAN policies). In a real situation these would be on different machines
(Alice has no access to Bob’s key!) and these keys would be password
protected.

``` r

path_key_alice <- cyphr::ssh_keygen(password = FALSE)
path_key_bob <- cyphr::ssh_keygen(password = FALSE)
```

Note that each directory contains a public key (`id_rsa.pub`) and a
private key (`id_rsa`).

``` r

dir(path_key_alice)
```

    ## [1] "id_rsa"     "id_rsa.pub"

``` r

dir(path_key_bob)
```

    ## [1] "id_rsa"     "id_rsa.pub"

Below, the full path to the key (e.g., `.../id_rsa`) could be used in
place of the directory name if you prefer.

If Alice wants to send a message to Bob she needs to use her private key
and his public key

``` r

pair_a <- cyphr::keypair_openssl(path_key_bob, path_key_alice)
pair_a
```

    ## <cyphr_keypair: openssl>

with this pair she can write a message to “bob”:

``` r

secret <- cyphr::encrypt_string("secret message", pair_a)
```

The secret is now just a big pile of bytes

``` r

secret
```

    ##   [1] 58 0a 00 00 00 03 00 04 06 00 00 03 05 00 00 00 00 05 55 54 46 2d 38 00 00
    ##  [26] 02 13 00 00 00 04 00 00 00 18 00 00 00 10 ad d5 c5 14 9d 30 14 07 eb 8e 48
    ##  [51] 71 75 90 b4 1f 00 00 00 18 00 00 01 00 6b 1c c8 a0 b7 63 d7 aa f8 fd fa 7e
    ##  [76] 51 d5 cc 4c c9 06 6d dd 6d bf 95 e1 21 53 85 bd a2 09 70 b2 7a 5f 14 bf f4
    ## [101] a5 55 8c c7 56 e6 a3 8f a8 03 fe 4d dd b7 35 97 45 14 07 cc 43 2d e5 8e 78
    ## [126] a1 8f bf 14 7f 10 23 35 c8 98 6b 87 94 8a 9a 5e 98 29 96 d8 1e 01 bd 50 66
    ## [151] 37 fc 56 aa 9c 85 8e 93 96 c2 01 d4 a0 3a fb 81 92 10 d8 aa c6 ad 36 19 9b
    ## [176] 38 bd 17 7f a0 0a 7b 26 6b d9 ca 22 79 c7 09 c8 8a 41 b9 e4 cc 80 7d 8f 1e
    ## [201] 7b ab 12 06 34 96 07 f1 da 85 ae 81 4d 44 bc 6f fc ad 8b 7b 24 fa 62 b0 af
    ## [226] a0 13 2d 3a a4 f6 6f 79 f3 a0 2a a9 f7 33 30 66 98 5f f8 d1 7e 69 e3 76 8d
    ## [251] 76 39 91 8b 8f 34 88 92 1b f8 bd eb a1 a2 ea a4 ab 91 43 f4 1d cd 11 52 3c
    ## [276] cf 35 9c 28 a8 cd 9a 9f 33 e1 8a ef ae f8 f9 73 61 e7 23 5a 8e 25 a8 66 ed
    ## [301] aa 1c 89 3a 3e 89 05 f0 d9 b5 88 3d c1 b9 27 a1 9a 96 b1 00 00 00 18 00 00
    ## [326] 00 10 55 fc 11 6c d8 d2 82 bc 3c ac ca bc df 5b fe ce 00 00 00 18 00 00 01
    ## [351] 00 0b 38 0c 59 10 a2 a8 05 cf 54 72 3d bf bb bb 60 9c b7 97 4a 04 3b ac 3b
    ## [376] bc cc 7a ab 41 79 19 a7 17 e8 eb 02 41 0d d6 19 e6 50 6c d0 bb 7a 3e 0a 88
    ## [401] ec 4d 35 84 c4 93 81 d8 48 80 db a3 4d 76 5c 07 9b af 33 62 f8 9b 10 4a a0
    ## [426] aa 61 c4 33 63 54 96 65 70 d3 5e 44 88 02 ab 26 3d 46 bf 8d 09 bd a5 a0 17
    ## [451] 37 25 78 94 37 be 2a 02 42 7f 80 58 db a7 fe d6 f8 2f 62 6d 8d b2 b1 8e 4f
    ## [476] 0f 29 95 77 d1 6a c5 11 f1 27 b3 d1 3e ea b1 7b 26 12 68 ef b8 e7 a5 57 62
    ## [501] b7 cc d4 45 e7 01 33 57 2f 5a ed 22 16 80 24 65 bc 0e d8 4f 20 5d 60 ff fd
    ## [526] 65 22 ef ba 9f 3e 47 de 6a 49 37 10 2d 5c c8 30 aa d8 19 74 f5 7d 19 6d b0
    ## [551] b1 70 50 3c ed 87 e3 6a 3d a9 93 48 33 5d 16 05 0e a7 69 5c 31 9a 33 77 cf
    ## [576] cc bb 16 e5 a1 e2 5b 93 6f c0 1c 0d 67 4b 9b 34 00 98 24 37 5f 7f b3 95 d0
    ## [601] a3 df 0e 3c b5 4d af 00 00 04 02 00 00 00 01 00 04 00 09 00 00 00 05 6e 61
    ## [626] 6d 65 73 00 00 00 10 00 00 00 04 00 04 00 09 00 00 00 02 69 76 00 04 00 09
    ## [651] 00 00 00 07 73 65 73 73 69 6f 6e 00 04 00 09 00 00 00 04 64 61 74 61 00 04
    ## [676] 00 09 00 00 00 09 73 69 67 6e 61 74 75 72 65 00 00 00 fe

Note that unlike symmetric encryption above, Alice cannot decrypt her
own message:

``` r

cyphr::decrypt_string(secret, pair_a)
```

    ## Error in `openssl::decrypt_envelope()`:
    ## ! OpenSSL error: 00588E98437F0000:error:03000082:digital envelope routines:EVP_CIPHER_CTX_set_key_length:invalid key length:../crypto/evp/evp_enc.c:1046:

For Bob to read the message, he uses his private key and Alice’s public
key (which she has transmitted to him previously).

``` r

pair_b <- cyphr::keypair_openssl(path_key_alice, path_key_bob)
```

With this keypair, Bob can decrypt Alice’s message

``` r

cyphr::decrypt_string(secret, pair_b)
```

    ## [1] "secret message"

And send one back of his own:

``` r

secret2 <- cyphr::encrypt_string("another message", pair_b)
secret2
```

    ##   [1] 58 0a 00 00 00 03 00 04 06 00 00 03 05 00 00 00 00 05 55 54 46 2d 38 00 00
    ##  [26] 02 13 00 00 00 04 00 00 00 18 00 00 00 10 01 01 83 b8 0f 59 18 36 f1 7f bd
    ##  [51] 17 5c f7 e0 8f 00 00 00 18 00 00 01 00 19 33 fd 3d 5e 63 d3 50 a0 36 e8 65
    ##  [76] a9 04 6e d9 3a 63 0b c0 da a4 89 50 90 08 cf 46 07 a4 3f 78 01 bf fc ba 43
    ## [101] 60 3e f3 94 2e 6e ab 9d 77 6d 69 ec ce 74 d6 81 00 aa 1d 86 de 50 84 1b 53
    ## [126] 20 be 6a 7f 04 c9 ff 8b 1a 88 65 4f 17 56 3b 29 d1 73 f3 94 c9 77 11 b8 ec
    ## [151] d6 df 31 1c 72 2b f6 ff ba 2a 4a d9 af 3a 1d 5c 40 6a 01 dd 84 a9 50 7b 02
    ## [176] fb b8 3d d8 e7 c8 05 ce a3 0a 8f 1b 20 62 f0 35 d6 d9 27 a2 0c c7 0a 48 10
    ## [201] 15 7f d2 15 bc a5 d2 5b 4b 6b b2 c0 bb 57 0b f2 df 82 a9 ff ec 0d 5d 8e 29
    ## [226] 54 90 f0 64 15 f3 44 42 36 9c 87 60 54 37 54 fc 20 28 84 ab e0 df 77 0d 9f
    ## [251] fa 34 bc 0d d0 41 97 f5 df 38 35 38 b3 2f ff 78 ed bc cf 30 a4 4c 6a 13 9c
    ## [276] 23 82 a9 6a a3 a2 06 6d 06 15 07 1e ef 8d f7 1f 69 c1 2a 96 89 83 48 c6 e6
    ## [301] 0e a3 c0 90 8f 1f 80 f2 44 08 1d c8 05 31 d1 ab dd bb 13 00 00 00 18 00 00
    ## [326] 00 10 3e 0c 00 8a 37 80 24 6f 4d f9 c0 00 b6 2f a8 0e 00 00 00 18 00 00 01
    ## [351] 00 4d d5 ca 56 4d 2e fe fa 21 84 af a5 3f fb 8a 4b b3 e7 46 a5 ac de a8 57
    ## [376] 4a e6 15 e7 07 b5 96 1b b9 83 4e 2f 68 32 c0 7c c5 c3 76 75 69 a0 a7 c2 bd
    ## [401] 37 c3 7a 5b fe 56 22 87 9f 3b b7 e1 7d f3 cf af 27 de 22 45 40 15 c6 05 2d
    ## [426] ef d8 76 40 af a8 29 97 6a 78 75 c0 a4 d3 bd a9 d4 fb fd 6b 5d d8 d2 5c ec
    ## [451] fb da 7a 94 e0 9e 53 42 05 0e 23 e6 c9 85 ca c2 be 0c ed 1c 0c 84 23 14 84
    ## [476] 9c 0e b6 c4 eb 10 14 f6 8f a5 45 c9 44 16 9d 20 a6 bc 3b 9f ee 8e d8 a1 54
    ## [501] 3b 4a 0f be f0 61 d8 1f 67 e3 df ac 8e 45 28 a3 ae cd a6 a7 b9 21 97 58 f2
    ## [526] 71 17 7a 6c 45 b6 59 40 a1 07 42 3f c7 10 41 89 86 cb af 0f 7e d3 dd af c9
    ## [551] 9a d4 c4 33 4d ee 7a 7f 68 41 d5 b9 0a 8f a0 5c d2 0c 34 13 2e af c8 3d 91
    ## [576] 22 d1 57 be b0 cb b1 6c bf 17 4d 82 c5 1a c3 bc 8e e1 8f 4b 00 ae 04 26 d0
    ## [601] 67 ec 33 62 03 5e 34 00 00 04 02 00 00 00 01 00 04 00 09 00 00 00 05 6e 61
    ## [626] 6d 65 73 00 00 00 10 00 00 00 04 00 04 00 09 00 00 00 02 69 76 00 04 00 09
    ## [651] 00 00 00 07 73 65 73 73 69 6f 6e 00 04 00 09 00 00 00 04 64 61 74 61 00 04
    ## [676] 00 09 00 00 00 09 73 69 67 6e 61 74 75 72 65 00 00 00 fe

which she can decrypt

``` r

cyphr::decrypt_string(secret2, pair_a)
```

    ## [1] "another message"

Chances are, you have an openssl keypair in your `.ssh/` directory. If
so, you would pass `NULL` as the path for the private (or less usefully,
the public) key pair part. So to send a message to Bob, we’d include the
path to Bob’s public key.

``` r

pair_us <- cyphr::keypair_openssl(path_key_bob, NULL)
```

This all skips over how Alice and Bob will exchange this secret
information. Because the secret is bytes, it’s a bit odd to work with.
Alice could save the secret to disk with

``` r

secret <- cyphr::encrypt_string("secret message", pair_a)
path_for_bob <- file.path(tempdir(), "for_bob_only")
writeBin(secret, path_for_bob)
```

And then send Bob the file `for_bob_only` (over email or any other
insecure medium).

and bob could read the secret in with:

``` r

secret <- readBin(path_for_bob, raw(), file.size(path_for_bob))
cyphr::decrypt_string(secret, pair_b)
```

    ## [1] "secret message"

As an alternative, you can “base64 encode” the bytes into something that
you can just email around:

``` r

secret_base64 <- openssl::base64_encode(secret)
secret_base64
```

    ## [1] "WAoAAAADAAQGAAADBQAAAAAFVVRGLTgAAAITAAAABAAAABgAAAAQUSMZGHFL9wE6iTbF+8oMbwAAABgAAAEAh4ZuVjUmOK92TIWsZcPct4j7zhMPEOCYj6qM+GD5adsVMB6KxXtS4o+h38zKDZF4QnqQlzOc+WEMoFk1FGbtUy+9S+ULvy4TZWFx8BWltz3lWCqWHYCimRgoyWv1Tn8K6MZXBj90+MZ4yYmQ5Y+6+qEuCyzypR/LA5m5ucYMywU9z6FRkyJdkKkutYN72UxgDpDzGuMLcBeYQVoD8WFPvDYaT7LdXOgeQUaixLovfBiF5DBHAW0JdYLZjT4SvCCCLE2x+LUAVdmb0ahZmkG0Ik24YrxgruYCB0LbPRA2qoKsQ+ksqPHPhb5+QptDREYmQjRfZXIQSJVeYzVypTI+WgAAABgAAAAQYOTVIoJ9eJ0jUIDyJ174IAAAABgAAAEACzgMWRCiqAXPVHI9v7u7YJy3l0oEO6w7vMx6q0F5GacX6OsCQQ3WGeZQbNC7ej4KiOxNNYTEk4HYSIDbo012XAebrzNi+JsQSqCqYcQzY1SWZXDTXkSIAqsmPUa/jQm9paAXNyV4lDe+KgJCf4BY26f+1vgvYm2NsrGOTw8plXfRasUR8Sez0T7qsXsmEmjvuOelV2K3zNRF5wEzVy9a7SIWgCRlvA7YTyBdYP/9ZSLvup8+R95qSTcQLVzIMKrYGXT1fRltsLFwUDzth+NqPamTSDNdFgUOp2lcMZozd8/MuxbloeJbk2/AHA1nS5s0AJgkN19/s5XQo98OPLVNrwAABAIAAAABAAQACQAAAAVuYW1lcwAAABAAAAAEAAQACQAAAAJpdgAEAAkAAAAHc2Vzc2lvbgAEAAkAAAAEZGF0YQAEAAkAAAAJc2lnbmF0dXJlAAAA/g=="

This can be converted back with
[`openssl::base64_decode`](https://jeroen.r-universe.dev/openssl/reference/base64_encode.html):

``` r

identical(openssl::base64_decode(secret_base64), secret)
```

    ## [1] TRUE

Or, less compactly but also suitable for email, you might just convert
the bytes into their hex representation:

``` r

secret_hex <- sodium::bin2hex(secret)
secret_hex
```

    ## [1] "580a000000030004060000030500000000055554462d380000021300000004000000180000001051231918714bf7013a8936c5fbca0c6f000000180000010087866e56352638af764c85ac65c3dcb788fbce130f10e0988faa8cf860f969db15301e8ac57b52e28fa1dfccca0d9178427a9097339cf9610ca059351466ed532fbd4be50bbf2e13656171f015a5b73de5582a961d80a2991828c96bf54e7f0ae8c657063f74f8c678c98990e58fbafaa12e0b2cf2a51fcb0399b9b9c60ccb053dcfa15193225d90a92eb5837bd94c600e90f31ae30b701798415a03f1614fbc361a4fb2dd5ce81e4146a2c4ba2f7c1885e43047016d097582d98d3e12bc20822c4db1f8b50055d99bd1a8599a41b4224db862bc60aee6020742db3d1036aa82ac43e92ca8f1cf85be7e429b4344462642345f65721048955e633572a5323e5a000000180000001060e4d522827d789d235080f2275ef82000000018000001000b380c5910a2a805cf54723dbfbbbb609cb7974a043bac3bbccc7aab417919a717e8eb02410dd619e6506cd0bb7a3e0a88ec4d3584c49381d84880dba34d765c079baf3362f89b104aa0aa61c4336354966570d35e448802ab263d46bf8d09bda5a0173725789437be2a02427f8058dba7fed6f82f626d8db2b18e4f0f299577d16ac511f127b3d13eeab17b261268efb8e7a55762b7ccd445e70133572f5aed2216802465bc0ed84f205d60fffd6522efba9f3e47de6a4937102d5cc830aad81974f57d196db0b170503ced87e36a3da99348335d16050ea7695c319a3377cfccbb16e5a1e25b936fc01c0d674b9b34009824375f7fb395d0a3df0e3cb54daf000004020000000100040009000000056e616d6573000000100000000400040009000000026976000400090000000773657373696f6e00040009000000046461746100040009000000097369676e6174757265000000fe"

and the reverse with
[`sodium::hex2bin`](https://docs.ropensci.org/sodium/reference/helpers.html):

``` r

identical(sodium::hex2bin(secret_hex), secret)
```

    ## [1] TRUE

(this is somewhat less space efficient than base64 encoding.

As a final option, you can just save the secret with `saveRDS` and read
it in with `readRDS` like any other option. This will be the best route
if the secret is saved into a more complicated R object (e.g., a list or
`data.frame`).

See the other cyphr vignette
([`vignette("data", package = "cyphr")`](https://docs.ropensci.org/cyphr/articles/data.md))
for a suggested workflow for exchanging secrets within a team, and the
wrapper functions below for more convenient ways of working with
encrypted data.

**Do you already have an ssh keypair?** To find out, run

``` r

cyphr::keypair_openssl(NULL, NULL)
```

One of three things will happen:

1.  you will be prompted for your password to decrypt your private key,
    and then after entering it an object `<cyphr_keypair: openssl>` will
    be returned - you’re good to go!

2.  you were *not* prompted for your password, but got a
    `<cyphr_keypair: openssl>` object. You should consider whether this
    is appropriate and consider generating a new keypair with the
    private key encrypted. If you don’t then anyone who can read your
    private key can decrypt any message intended for you.

3.  you get an error like
    `Did not find default ssh public key at ~/.ssh/id_rsa.pub`. You need
    to create a keypair.

To create a keypair, you can use the
[`cyphr::ssh_keygen()`](https://docs.ropensci.org/cyphr/reference/ssh_keygen.md)
function as

``` r

cyphr::ssh_keygen("~/.ssh")
```

This will create the keypair as `~/.ssh/id_rsa` and `~/.ssh/id_rsa.pub`,
which is where `cyphr` will look for your keys by default. See
[`?ssh_keygen`](https://docs.ropensci.org/cyphr/reference/ssh_keygen.md)
for more information. (On Linux and macOS you might use the `ssh-keygen`
command line utility. On windows, PuTTY\` has a utility for creating
keys.)

#### `sodium`

With `sodium`, things are largely the same with the exception that there
is no standard format for saving sodium keys. The bits below use an
in-memory key (which is just a collection of bytes) but these can also
be filenames, each of which contains the contents of the key written out
with `writeBin`.

First, generate keys for Alice:

``` r

key_a <- sodium::keygen()
pub_a <- sodium::pubkey(key_a)
```

the public key is derived from the private key, and Alice can share that
with Bob. We next generate Bob’s keys

``` r

key_b <- sodium::keygen()
pub_b <- sodium::pubkey(key_b)
```

Bob would now share is public key with Alice.

If Alice wants to send a message to Bob she again uses her private key
and Bob’s public key:

``` r

pair_a <- cyphr::keypair_sodium(pub_b, key_a)
```

As above, she can now send a message:

``` r

secret <- cyphr::encrypt_string("secret message", pair_a)
secret
```

    ##  [1] be 52 c3 66 3c 1e 39 d1 6d 7f bd d8 06 3a 95 c8 7d f0 6f eb e1 fa e2 54 b0
    ## [26] b1 c5 a7 68 29 e3 3a 9b 77 b6 f6 fe c2 3f 81 92 9b 1a fe 3a 97 22 f8 c6 9f
    ## [51] d2 b6 44 c1

Note how this line is identical to the one in the `openssl` section.

To decrypt this message, Bob would use Alice’s public key and his
private key:

``` r

pair_b <- cyphr::keypair_sodium(pub_a, key_b)
cyphr::decrypt_string(secret, pair_b)
```

    ## [1] "secret message"

## Encrypting things

Above, we used
[`cyphr::encrypt_string`](https://docs.ropensci.org/cyphr/reference/encrypt_data.md)
and
[`cyphr::decrypt_string`](https://docs.ropensci.org/cyphr/reference/encrypt_data.md)
to encrypt and decrypt a string. There are several such functions in the
package that encrypt and decrypt

- R objects `encrypt_object` / `decrypt_object` (using serialization and
  deserialization)
- strings: `encrypt_string` / `decrypt_string`
- raw vectors: `encrypt_data` / `decrypt_data`
- files: `encrypt_file` / `decrypt_file`

For this section we will just use a sodium symmetric encryption key

``` r

key <- cyphr::key_sodium(sodium::keygen())
```

For the examples below, in the case of asymmetric encryption (using
either
[`cyphr::keypair_openssl`](https://docs.ropensci.org/cyphr/reference/keypair_openssl.md)
or
[`cyphr::keypair_sodium`](https://docs.ropensci.org/cyphr/reference/keypair_sodium.md))
the sender would use their private key and the recipient’s public key
and the recipient would use the complementary key pair.

### Objects

Here’s an object to encrypt:

``` r

obj <- list(x = 1:10, y = "secret")
```

This creates a bunch of raw bytes corresponding to the data (it’s not
really possible to print this as anything nicer than bytes).

``` r

secret <- cyphr::encrypt_object(obj, key)
secret
```

    ##   [1] 87 a0 de b1 5d 5a 04 e4 64 1e ef 21 18 73 5e 5e ef 87 db fe f8 ff 17 e1 8d
    ##  [26] cb 71 df b2 4a b7 20 61 9e 83 b3 7c 99 65 01 23 e6 cc 6e af 6f e2 81 f5 79
    ##  [51] 67 b7 b6 b4 a0 77 63 cd 81 a4 ec cd b6 52 30 c5 64 43 b2 37 38 ce 14 ef 7a
    ##  [76] 2d 19 c2 de dc 99 fd f9 5d 18 58 42 87 5b 67 b2 b6 91 64 de 9b e0 74 3e 2d
    ## [101] c7 55 29 c4 f7 0c 6a c7 bd b3 a5 d6 08 e4 62 36 cb bc 79 86 29 ae 95 6b c9
    ## [126] 18 14 3d 20 9e 92 e6 fb 61 04 95 2f 72 1e 77 4c a1 34 e9 88 b2 2b 03 d1 5a
    ## [151] 55 a1 d9 f3 1b 93 61 d8 0d be a0 59 76 b6 7b 2c d6 d2 8e e9 c7 99 3e 93 e9
    ## [176] 1a 00 30 89 cb ba f7 96 7d 1c ab 6e 51 da d8 db 3d f0 9d ab c0 46 68 9a a4
    ## [201] 9e f5 79 c1 de e3 89 0d 2d e3 24 54 b8 3a 01 2b a8 86 5b 3a 07 10 97 41 52
    ## [226] 64 9c 8b d8 62 d2 ab e6 96 66 f8 50 72 78 1d 8c 34 49 46 2f 24 c7 d3 3c 2f
    ## [251] 85 33 52 17

The data can be decrypted with the `decrypt_object` function:

``` r

cyphr::decrypt_object(secret, key)
```

    ## $x
    ##  [1]  1  2  3  4  5  6  7  8  9 10
    ## 
    ## $y
    ## [1] "secret"

Optionally, this process can go via a file, using a third argument to
the functions (note that temporary files are used here for compliance
with CRAN policies - any path may be used in practice).

``` r

path_secret <- file.path(tempdir(), "secret.rds")
cyphr::encrypt_object(obj, key, path_secret)
```

There is now a file called `secret.rds` in the temporary directory:

``` r

file.exists(path_secret)
```

    ## [1] TRUE

though it is not actually an rds file:

``` r

readRDS(path_secret)
```

    ## Error in `readRDS()`:
    ## ! unknown input format

When passed a filename (as opposed to a raw vector),
[`cyphr::decrypt_object`](https://docs.ropensci.org/cyphr/reference/encrypt_data.md)
will read the object in before decrypting it

``` r

cyphr::decrypt_object(path_secret, key)
```

    ## $x
    ##  [1]  1  2  3  4  5  6  7  8  9 10
    ## 
    ## $y
    ## [1] "secret"

### Strings

For the case of strings we can do this in a slightly more lightweight
way (the above function routes through `serialize` / `deserialize` which
can be slow and will create larger objects than using `charToRaw` /
`rawToChar`)

``` r

secret <- cyphr::encrypt_string("secret", key)
secret
```

    ##  [1] 52 42 ed 03 fc a6 7e 84 42 c9 59 87 22 c4 07 81 c4 d5 5d 86 f4 05 66 5c 23
    ## [26] a6 5f 5e 50 90 6d 41 2e 1a c9 97 27 16 1f 22 4a 0b fe 08 82 49

and decrypt:

``` r

cyphr::decrypt_string(secret, key)
```

    ## [1] "secret"

### Plain raw data

If these are not enough for you, you can work directly with raw objects
(bunches of bytes) by using `encrypt_data`:

``` r

dat <- sodium::random(100)
dat # some random bytes
```

    ##   [1] 77 4f 7e 5c ba 81 f1 54 a9 12 44 72 b9 a8 fb 8f e7 46 ac 55 20 c0 70 2d a4
    ##  [26] c1 0a 12 23 e9 c0 30 20 75 e8 b8 93 c2 57 11 49 4a 43 ef 85 27 8e e3 b1 f7
    ##  [51] d8 d3 0f 20 70 f7 b6 e6 07 e1 79 ae 35 ac b2 22 9d 24 37 07 d1 0d a2 be ca
    ##  [76] 68 6e ce 93 83 58 13 5b 56 85 d4 e0 53 49 cf 56 b4 5a 2e 46 82 39 40 ff a6

``` r

secret <- cyphr::encrypt_data(dat, key)
secret
```

    ##   [1] 38 c7 b0 59 23 50 60 c4 6e 94 50 ea 1f a9 82 29 67 7a 6d 84 95 7a 0d 61 69
    ##  [26] f0 57 f2 07 07 74 f5 ba 51 bc b5 25 53 8e 99 d7 2c a9 77 b2 c8 d9 3c 76 1c
    ##  [51] 3a be cf 52 33 c7 a1 4d 17 dd c2 d4 fc 5e bd 9a 64 5f 53 ea ac 24 17 30 90
    ##  [76] 43 2f ba 98 fc 44 d9 0d b7 13 fe b1 a1 55 6f 4d ce 03 ca e9 9a 8a 77 39 fc
    ## [101] 4a 7b c6 d2 a4 2d b8 47 5b e4 32 c1 a2 e5 85 14 e3 c2 cc 2a 8d 17 23 db 01
    ## [126] 7c d9 2f 63 ac 19 39 60 5f ed 9a 58 e0 86 4c

Decrypted data is the same as a the original data

``` r

identical(cyphr::decrypt_data(secret, key), dat)
```

    ## [1] TRUE

### Files

Suppose we have written a file that we want to encrypt to send to
someone (in a temporary directory for compliance with CRAN policies)

``` r

path_data_csv <- file.path(tempdir(), "iris.csv")
write.csv(iris, path_data_csv, row.names = FALSE)
```

You can encrypt that file with

``` r

path_data_enc <- file.path(tempdir(), "iris.csv.enc")
cyphr::encrypt_file(path_data_csv, key, path_data_enc)
```

This encrypted file can then be decrypted with

``` r

path_data_decrypted <- file.path(tempdir(), "idis2.csv")
cyphr::decrypt_file(path_data_enc, key, path_data_decrypted)
```

Which is identical to the original:

``` r

tools::md5sum(c(path_data_csv, path_data_decrypted))
```

    ##           /tmp/RtmpApHXtx/iris.csv          /tmp/RtmpApHXtx/idis2.csv 
    ## "5fe92fe6a2c1928ef5a67b8939fdaf8d" "5fe92fe6a2c1928ef5a67b8939fdaf8d"

## An even higher level interface for files

This is the most user-friendly way of using the package when the aim is
to encrypt and decrypt files. The package provides a pair of functions
[`cyphr::encrypt`](https://docs.ropensci.org/cyphr/reference/encrypt.md)
and
[`cyphr::decrypt`](https://docs.ropensci.org/cyphr/reference/encrypt.md)
that wrap file writing and file reading functions. In general you would
use `encrypt` when writing a file and `decrypt` when reading one.
They’re designed to be used like so:

Suppose you have a super-secret object that you want to share privately

``` r

key <- cyphr::key_sodium(sodium::keygen())
x <- list(a = 1:10, b = "don't tell anyone else")
```

If you save `x` to disk with `saveRDS` it will be readable by everyone
until it is deleted. But if you encrypted the file that `saveRDS`
produced it would be protected and only people with the key can read it:

``` r

path_object <- file.path(tempdir(), "secret.rds")
cyphr::encrypt(saveRDS(x, path_object), key)
```

(see below for some more details on how this works).

This file cannot be read with `readRDS`:

``` r

readRDS(path_object)
```

    ## Error in `readRDS()`:
    ## ! unknown input format

but if we wrap the call with `decrypt` and pass in the config object it
can be decrypted and read:

``` r

cyphr::decrypt(readRDS(path_object), key)
```

    ## $a
    ##  [1]  1  2  3  4  5  6  7  8  9 10
    ## 
    ## $b
    ## [1] "don't tell anyone else"

What happens in the call above is `cyphr` uses “non standard evaluation”
to rewrite the call above so that it becomes (approximately)

1.  use
    [`cyphr::decrypt_file`](https://docs.ropensci.org/cyphr/reference/encrypt_data.md)
    to decrypt “secret.rds” as a temporary file
2.  call `readRDS` on that temporary file
3.  delete the temporary file (even if there is an error in the above
    calls)

This non-standard evaluation breaks referential integrity (so may not be
suitable for programming). You can always do this manually with
`encrypt_file` / `decrypt_file` so long as you make sure to clean up
after yourself.

The `encrypt` function inspects the call in the first argument passed to
it and works out for the function provided (`saveRDS`) which argument
corresponds to the filename (here `"secret.rds"`). It then rewrites the
call to write out to a temporary file (using
[`tempfile()`](https://rdrr.io/r/base/tempfile.html)). Then it calls
`encrypt_file` (see below) on this temporary file to create the file
asked for (`"secret.rds"`). Then it deletes the temporary file, though
this will also happen in case of an error in any of the above.

The `decrypt` function works similarly. It inspects the call and detects
that the first argument represents the filename. It decrypts that file
to create a temporary file, and then runs `readRDS` on that file. Again
it will delete the temporary file on exit.

The functions supported via this interface are:

- `readLines` / `writeLines`
- `readRDS` / `writeRDS`
- `read` / `save`
- `read.table` / `write.table`
- `read.csv` / `read.csv2` / `write.csv`
- `read.delim` / `read.delim2`

But new functions can be added with the `rewrite_register` function. For
example, to support the excellent
[rio](https://cran.r-project.org/package=rio) package, whose `import`
and `export` functions take the filename `file` you could use:

``` r

cyphr::rewrite_register("rio", "import", "file")
cyphr::rewrite_register("rio", "export", "file")
```

now you can read and write tabular data into and out of a great many
different file formats with encryption with calls like

``` r

cyphr::encrypt(rio::export(mtcars, "file.json"), key)
cyphr::decrypt(rio::import("file.json"), key)
```

The functions above use [non standard
evaluation](http://adv-r.had.co.nz/Computing-on-the-language.md) and so
may not be suitable for programming or use in packages. An “escape
hatch” is provided via `encrypt_` and `decrypt_` where the first
argument is a quoted expression.

``` r

cyphr::encrypt_(quote(saveRDS(x, path_object)), key)
cyphr::decrypt_(quote(readRDS(path_object)), key)
```

    ## $a
    ##  [1]  1  2  3  4  5  6  7  8  9 10
    ## 
    ## $b
    ## [1] "don't tell anyone else"

## Session keys

When using `key_openssl`, `keypair_openssl`, `key_sodium`, or
`keypair_sodium` we generate something that can decrypt data. The
objects that are returned by these functions can encrypt and decrypt
data and so it is reasonable to be concerned that if these objects were
themselves saved to disk your data would be compromised.

To avoid this, `cyphr` does not store private or symmetric keys directly
in these objects but instead encrypts the sensitive keys with a
`cyphr`-specific session key that is regenerated each time the package
is loaded. This means that the objects are practically only useful
within one session, and if saved with `save.image` (perhaps
automatically at the end of a session) the keys cannot be used to
decrypt data.

To manually invalidate all keys you can use the
[`cyphr::session_key_refresh`](https://docs.ropensci.org/cyphr/reference/session_key_refresh.md)
function. For example, here is a symmetric key:

``` r

key <- cyphr::key_sodium(sodium::keygen())
```

which we can use to encrypt a secret string

``` r

secret <- cyphr::encrypt_string("my secret", key)
```

and decrypt it:

``` r

cyphr::decrypt_string(secret, key)
```

    ## [1] "my secret"

If we refresh the session key we invalidate the `key` object

``` r

cyphr::session_key_refresh()
```

and after this point the key cannot be used any further

``` r

cyphr::decrypt_string(secret, key)
```

    ## Error:
    ## ! Failed to decrypt key as session key has changed

This approach works because the package holds the session key within its
environment (in `cyphr:::session$key`) which R will not serialize. As
noted above - this approach does not prevent an attacker with the
ability to snoop on your R session from discovering your private keys or
sensitive data but it does prevent accidentally saving keys in a way
that would be useful for an attacker to use in a subsequent session.

## Further reading

- The wikipedia page on Public Key cryptography has some nice diagrams
  that explain how key and data interact
  <https://en.wikipedia.org/wiki/Public-key_cryptography>
- The vignettes in the `openssl` (`vignette(package = "openssl")`) and
  `sodium` (`vignette(package = "openssl")`) packages have explanations
  of how the tools used in `cyphr` work and interface with R.

Confused? Need help? Found a bug?

- Post an issue on the [`cyphr` issue
  tracker](https://github.com/ropensci/cyphr/issues)
- Start a discussion on the [rOpenSci discussion
  forum](https://discuss.ropensci.org/)

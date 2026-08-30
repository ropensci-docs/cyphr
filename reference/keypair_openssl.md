# Asymmetric encryption with openssl

Wrap a pair of openssl keys. You should pass your private key and the
public key of the person that you are communicating with.

## Usage

``` r
keypair_openssl(
  pub,
  key,
  envelope = TRUE,
  password = NULL,
  authenticated = TRUE
)
```

## Arguments

- pub:

  An openssl public key. Usually this will be the path to the key, in
  which case it may either the path to a public key or be the path to a
  directory containing a file `id_rsa.pub`. If `NULL`, then your public
  key will be used (found via the environment variable `USER_PUBKEY`,
  then `~/.ssh/id_rsa.pub`). However, it is not that common to use your
  own public key - typically you want either the sender of a message you
  are going to decrypt, or the recipient of a message you want to send.

- key:

  An openssl private key. Usually this will be the path to the key, in
  which case it may either the path to a private key or be the path to a
  directory containing a file. You may specify `NULL` here, in which
  case the environment variable `USER_KEY` is checked and if that is not
  defined then `~/.ssh/id_rsa` will be used.

- envelope:

  A logical indicating if "envelope" encryption functions should be
  used. If so, then we use
  [`openssl::encrypt_envelope()`](https://jeroen.r-universe.dev/openssl/reference/encrypt_envelope.html)
  and
  [`openssl::decrypt_envelope()`](https://jeroen.r-universe.dev/openssl/reference/encrypt_envelope.html).
  If `FALSE` then we use
  [`openssl::rsa_encrypt()`](https://jeroen.r-universe.dev/openssl/reference/rsa_encrypt.html)
  and
  [`openssl::rsa_decrypt()`](https://jeroen.r-universe.dev/openssl/reference/rsa_encrypt.html).
  See the openssl docs for further details. The main effect of this is
  that using `envelope = TRUE` will allow you to encrypt much larger
  data than `envelope = FALSE`; this is because openssl asymmetric
  encryption can only encrypt data up to the size of the key itself.

- password:

  A password for the private key. If `NULL` then you will be prompted
  interactively for your password, and if a string then that string will
  be used as the password (but be careful in scripts!)

- authenticated:

  Logical, indicating if the result should be signed with your public
  key. If `TRUE` then your key will be verified on decryption. This
  provides tampering detection.

## See also

[`keypair_sodium()`](https://docs.ropensci.org/cyphr/reference/keypair_sodium.md)
for a similar function using sodium keypairs

## Examples

``` r

# Note this uses password = FALSE for use in examples only, but
# this should not be done for any data you actually care about.

# Note that the vignette contains much more information than this
# short example and should be referred to before using these
# functions.

# Generate two keypairs, one for Alice, and one for Bob
path_alice <- tempfile()
path_bob <- tempfile()
cyphr::ssh_keygen(path_alice, password = FALSE)
cyphr::ssh_keygen(path_bob, password = FALSE)

# Alice wants to send Bob a message so she creates a key pair with
# her private key and bob's public key (she does not have bob's
# private key).
pair_alice <- cyphr::keypair_openssl(pub = path_bob, key = path_alice)

# She can then encrypt a secret message:
secret <- cyphr::encrypt_string("hi bob", pair_alice)
secret
#>   [1] 58 0a 00 00 00 03 00 04 06 00 00 03 05 00 00 00 00 05 55 54 46 2d 38 00 00
#>  [26] 02 13 00 00 00 04 00 00 00 18 00 00 00 10 ed 62 a6 4c 26 6b d4 2c ba 34 6d
#>  [51] 89 fc bb e6 e4 00 00 00 18 00 00 01 00 52 00 68 e9 ff 3c 81 f2 5a fc dc ec
#>  [76] f0 17 ce e2 af c0 d0 f8 4d 9c cc 08 2b a5 15 da c9 af cf 2a 0b 25 f9 9b c9
#> [101] 33 d0 17 b2 e5 7b a1 07 51 7f dc 07 73 12 34 e9 11 f5 08 d3 c7 ad 3b 70 43
#> [126] 07 9d 23 42 8b 66 6e 7b 07 a2 dd 50 c6 83 71 58 67 3c 25 d9 fe c6 fe 07 3b
#> [151] 6e 3a 21 57 aa 95 64 35 e3 4f 12 1c eb 5b 2f 9d a4 a4 da c1 c2 01 46 e2 0d
#> [176] 44 7b 89 5d df f3 32 8f db 98 da 49 65 b1 af 79 4b c2 f8 66 4f c8 12 14 39
#> [201] 99 70 24 a3 ea b7 ba 37 75 60 3d e5 c6 37 dc 0b 23 ca c7 c0 7f 36 fc 6c 5c
#> [226] 0a f9 00 d8 b7 a3 6b 52 09 5a 4a d4 58 77 64 27 36 06 c8 d9 d4 4a 73 c9 e8
#> [251] fe 67 80 a4 ae 52 f3 14 cd 82 25 ab fb 2f fe 6b 07 16 91 04 ec b5 75 c7 09
#> [276] 37 1b 2b f5 a5 5e 93 9f 0d 9b 3a e2 fa 12 ba fc d8 d8 92 81 f1 dd 86 47 69
#> [301] 4d 74 0a fb 3c 39 bc be 9e d4 03 ee c8 67 b3 27 4c 24 04 00 00 00 18 00 00
#> [326] 00 10 a4 a4 33 7b a6 1a 73 a9 f1 36 55 f1 32 c1 23 d8 00 00 00 18 00 00 01
#> [351] 00 95 a5 20 14 a7 c0 02 a8 0c 99 35 0f f7 e1 ba fd c6 25 63 88 21 46 89 a8
#> [376] 25 d5 96 f0 69 7d ff 5e c8 23 d1 32 d0 32 70 fe 7a d5 7e 30 bb 95 4d 5f 4b
#> [401] e8 89 8f 12 e4 38 be d7 ef 69 3f f1 f2 95 ce 8d c7 88 27 b3 82 78 cc 14 27
#> [426] 5d e0 b2 d0 9e 41 fe 23 09 3c bc 1e 2d 1b 25 5b 11 36 8c ab 4c 58 44 c8 98
#> [451] 18 1c ca 20 17 54 a6 03 ba a2 ca 31 de bb 2e 7b d3 9f 86 b3 6f 46 d3 e7 59
#> [476] c5 d9 ce ec bd 48 4a 6d 9d 00 30 37 bb 73 c0 f9 02 e1 28 2d 08 a4 f6 c1 5c
#> [501] 99 18 8d 90 78 35 f0 92 27 14 fc 2f 3a fb 63 d2 26 17 05 e8 40 60 e9 bf 23
#> [526] aa f7 5f ac 81 a7 2f ce 9a b1 ee 5f 66 df e8 8a 7f f8 8d d3 bc 06 f4 22 4a
#> [551] a4 09 33 40 f4 fa 49 2e 0d d1 64 09 c9 12 7b da 06 4e eb 32 82 c6 0c 44 d7
#> [576] d7 3d 10 20 d7 a0 cc 6c 39 0c 00 47 e9 b3 40 d3 69 9b 1a b5 9a 81 39 ce 58
#> [601] 58 1d 08 82 6b cc ef 00 00 04 02 00 00 00 01 00 04 00 09 00 00 00 05 6e 61
#> [626] 6d 65 73 00 00 00 10 00 00 00 04 00 04 00 09 00 00 00 02 69 76 00 04 00 09
#> [651] 00 00 00 07 73 65 73 73 69 6f 6e 00 04 00 09 00 00 00 04 64 61 74 61 00 04
#> [676] 00 09 00 00 00 09 73 69 67 6e 61 74 75 72 65 00 00 00 fe

# Bob wants to read the message so he creates a key pair using
# Alice's public key and his private key:
pair_bob <- cyphr::keypair_openssl(pub = path_alice, key = path_bob)

cyphr::decrypt_string(secret, pair_bob)
#> [1] "hi bob"

# Clean up
unlink(path_alice, recursive = TRUE)
unlink(path_bob, recursive = TRUE)
```

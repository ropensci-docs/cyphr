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
#>  [26] 02 13 00 00 00 04 00 00 00 18 00 00 00 10 15 05 2e d7 d4 35 d3 af af db ea
#>  [51] 17 80 62 9f 02 00 00 00 18 00 00 01 00 29 3e 9d e8 cc 27 77 cc 2a 9e 52 e5
#>  [76] 71 d2 af ae 31 8f 6a 9c db 3f 70 87 bf cd 68 aa ed da d3 80 2e 97 4b 6d 52
#> [101] 22 70 37 03 9e ca 28 d3 02 12 b7 77 4f 9d 02 c6 18 34 52 28 8e 00 49 75 c3
#> [126] ca 68 91 b3 a6 09 16 e5 ae 02 d4 c4 1a e5 df 96 06 12 fd 4b c7 37 5b 7a ee
#> [151] 66 66 44 4e a4 5d 59 8b 3b a0 1f e5 6b 47 98 97 6b 72 9b bf 2b 44 89 e8 b8
#> [176] d5 a2 4c ed 3b 28 b1 5c 68 c5 c3 37 2f 0a 66 e1 cc 46 ad 44 ff 4d a8 07 9b
#> [201] 05 e4 e9 ba c8 55 dd ef 2b 13 13 dd d8 60 18 c9 13 3f 76 a8 3b 88 21 f1 5a
#> [226] a0 5b c2 10 f2 4b 65 11 39 a8 5c 15 05 78 ee b5 ee 50 d2 9a aa 7b 52 d9 9d
#> [251] 2f b2 85 9e 11 00 22 71 a4 b6 ad 6f 77 9d 71 cf aa 23 37 a3 ae 30 8e d1 f9
#> [276] 97 7c ce 26 bb 9d 87 e1 2c 1c b8 ae 8e 7e 22 34 76 1f de 9f 7e de d8 d2 05
#> [301] be 91 b9 f2 5f 25 31 a5 66 7f 02 80 87 67 37 dc 28 75 d9 00 00 00 18 00 00
#> [326] 00 10 f6 90 e2 a3 d5 13 48 c7 c3 1e 6d 55 78 3b 96 57 00 00 00 18 00 00 01
#> [351] 00 8f 3d 2f 60 c9 1c 47 8f 5b f8 2d 6d b7 0a dc 66 c5 a6 98 96 6d 12 f1 17
#> [376] 46 11 da ec 3f f1 f9 f5 35 b9 a6 78 3c 32 8b 1f ce d7 f5 2b 27 08 11 64 c6
#> [401] 5d 51 78 b8 1f 91 4a de d6 16 f2 b8 88 a0 7a be fb f2 59 0b 9a c3 0b 53 81
#> [426] 72 b4 28 d8 17 55 0b ed 6a 8c 4d a1 0a ad 56 c6 4c 02 a4 2e 44 3a 78 66 37
#> [451] 7f a6 11 ed f3 e8 50 8c e2 5c 8d da c5 a4 7b 12 f3 75 02 06 65 c2 16 50 ca
#> [476] 39 a4 02 d3 f6 bf d1 c7 7a 8b 62 36 bb 5f c2 6d 98 b6 15 7e 9f a1 8d cc 32
#> [501] e2 52 93 d2 d8 1c 08 7f bd 58 f6 10 12 79 9d d4 94 30 6e 0a 25 9f 81 f9 3c
#> [526] 32 92 8f 80 27 50 94 7b 2d ea ec d8 73 09 a6 b0 5e ab 10 78 d4 65 c8 eb 3c
#> [551] 5c 63 cc 7f e4 bc 29 4c 2c 89 de a5 fb b3 3c 7b 4e 23 58 61 fe 50 b7 df 26
#> [576] 28 06 79 a7 2f 41 64 55 15 aa 58 bc be 78 a8 6d ba bf de d6 50 c4 3a d3 c9
#> [601] a8 cb 26 a3 73 dc 97 00 00 04 02 00 00 00 01 00 04 00 09 00 00 00 05 6e 61
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

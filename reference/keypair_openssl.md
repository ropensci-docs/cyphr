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
#>  [26] 02 13 00 00 00 04 00 00 00 18 00 00 00 10 64 ad 4f 55 62 dc 24 8a 54 bd 33
#>  [51] fc 24 d7 d2 30 00 00 00 18 00 00 01 00 4d 67 f1 f6 35 76 84 eb 9f 4b 0c 95
#>  [76] 07 77 36 34 82 5c 69 25 d1 2c d1 ac f7 81 67 e0 7e 09 2e e3 fe d2 0a fc 1f
#> [101] c6 ff 5b f2 35 be 8b eb 33 8b 4b ad 58 de 15 92 47 c8 26 aa 3f dc 7d 17 11
#> [126] 13 59 ef 43 61 77 ff 27 e1 35 64 f4 d2 f3 13 45 ef a8 62 9b dd a6 13 e2 e8
#> [151] e5 de 51 d3 92 8d 8e 0c 9e 2f 6b 11 2e 5c 32 fa 99 09 32 62 80 98 9e a0 ec
#> [176] 56 65 e5 43 ce 00 1e 43 7a 47 c9 34 52 f0 51 cb 28 32 e9 d7 dc 3a 2f b8 ac
#> [201] 88 2a 1a 54 e6 84 e4 c1 95 a8 11 42 79 7c 9a 19 1c 5b ed 7a 69 45 29 fd 2a
#> [226] 2d 18 b8 7b d8 51 db fc ca 91 ae 0c 3c 96 60 37 54 e2 dc 73 a8 73 e7 1c c7
#> [251] 33 84 84 aa ff 89 df e5 cb ed f7 fc d4 03 77 0a 05 51 dd 81 bc 90 51 0a 8a
#> [276] 96 02 90 e5 ca 9f 50 31 35 cb 25 cf 1d d8 07 d4 0f b7 6c 28 9f 11 e0 d5 1c
#> [301] 69 06 59 25 7e e6 a6 ae 3d ae 31 9e 8a 6f 7d e8 fa 4a be 00 00 00 18 00 00
#> [326] 00 10 82 dd 85 71 02 6e 52 12 d3 97 6e f8 61 00 c2 9f 00 00 00 18 00 00 01
#> [351] 00 44 22 01 31 19 dc 79 3c 37 27 e9 d6 01 8a 14 16 11 44 22 05 4f 9e 9b 7a
#> [376] 94 5f 56 04 5d b0 bb 95 9f 6c 2f 4e 34 c5 f2 33 cb 77 55 ad d3 0b 52 c4 b5
#> [401] e3 6d 31 07 a6 d7 e5 a5 79 29 b7 99 49 05 00 32 e2 98 ad 14 43 65 42 99 c3
#> [426] 11 b9 83 e3 2e 02 39 2e a9 03 c5 f8 43 ce 6f a9 c0 06 d4 5d be 51 e3 81 5a
#> [451] 8d ce c3 5c b9 46 0d 9f bb 2b e9 12 9b de a0 b4 c1 22 03 8a af d6 3b b4 63
#> [476] 6b fd 1f c0 f4 ef 60 ca bb 5f 22 fd 71 15 9c ee c4 a6 a3 67 16 6d e5 7d 2b
#> [501] 16 41 98 a4 53 a6 9e ee c9 f1 a6 62 5c 5e 2d b4 3a ce 86 5e ef fc 7c 57 d5
#> [526] 8a 1a a7 c5 66 61 14 3b 63 9d e9 7f 9d 9e 72 55 86 67 96 a5 ac 9f 1f 5b 17
#> [551] e3 fc d0 62 28 64 08 d1 95 a3 e5 1e 2b 1e e3 5e 4c 33 0b b7 93 5e e8 b3 de
#> [576] f4 83 70 8c 15 05 34 85 33 8b 79 07 e8 13 32 5f 4b e4 7d 88 90 35 74 cc 15
#> [601] d6 04 a6 b2 1a b1 27 00 00 04 02 00 00 00 01 00 04 00 09 00 00 00 05 6e 61
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

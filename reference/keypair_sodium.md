# Asymmetric encryption with sodium

Wrap a pair of sodium keys for asymmetric encryption. You should pass
your private key and the public key of the person that you are
communicating with.

## Usage

``` r
keypair_sodium(pub, key, authenticated = TRUE)
```

## Arguments

- pub:

  A sodium public key. This is either a raw vector of length 32 or a
  path to file containing the contents of the key (written by
  [`writeBin()`](https://rdrr.io/r/base/readBin.html)).

- key:

  A sodium private key. This is either a raw vector of length 32 or a
  path to file containing the contents of the key (written by
  [`writeBin()`](https://rdrr.io/r/base/readBin.html)).

- authenticated:

  Logical, indicating if authenticated encryption (via
  [`sodium::auth_encrypt()`](https://docs.ropensci.org/sodium/reference/messaging.html)
  /
  [`sodium::auth_decrypt()`](https://docs.ropensci.org/sodium/reference/messaging.html))
  should be used. If `FALSE` then
  [`sodium::simple_encrypt()`](https://docs.ropensci.org/sodium/reference/simple.html)
  /
  [`sodium::simple_decrypt()`](https://docs.ropensci.org/sodium/reference/simple.html)
  will be used. The difference is that with `authenticated = TRUE` the
  message is signed with your private key so that tampering with the
  message will be detected.

## Details

*NOTE*: the order here (pub, key) is very important; if the wrong order
is used you cannot decrypt things. Unfortunately because sodium keys are
just byte sequences there is nothing to distinguish the public and
private keys so this is a pretty easy mistake to make.

## See also

[`keypair_openssl()`](https://docs.ropensci.org/cyphr/reference/keypair_openssl.md)
for a similar function using openssl keypairs

## Examples

``` r

# Generate two keypairs, one for Alice, and one for Bob
key_alice <- sodium::keygen()
pub_alice <- sodium::pubkey(key_alice)
key_bob <- sodium::keygen()
pub_bob <- sodium::pubkey(key_bob)

# Alice wants to send Bob a message so she creates a key pair with
# her private key and bob's public key (she does not have bob's
# private key).
pair_alice <- cyphr::keypair_sodium(pub = pub_bob, key = key_alice)

# She can then encrypt a secret message:
secret <- cyphr::encrypt_string("hi bob", pair_alice)
secret
#>  [1] 6d b1 e4 f5 de a5 01 fc a6 80 17 55 46 40 c0 10 8a 2f 28 7b e2 11 8d e1 3b
#> [26] c0 93 a9 da 88 be e2 8f c6 b5 53 4b 5b 23 d6 84 c4 14 13 4e 81

# Bob wants to read the message so he creates a key pair using
# Alice's public key and his private key:
pair_bob <- cyphr::keypair_sodium(pub = pub_alice, key = key_bob)

cyphr::decrypt_string(secret, pair_bob)
#> [1] "hi bob"
```

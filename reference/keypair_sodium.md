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
#>  [1] 18 3b 79 95 c4 71 b1 27 43 b1 c8 f1 51 88 80 da 99 d0 39 c0 fa a2 38 46 6f
#> [26] 66 85 0c 4a 8a a2 16 3c ac c3 d0 31 eb 2b b8 1c d0 b6 08 94 8c

# Bob wants to read the message so he creates a key pair using
# Alice's public key and his private key:
pair_bob <- cyphr::keypair_sodium(pub = pub_alice, key = key_bob)

cyphr::decrypt_string(secret, pair_bob)
#> [1] "hi bob"
```

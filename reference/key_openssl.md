# Symmetric encryption with openssl

Wrap an openssl symmetric (aes) key. This can be used with the functions
[`encrypt_data()`](https://docs.ropensci.org/cyphr/reference/encrypt_data.md)
and
[`decrypt_data()`](https://docs.ropensci.org/cyphr/reference/encrypt_data.md),
along with the higher level wrappers
[`encrypt()`](https://docs.ropensci.org/cyphr/reference/encrypt.md) and
[`decrypt()`](https://docs.ropensci.org/cyphr/reference/encrypt.md).
With a symmetric key, everybody uses the same key for encryption and
decryption.

## Usage

``` r
key_openssl(key, mode = "cbc")
```

## Arguments

- key:

  An openssl aes key (i.e., an object of class `aes`).

- mode:

  The encryption mode to use. Options are `cbc`, `ctr` and `gcm` (see
  the `openssl` package for more details)

## Examples

``` r
# Create a new key
key <- cyphr::key_openssl(openssl::aes_keygen())
key
#> <cyphr_key: openssl>

# With this key encrypt a string
secret <- cyphr::encrypt_string("my secret string", key)
# And decrypt it again:
cyphr::decrypt_string(secret, key)
#> [1] "my secret string"
```

# Symmetric encryption with sodium

Wrap a sodium symmetric key. This can be used with the functions
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
key_sodium(key)
```

## Arguments

- key:

  A sodium key (i.e., generated with
  [`sodium::keygen()`](https://docs.ropensci.org/sodium/reference/keygen.html)

## Examples

``` r
# Create a new key
key <- cyphr::key_sodium(sodium::keygen())
key
#> <cyphr_key: sodium>

# With this key encrypt a string
secret <- cyphr::encrypt_string("my secret string", key)
# And decrypt it again:
cyphr::decrypt_string(secret, key)
#> [1] "my secret string"
```

# Refresh the session key

Refresh the session key, invalidating all keys created by
[`key_openssl()`](https://docs.ropensci.org/cyphr/reference/key_openssl.md),
[`keypair_openssl()`](https://docs.ropensci.org/cyphr/reference/keypair_openssl.md),
[`key_sodium()`](https://docs.ropensci.org/cyphr/reference/key_sodium.md)
and
[`keypair_sodium()`](https://docs.ropensci.org/cyphr/reference/keypair_sodium.md).

## Usage

``` r
session_key_refresh()
```

## Details

Running this function will invalidate *all* keys loaded with the above
functions. It should not be needed very often.

## Examples

``` r

# Be careful - if you run this then all keys loaded from file will
# no longer work until reloaded
if (FALSE) {
  cyphr::session_key_refresh()
}
```

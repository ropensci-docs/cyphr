# User commands

User commands

## Usage

``` r
data_request_access(path_data = NULL, path_user = NULL, quiet = FALSE)

data_key(
  path_data = NULL,
  path_user = NULL,
  test = TRUE,
  quiet = FALSE,
  cache = TRUE
)
```

## Arguments

- path_data:

  Path to the data. If not given, then we look recursively down below
  the working directory for a ".cyphr" directory, and use that as the
  data directory.

- path_user:

  Path to the directory with your user key. Usually this can be omitted.
  This argument is passed in as both `pub` and `key` to
  [`keypair_openssl()`](https://docs.ropensci.org/cyphr/reference/keypair_openssl.md).
  Briefly, if this argument is not given we look at the environment
  variables `USER_PUBKEY` and `USER_KEY` - if set then these must refer
  to path of your public and private keys. If these environment
  variables are not set then we fall back on `~/.ssh/id_rsa.pub` and
  `~/.ssh/id_rsa`, which should work in most environments.
  Alternatively, provide a path to a directory where the file
  `id_rsa.pub` and `id_rsa` can be found.

- quiet:

  Suppress printing of informative messages.

- test:

  Test that the encryption is working? (Recommended)

- cache:

  Cache the key within the session. This will be useful if you are using
  ssh keys that have passwords, as if the key is found within the cache,
  then you will not have to re-enter your password. Using
  `cache = FALSE` neither looks for the key in the cache, nor saves it.

## Examples

``` r

# The workflow here does not really lend itself to an example,
# please see the vignette.

# Suppose that Alice has created a data directory:
path_alice <- tempfile()
cyphr::ssh_keygen(path_alice, password = FALSE)
path_data <- tempfile()
dir.create(path_data, FALSE, TRUE)
cyphr::data_admin_init(path_data, path_user = path_alice)
#> Generating data key
#> Authorising ourselves
#> Adding key f2:4f:25:20:0d:b7:82:64:d6:a7:25:41:b6:76:ae:cc:a1:22:dc:0d:5f:be:07:bb:61:19:8b:5e:9c:59:00:a3
#>   user: root
#>   host: 8308a62867f6
#>   date: 2026-09-29 07:31:15.156694
#> Verifying

# If Bob can also write to the data directory (e.g., it is a
# shared git repo, on a shared drive, etc), then he can request
# access
path_bob <- tempfile()
cyphr::ssh_keygen(path_bob, password = FALSE)
hash <- cyphr::data_request_access(path_data, path_user = path_bob)
#> A request has been added
#> Email someone with access to add you
#> 
#>     hash: a7:dd:e9:9f:ef:b5:43:5f:61:e2:d1:9c:32:15:a5:87:1c:7e:3e:8b:dc:98:cd:e0:72:e1:ce:c4:a6:98:87:a4
#> 
#> If you are using git, you will need to commit and push first:
#> 
#>     git add .cyphr
#>     git commit -m "Please add me to the dataset"
#>     git push

# Alice can authorise Bob
cyphr::data_admin_authorise(path_data, path_user = path_alice, yes = TRUE)
#> There is 1 request for access
#> Adding key a7:dd:e9:9f:ef:b5:43:5f:61:e2:d1:9c:32:15:a5:87:1c:7e:3e:8b:dc:98:cd:e0:72:e1:ce:c4:a6:98:87:a4
#>   user: root
#>   host: 8308a62867f6
#>   date: 2026-09-29 07:31:15.350404
#> Added 1 key
#> If you are using git, you will need to commit and push:
#> 
#>     git add .cyphr
#>     git commit -m "Authorised root"
#>     git push

# After which Bob can get the data key
cyphr::data_key(path_data, path_user = path_bob)
#> <cyphr_key: sodium>

# See the vignette for more details.  This is not the best medium
# to explore this.

# Cleanup
unlink(path_alice, recursive = TRUE)
unlink(path_bob, recursive = TRUE)
unlink(path_data, recursive = TRUE)
```

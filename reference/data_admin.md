# Encrypted data administration

Encrypted data administration; functions for setting up, adding users,
etc.

## Usage

``` r
data_admin_init(path_data, path_user = NULL, quiet = FALSE)

data_admin_authorise(
  path_data = NULL,
  hash = NULL,
  path_user = NULL,
  yes = FALSE,
  quiet = FALSE
)

data_admin_list_requests(path_data = NULL)

data_admin_list_keys(path_data = NULL)
```

## Arguments

- path_data:

  Path to the data set. We will store a bunch of things in a hidden
  directory within this path. By default in most functions we will
  search down the tree until we find the .cyphr directory

- path_user:

  Path to the directory with your ssh key. Usually this can be omitted.

- quiet:

  Suppress printing of informative messages.

- hash:

  A vector of hashes to add. If provided, each hash can be the binary or
  string representation of the hash to add. Or omit to add each request.

- yes:

  Skip the confirmation prompt? If any request is declined then the
  function will throw an error on exit.

## Details

`data_admin_init` initialises the system; it will create a data key if
it does not exist and authorise you. If it already exists and you do not
have access it will throw an error.

`data_admin_authorise` authorises a key by creating a key to the data
that the user can use in conjunction with their personal key.

`data_admin_list_requests` lists current requests.

`data_admin_list_keys` lists known keys that can access the data. Note
that this is *not secure*; keys not listed here may still be able to
access the data (if a key was authorised and moved elsewhere for
example). Conversely, if the user has deleted or changed their key they
will not be able to access the data despite the key being listed here.

## See also

[`data_request_access()`](https://docs.ropensci.org/cyphr/reference/data_user.md)
for requesting access to the data, and and `data_key` for using the data
itself. But for a much more thorough overview, see the vignette
([`vignette("data", package = "cyphr")`](https://docs.ropensci.org/cyphr/articles/data.md)).

## Examples

``` r

# The workflow here does not really lend itself to an example,
# please see the vignette instead.

# First we need a set of user ssh keys.  In a non example
# environment your personal ssh keys will probably work well, but
# hopefully they are password protected so cannot be used in
# examples.  The password = FALSE argument is only for testing,
# and should not be used for data that you care about.
path_ssh_key <- tempfile()
cyphr::ssh_keygen(path_ssh_key, password = FALSE)

# Initialise the data directory, using this key path.  Ordinarily
# the path_user argument would not be needed because we would be
# using your user ssh keys:
path_data <- tempfile()
dir.create(path_data, FALSE, TRUE)
cyphr::data_admin_init(path_data, path_user = path_ssh_key)
#> Generating data key
#> Authorising ourselves
#> Adding key 81:9a:9b:b3:e2:a3:ff:a0:e1:bd:a2:20:6a:49:d7:97:9c:44:9e:4d:8d:96:5d:f0:f1:dd:bb:d0:fe:a8:6e:c8
#>   user: root
#>   host: 8308a62867f6
#>   date: 2026-09-29 07:31:14.795403
#> Verifying

# Now you can get the data key
key <- cyphr::data_key(path_data, path_user = path_ssh_key)

# And encrypt things with it
cyphr::encrypt_string("hello", key)
#>  [1] 72 4d 63 df ef 00 f5 ee 3e 42 b8 0e a5 3c df 6b 99 6c 4a 57 ac 78 25 bc ce
#> [26] 8f dd 4e e8 92 bd 4c 3b 00 43 2d ec 29 fa 2d a0 86 19 ba ba

# See the vignette for more details.  This is not the best medium
# to explore this.

# Cleanup
unlink(path_ssh_key, recursive = TRUE)
unlink(path_data, recursive = TRUE)
```

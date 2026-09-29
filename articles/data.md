# Data Encryption

**The scenario:**

A group of people are working on a sensitive data set that for practical
reasons needs to be stored in a place that we’re not 100% happy with the
security (e.g., Dropbox), or we’re concerned that files stored in plain
text on users computers (e.g. laptops) may lead to the data being
compromised.

If the data can be stored encrypted but everyone in the group can still
read and write the data then we’ve improved the situation somewhat. But
organising for everyone to get a copy of the key to decrypt the data
files is non-trivial. The workflow described here aims to simplify this
procedure using lower-level functions in the `cyphr` package.

The general procedure is this:

1.  A person will set up a set of personal keys and a key for the data.
    The data key will be encrypted with their personal key so they have
    access to the data but nobody else does. At this point the data can
    be encrypted.

2.  Additional users set up personal keys and request access to the
    data. Anyone with access to the data can grant access to anyone
    else.

Before doing any of this, everyone needs to have ssh keys set up. By
default the package will use your ssh keys found at “~/.ssh”; see the
main package vignette for how to use this.

For clarity here we will generate two sets of key pairs for two actors
Alice and Bob:

``` r

path_key_alice <- cyphr::ssh_keygen(password = FALSE)
path_key_bob <- cyphr::ssh_keygen(password = FALSE)
```

These would ordinarily be on different machines (nobody has access to
anyone else’s private key) and they would be password protected. In the
function calls below, all the `path_user` arguments would be omitted.

We’ll store data in the directory `data`; at present there is nothing
there (this is in a temporary directory for compliance with CRAN
policies but would ordinarily be somewhere persistent and under version
control ideally).

``` r

data_dir <- file.path(tempdir(), "data")
dir.create(data_dir)
dir(data_dir)
```

    ## character(0)

**First**, create a personal set of keys. These will be shared across
all projects and stored away from the data. Ideally one would do this
with `ssh-keygen` at the command line, following one of the many guides
available. A utility function `ssh_keygen` (which simply calls
`ssh-keygen` for you) is available in this package though. You will need
to generate a key on each computer you want access from. Don’t copy the
key around. If you lose your user key you will lose access to the data!

**Second**, create a key for the data and encrypt that key with your
personal key. Note that the data key is never stored directly - it is
always stored encrypted by a personal key.

``` r

cyphr::data_admin_init(data_dir, path_user = path_key_alice)
```

    ## Generating data key

    ## Authorising ourselves

    ## Adding key 56:85:b6:f2:25:0e:80:b2:64:d7:51:32:b6:05:01:d0:e0:57:88:a2:5b:df:4e:9c:6c:e9:13:99:70:ad:ec:ae
    ##   user: root
    ##   host: 8308a62867f6
    ##   date: 2026-09-29 07:31:27.424928

    ## Verifying

The data key is very important. If it is deleted, then the data cannot
be decrypted. So do not delete the directory `data_dir/.cyphr`! Ideally
add it to your version control system so that it cannot be lost. Of
course, if you’re working in a group, there are multiple copies of the
data key (each encrypted with a different person’s personal key) which
reduces the chance of total loss.

This command can be run multiple times safely; if it detects it has been
rerun and the data key will not be regenerated.

``` r

cyphr::data_admin_init(data_dir, path_user = path_key_alice)
```

    ## Already set up at /tmp/RtmpdvvMlL/data

    ## Verifying

**Third**, you can add encrypted data to the directory (or to anywhere
really). When run, `cyphr::config_data` will verify that it can actually
decrypt things.

``` r

key <- cyphr::data_key(data_dir, path_user = path_key_alice)
```

This object can be used with all the `cyphr` functions (see the “cyphr”
vignette;
[`vignette("cyphr")`](https://docs.ropensci.org/cyphr/articles/cyphr.md))

``` r

filename <- file.path(data_dir, "iris.rds")
cyphr::encrypt(saveRDS(iris, filename), key)
dir(data_dir)
```

    ## [1] "iris.rds"

The file is encrypted and so cannot be read with `readRDS`:

``` r

readRDS(filename)
```

    ## Error in `readRDS()`:
    ## ! unknown input format

But we can decrypt and read it:

``` r

head(cyphr::decrypt(readRDS(filename), key))
```

    ##   Sepal.Length Sepal.Width Petal.Length Petal.Width Species
    ## 1          5.1         3.5          1.4         0.2  setosa
    ## 2          4.9         3.0          1.4         0.2  setosa
    ## 3          4.7         3.2          1.3         0.2  setosa
    ## 4          4.6         3.1          1.5         0.2  setosa
    ## 5          5.0         3.6          1.4         0.2  setosa
    ## 6          5.4         3.9          1.7         0.4  setosa

**Fourth**, have someone else join in. Recall that to simulate another
person here, I’m going to pass an argument `path_user = path_key_bob`
though to the functions. This contains the path to “Bob”’s ssh keypair.
If run on an actually different computer this would not be needed; this
is just to simulate two users in a single session for this vignette (see
minimal example below where this is simulated). Again, typically this
user would also not use the
[`cyphr::ssh_keygen`](https://docs.ropensci.org/cyphr/reference/ssh_keygen.md)
function but use the `ssh-keygen` command from their shell.

We’re going to assume that the user can read and write to the data. This
is the case for my use case where the data are stored on dropbox and
will be the case with GitHub based distribution, though there would be a
pull request step in here.

This user cannot read the data, though trying to will print a message
explaining how you might request access:

``` r

key_bob <- cyphr::data_key(data_dir, path_user = path_key_bob)
```

But `bob` is your collaborator and needs access! What they need to do is
run:

``` r

cyphr::data_request_access(data_dir, path_user = path_key_bob)
```

    ## A request has been added

    ## Email someone with access to add you
    ## 
    ##     hash: c6:19:d1:eb:0d:10:d4:23:b7:e8:c8:c0:ec:23:36:34:d0:b7:38:16:65:8b:71:fc:c1:31:f8:97:08:71:e7:15
    ## 
    ## If you are using git, you will need to commit and push first:
    ## 
    ##     git add .cyphr
    ##     git commit -m "Please add me to the dataset"
    ##     git push

(again, ordinarily you would not need the `bob` bit here)

The user should the send an email to someone with access and quote the
hash in the message above.

**Fifth**, back on the first computer we can authorise the second user.
First, see who has requested access:

``` r

req <- cyphr::data_admin_list_requests(data_dir)
req
```

    ## 1 key:
    ##   c6:19:d1:eb:0d:10:d4:23:b7:e8:c8:c0:ec:23:36:34:d0:b7:38:16:65:8b:71:fc:c1:31:f8:97:08:71:e7:15
    ##     user: root
    ##     host: 8308a62867f6
    ##     date: 2026-09-29 07:31:28.036858

We can see the same hash here as above
(`c619d1eb0d10d423b7e8c8c0ec233634d0b73816658b71fcc131f8970871e715`)

…and then grant access to them with the
[`cyphr::data_admin_authorise`](https://docs.ropensci.org/cyphr/reference/data_admin.md)
function.

``` r

cyphr::data_admin_authorise(data_dir, yes = TRUE, path_user = path_key_alice)
```

    ## There is 1 request for access

    ## Adding key c6:19:d1:eb:0d:10:d4:23:b7:e8:c8:c0:ec:23:36:34:d0:b7:38:16:65:8b:71:fc:c1:31:f8:97:08:71:e7:15
    ##   user: root
    ##   host: 8308a62867f6
    ##   date: 2026-09-29 07:31:28.036858

    ## Added 1 key

    ## If you are using git, you will need to commit and push:
    ## 
    ##     git add .cyphr
    ##     git commit -m "Authorised root"
    ##     git push

If you do not specify `yes = TRUE` will prompt for confirmation at each
key added.

This has cleared the request queue:

``` r

cyphr::data_admin_list_requests(data_dir)
```

    ## (empty)

and added it to our set of keys:

``` r

cyphr::data_admin_list_keys(data_dir)
```

    ## 2 keys:
    ##   56:85:b6:f2:25:0e:80:b2:64:d7:51:32:b6:05:01:d0:e0:57:88:a2:5b:df:4e:9c:6c:e9:13:99:70:ad:ec:ae
    ##     user: root
    ##     host: 8308a62867f6
    ##     date: 2026-09-29 07:31:27.424928
    ##   c6:19:d1:eb:0d:10:d4:23:b7:e8:c8:c0:ec:23:36:34:d0:b7:38:16:65:8b:71:fc:c1:31:f8:97:08:71:e7:15
    ##     user: root
    ##     host: 8308a62867f6
    ##     date: 2026-09-29 07:31:28.036858

**Finally**, as soon as the authorisation has happened, the user can
encrypt and decrypt files:

``` r

key_bob <- cyphr::data_key(data_dir, path_user = path_key_bob)
head(cyphr::decrypt(readRDS(filename), key_bob))
```

    ##   Sepal.Length Sepal.Width Petal.Length Petal.Width Species
    ## 1          5.1         3.5          1.4         0.2  setosa
    ## 2          4.9         3.0          1.4         0.2  setosa
    ## 3          4.7         3.2          1.3         0.2  setosa
    ## 4          4.6         3.1          1.5         0.2  setosa
    ## 5          5.0         3.6          1.4         0.2  setosa
    ## 6          5.4         3.9          1.7         0.4  setosa

## Minimal example

As above, but with less discussion:

Setup, on Alice’s computer:

``` r

cyphr::data_admin_init(data_dir, path_user = path_key_alice)
```

    ## Generating data key

    ## Authorising ourselves

    ## Adding key 56:85:b6:f2:25:0e:80:b2:64:d7:51:32:b6:05:01:d0:e0:57:88:a2:5b:df:4e:9c:6c:e9:13:99:70:ad:ec:ae
    ##   user: root
    ##   host: 8308a62867f6
    ##   date: 2026-09-29 07:31:28.494328

    ## Verifying

Get the data key key:

``` r

key <- cyphr::data_key(data_dir, path_user = path_key_alice)
```

Encrypt a file:

``` r

cyphr::encrypt(saveRDS(iris, filename), key)
```

Request access, on Bob’s computer:

``` r

hash <- cyphr::data_request_access(data_dir, path_user = path_key_bob)
```

    ## A request has been added

    ## Email someone with access to add you
    ## 
    ##     hash: c6:19:d1:eb:0d:10:d4:23:b7:e8:c8:c0:ec:23:36:34:d0:b7:38:16:65:8b:71:fc:c1:31:f8:97:08:71:e7:15
    ## 
    ## If you are using git, you will need to commit and push first:
    ## 
    ##     git add .cyphr
    ##     git commit -m "Please add me to the dataset"
    ##     git push

Alice authorises this request::

``` r

cyphr::data_admin_authorise(data_dir, yes = TRUE, path_user = path_key_alice)
```

    ## There is 1 request for access

    ## Adding key c6:19:d1:eb:0d:10:d4:23:b7:e8:c8:c0:ec:23:36:34:d0:b7:38:16:65:8b:71:fc:c1:31:f8:97:08:71:e7:15
    ##   user: root
    ##   host: 8308a62867f6
    ##   date: 2026-09-29 07:31:28.697285

    ## Added 1 key

    ## If you are using git, you will need to commit and push:
    ## 
    ##     git add .cyphr
    ##     git commit -m "Authorised root"
    ##     git push

Bob can get the data key:

``` r

key <- cyphr::data_key(data_dir, path_user = path_key_bob)
```

Bob can read the secret data:

``` r

head(cyphr::decrypt(readRDS(filename), key))
```

    ##   Sepal.Length Sepal.Width Petal.Length Petal.Width Species
    ## 1          5.1         3.5          1.4         0.2  setosa
    ## 2          4.9         3.0          1.4         0.2  setosa
    ## 3          4.7         3.2          1.3         0.2  setosa
    ## 4          4.6         3.1          1.5         0.2  setosa
    ## 5          5.0         3.6          1.4         0.2  setosa
    ## 6          5.4         3.9          1.7         0.4  setosa

## Details & disclosure

Encryption does not work through security through obscurity; it works
because we can rely on the underlying maths enough to be open about how
things are stored and where.

Most encryption libraries require some degree of security in the
underlying software. Because of the way R works this is very difficult
to guarantee; it is trivial to rewrite code in running packages to skip
past verification checks. So this package is *not* designed to (or able
to) avoid exploits in your running code; an attacker could intercept
your private keys, the private key to the data, or skip the verification
checks that are used to make sure that the keys you load are what they
say they are. However, the *data* are safe; only people who have keys to
the data will be able to read it.

`cyphr` uses two different encryption algorithms; it uses RSA encryption
via the `openssl` package for user keys, because there is a common file
format for these keys so it makes user configuration easier. It uses the
modern sodium package (and through that the libsodium library) for data
encryption because it is very fast and simple to work with. This does
leave two possible points of weakness as a vulnerability in either of
these libraries could lead to an exploit that could allow decryption of
your data.

Each user has a public/private key pair. Typically this is in
`~/.ssh/id_rsa.pub` and `~/.ssh/id_rsa`, and if found these will be
used. Alternatively the location of the keypair can be stored elsewhere
and pointed at with the `USER_KEY` or `USER_PUBKEY` environment
variables. The key may be password protected (and this is recommended!)
and the password will be requested without ever echoing it to the
terminal.

The data directory has a hidden directory `.cyphr` in it.

``` r

dir(data_dir, all.files = TRUE, no.. = TRUE)
```

    ## [1] ".cyphr"   "iris.rds"

This does not actually need to be stored with the data but it makes
sense to (there are workflows where data is stored remotely where
storing this directory might make sense). The “keys” directory contains
a number of files; one for each person who has access to the data.

``` r

dir(file.path(data_dir, ".cyphr", "keys"))
```

    ## [1] "5685b6f2250e80b264d75132b60501d0e05788a25bdf4e9c6ce9139970adecae"
    ## [2] "c619d1eb0d10d423b7e8c8c0ec233634d0b73816658b71fcc131f8970871e715"

``` r

names(cyphr::data_admin_list_keys(data_dir))
```

    ## [1] "5685b6f2250e80b264d75132b60501d0e05788a25bdf4e9c6ce9139970adecae"
    ## [2] "c619d1eb0d10d423b7e8c8c0ec233634d0b73816658b71fcc131f8970871e715"

(the file `test` is a small file encrypted with the data key used to
verify everything is working OK).

Each file is stored in RDS format and is a list with elements:

- user: the reported user name of the person who created request for
  data
- host: the reported computer name
- date: the time the request was generated
- pub: the RSA public key of the user
- key: the data key, encrypted with the user key. Without the private
  key, this cannot be used. With the user’s private key this can be used
  to generate the symmetric key to the data.

``` r

h <- names(cyphr::data_admin_list_keys(data_dir))[[1]]
readRDS(file.path(data_dir, ".cyphr", "keys", h))
```

    ## $user
    ## [1] "root"
    ## 
    ## $host
    ## [1] "8308a62867f6"
    ## 
    ## $date
    ## [1] "2026-09-29 07:31:28 UTC"
    ## 
    ## $pub
    ## [2048-bit rsa public key]
    ## md5: 40900e2bc2651ef9a84a65c38ebe89f5
    ## sha256: 5685b6f2250e80b264d75132b60501d0e05788a25bdf4e9c6ce9139970adecae
    ## 
    ## $key
    ##   [1] 5d 90 39 77 69 98 2c 15 b9 62 c3 b4 ca 57 63 bf 6e ff c6 79 7d 92 31 44 e8
    ##  [26] 2b ed f9 c9 00 a4 11 da 38 cf 72 1b 0f 8f 11 8d a2 d3 eb b0 ed d9 68 92 0d
    ##  [51] b9 e0 37 99 dc 70 2b 26 5c 55 84 cb 0e a3 ea 2d 10 9b 20 d1 d1 01 bf 18 0f
    ##  [76] 41 8a 45 a1 e1 79 e6 90 ff 93 6a 43 99 48 18 b0 80 10 17 58 21 f3 64 db cb
    ## [101] 5b 0c 55 82 07 7d 3c 0f cf 91 d1 74 7c fe 3b 48 2b 9b b4 7a 9b 09 67 c0 93
    ## [126] a8 11 43 06 a9 d2 f4 ac 1d e8 9a 41 cc 65 f1 97 0d 0e dd b7 fb a2 78 e4 fd
    ## [151] 86 9a 55 0e 16 ab 6a 79 12 be eb df b1 3e 14 e7 fb c9 56 98 fc ed f7 c1 cf
    ## [176] ec 88 a0 ec 85 cd c1 c2 0f 55 5e 99 f6 db 74 19 92 a0 74 8e ea 74 75 96 aa
    ## [201] e9 3d f7 24 d9 33 80 63 29 b8 37 5b 8f 5f 42 eb 03 d9 4c e7 ed c7 a8 d1 af
    ## [226] 78 cc e6 0e f1 06 91 de cd 1f 4a 19 1f 93 ba 2a 62 ee 4a 6b a8 9f 85 20 79
    ## [251] b5 c0 68 c5 b2 4e

You can see that the hash of the public key is the same as name of the
stored file here (which is used to prevent collisions when multiple
people request access at the same time).

``` r

h
```

    ## [1] "5685b6f2250e80b264d75132b60501d0e05788a25bdf4e9c6ce9139970adecae"

When a request is posted it is an RDS file with all of the above except
for the `key` element, which is added during authorisation.

(Note that the verification relies on the package code not being
attacked, and given R’s highly dynamic nature an attacker could easily
swap out the definition for the verification function with something
that always returns `TRUE`.)

When an authorised user creates the `data_key` object (which allows
decryption of the data) `secret` will:

- read their private user key (probably from `~/.ssh/id_rsa`)
- read the encrypted data key from the data directory (the `$key`
  element from the list above).
- decrypt this data key using their user key to yield the the data
  symmetric key.

## Limitations

In the Dropbox scenario, non-password protected keys will afford only
limited protection. This is because even though the keys and data are
stored separately on Dropbox, they will be in the same place on a local
computer; if that computer is lost then the only thing preventing an
attacker recovering the data is security through obscurity (the data
would appear to be random junk but they will be able to run your
analysis scripts as easily as you can). Password protected keys will
improve this situation considerably as without a password the data
cannot be recovered.

The data is not encrypted during a running R session. R allows arbitrary
modification of code at runtime so this package provides no security
from the point where the data can be decrypted. If your computer was
compromised then stealing the data while you are running R should be
assumed to be straightforward.

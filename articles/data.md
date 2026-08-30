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

    ## Adding key b6:67:0a:ec:d2:9f:03:0a:de:5e:8d:d3:90:67:65:78:44:c1:97:cf:f1:f4:a6:04:f6:6e:7b:87:3e:72:38:2e
    ##   user: root
    ##   host: 980ca3fd58df
    ##   date: 2026-08-30 07:19:53.341381

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

    ## Already set up at /tmp/RtmpKp4HVf/data

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
    ##     hash: 51:22:16:06:f6:22:b8:87:2e:f2:12:03:db:96:3d:69:90:ae:d3:41:22:58:ba:a5:6a:ef:2c:c6:91:e4:26:2b
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
    ##   51:22:16:06:f6:22:b8:87:2e:f2:12:03:db:96:3d:69:90:ae:d3:41:22:58:ba:a5:6a:ef:2c:c6:91:e4:26:2b
    ##     user: root
    ##     host: 980ca3fd58df
    ##     date: 2026-08-30 07:19:53.969273

We can see the same hash here as above
(`51221606f622b8872ef21203db963d6990aed3412258baa56aef2cc691e4262b`)

…and then grant access to them with the
[`cyphr::data_admin_authorise`](https://docs.ropensci.org/cyphr/reference/data_admin.md)
function.

``` r

cyphr::data_admin_authorise(data_dir, yes = TRUE, path_user = path_key_alice)
```

    ## There is 1 request for access

    ## Adding key 51:22:16:06:f6:22:b8:87:2e:f2:12:03:db:96:3d:69:90:ae:d3:41:22:58:ba:a5:6a:ef:2c:c6:91:e4:26:2b
    ##   user: root
    ##   host: 980ca3fd58df
    ##   date: 2026-08-30 07:19:53.969273

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
    ##   51:22:16:06:f6:22:b8:87:2e:f2:12:03:db:96:3d:69:90:ae:d3:41:22:58:ba:a5:6a:ef:2c:c6:91:e4:26:2b
    ##     user: root
    ##     host: 980ca3fd58df
    ##     date: 2026-08-30 07:19:53.969273
    ##   b6:67:0a:ec:d2:9f:03:0a:de:5e:8d:d3:90:67:65:78:44:c1:97:cf:f1:f4:a6:04:f6:6e:7b:87:3e:72:38:2e
    ##     user: root
    ##     host: 980ca3fd58df
    ##     date: 2026-08-30 07:19:53.341381

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

    ## Adding key b6:67:0a:ec:d2:9f:03:0a:de:5e:8d:d3:90:67:65:78:44:c1:97:cf:f1:f4:a6:04:f6:6e:7b:87:3e:72:38:2e
    ##   user: root
    ##   host: 980ca3fd58df
    ##   date: 2026-08-30 07:19:54.441115

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
    ##     hash: 51:22:16:06:f6:22:b8:87:2e:f2:12:03:db:96:3d:69:90:ae:d3:41:22:58:ba:a5:6a:ef:2c:c6:91:e4:26:2b
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

    ## Adding key 51:22:16:06:f6:22:b8:87:2e:f2:12:03:db:96:3d:69:90:ae:d3:41:22:58:ba:a5:6a:ef:2c:c6:91:e4:26:2b
    ##   user: root
    ##   host: 980ca3fd58df
    ##   date: 2026-08-30 07:19:54.646914

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

    ## [1] "51221606f622b8872ef21203db963d6990aed3412258baa56aef2cc691e4262b"
    ## [2] "b6670aecd29f030ade5e8dd39067657844c197cff1f4a604f66e7b873e72382e"

``` r

names(cyphr::data_admin_list_keys(data_dir))
```

    ## [1] "51221606f622b8872ef21203db963d6990aed3412258baa56aef2cc691e4262b"
    ## [2] "b6670aecd29f030ade5e8dd39067657844c197cff1f4a604f66e7b873e72382e"

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
    ## [1] "980ca3fd58df"
    ## 
    ## $date
    ## [1] "2026-08-30 07:19:54 UTC"
    ## 
    ## $pub
    ## [2048-bit rsa public key]
    ## md5: e523b5820d0bcf6847e508b2944c6480
    ## sha256: 51221606f622b8872ef21203db963d6990aed3412258baa56aef2cc691e4262b
    ## 
    ## $key
    ##   [1] 06 1a c4 81 88 29 65 69 24 43 1c b9 7b ab ac b4 01 66 7b 99 4f c0 3a 04 46
    ##  [26] c9 fa c9 7d dc 1b 20 35 f2 79 77 67 53 6e 34 ad 76 50 66 0b 25 0b d3 7e 48
    ##  [51] fd c6 d1 4a 9a 14 d0 76 64 24 83 06 58 e2 ee be c1 e9 5d a7 db 27 bc bd d3
    ##  [76] 02 97 f6 3b 04 86 d7 b4 d4 df 7c 9c 12 1a e4 59 6f e8 d4 0a a7 6a b9 a2 92
    ## [101] 0b cb ab 81 84 d7 71 03 74 98 52 1e e5 cb 70 bc 46 17 4a a2 d7 ae 33 0f 3a
    ## [126] 0a 94 b6 35 c0 1f fa 5c cc 16 8b 80 a0 b3 32 76 54 c8 2b 4e 40 99 e1 c1 3f
    ## [151] 97 d8 46 22 2d 02 b5 4d 17 79 ec 3f 90 b0 3c 6d 05 e9 84 12 e6 57 b5 2b 64
    ## [176] 47 18 f5 ad dc 99 1f aa 54 82 d3 4e a7 b1 6a ea eb 1f 07 0f 3a 7c 73 4d 37
    ## [201] 84 98 ee 40 15 d4 5d 12 ff d3 1f 48 83 93 74 fc c2 e4 2a 06 22 ce a5 dc d8
    ## [226] 98 3f 47 d0 ff c1 f9 bb d6 1c ef 49 c1 cc ad 34 a6 56 63 d6 1e ec 63 46 67
    ## [251] e8 54 a8 e1 c3 67

You can see that the hash of the public key is the same as name of the
stored file here (which is used to prevent collisions when multiple
people request access at the same time).

``` r

h
```

    ## [1] "51221606f622b8872ef21203db963d6990aed3412258baa56aef2cc691e4262b"

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

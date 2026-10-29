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

    ## Adding key ef:a3:19:be:92:a2:58:4a:a1:0b:c3:e3:b6:ea:a7:d1:20:58:b7:06:20:dc:81:1c:92:1d:f2:0b:61:8f:2b:38
    ##   user: root
    ##   host: ce6c4b0ffaba
    ##   date: 2026-10-29 07:48:06.415286

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

    ## Already set up at /tmp/Rtmp6apXTy/data

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
    ##     hash: 50:a9:4e:83:0f:5c:f6:fa:59:85:fd:c4:93:bd:f7:27:ae:2c:a0:bd:5d:42:15:4f:e7:c9:c7:b8:f5:ed:34:aa
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
    ##   50:a9:4e:83:0f:5c:f6:fa:59:85:fd:c4:93:bd:f7:27:ae:2c:a0:bd:5d:42:15:4f:e7:c9:c7:b8:f5:ed:34:aa
    ##     user: root
    ##     host: ce6c4b0ffaba
    ##     date: 2026-10-29 07:48:07.039611

We can see the same hash here as above
(`50a94e830f5cf6fa5985fdc493bdf727ae2ca0bd5d42154fe7c9c7b8f5ed34aa`)

…and then grant access to them with the
[`cyphr::data_admin_authorise`](https://docs.ropensci.org/cyphr/reference/data_admin.md)
function.

``` r

cyphr::data_admin_authorise(data_dir, yes = TRUE, path_user = path_key_alice)
```

    ## There is 1 request for access

    ## Adding key 50:a9:4e:83:0f:5c:f6:fa:59:85:fd:c4:93:bd:f7:27:ae:2c:a0:bd:5d:42:15:4f:e7:c9:c7:b8:f5:ed:34:aa
    ##   user: root
    ##   host: ce6c4b0ffaba
    ##   date: 2026-10-29 07:48:07.039611

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
    ##   50:a9:4e:83:0f:5c:f6:fa:59:85:fd:c4:93:bd:f7:27:ae:2c:a0:bd:5d:42:15:4f:e7:c9:c7:b8:f5:ed:34:aa
    ##     user: root
    ##     host: ce6c4b0ffaba
    ##     date: 2026-10-29 07:48:07.039611
    ##   ef:a3:19:be:92:a2:58:4a:a1:0b:c3:e3:b6:ea:a7:d1:20:58:b7:06:20:dc:81:1c:92:1d:f2:0b:61:8f:2b:38
    ##     user: root
    ##     host: ce6c4b0ffaba
    ##     date: 2026-10-29 07:48:06.415286

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

    ## Adding key ef:a3:19:be:92:a2:58:4a:a1:0b:c3:e3:b6:ea:a7:d1:20:58:b7:06:20:dc:81:1c:92:1d:f2:0b:61:8f:2b:38
    ##   user: root
    ##   host: ce6c4b0ffaba
    ##   date: 2026-10-29 07:48:07.534462

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
    ##     hash: 50:a9:4e:83:0f:5c:f6:fa:59:85:fd:c4:93:bd:f7:27:ae:2c:a0:bd:5d:42:15:4f:e7:c9:c7:b8:f5:ed:34:aa
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

    ## Adding key 50:a9:4e:83:0f:5c:f6:fa:59:85:fd:c4:93:bd:f7:27:ae:2c:a0:bd:5d:42:15:4f:e7:c9:c7:b8:f5:ed:34:aa
    ##   user: root
    ##   host: ce6c4b0ffaba
    ##   date: 2026-10-29 07:48:07.734985

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

    ## [1] "50a94e830f5cf6fa5985fdc493bdf727ae2ca0bd5d42154fe7c9c7b8f5ed34aa"
    ## [2] "efa319be92a2584aa10bc3e3b6eaa7d12058b70620dc811c921df20b618f2b38"

``` r

names(cyphr::data_admin_list_keys(data_dir))
```

    ## [1] "50a94e830f5cf6fa5985fdc493bdf727ae2ca0bd5d42154fe7c9c7b8f5ed34aa"
    ## [2] "efa319be92a2584aa10bc3e3b6eaa7d12058b70620dc811c921df20b618f2b38"

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
    ## [1] "ce6c4b0ffaba"
    ## 
    ## $date
    ## [1] "2026-10-29 07:48:07 UTC"
    ## 
    ## $pub
    ## [2048-bit rsa public key]
    ## md5: f87da4dbadf7be1559bba07dda36ad38
    ## sha256: 50a94e830f5cf6fa5985fdc493bdf727ae2ca0bd5d42154fe7c9c7b8f5ed34aa
    ## 
    ## $key
    ##   [1] a6 24 f9 ce 89 00 aa 26 a0 7b 51 13 02 6d 65 72 23 3b 5a c7 fa da 36 86 1c
    ##  [26] 0f 41 83 e9 1d be fa b7 c2 1a d3 3f 1c 04 01 3e 16 e4 61 fc ab 0e de d8 2e
    ##  [51] e3 61 41 bc bf 6c 99 85 40 a8 a2 4c 51 7f 74 d1 43 e3 19 c8 c6 bc c0 1c 5a
    ##  [76] b7 4f 0f 85 83 36 ef 4f ba 8d e5 79 7f 39 71 51 aa e7 d9 22 d6 1b 28 7f 37
    ## [101] ac c0 3c 89 de 27 84 f0 fb 00 5a eb 0e 84 0c af b0 6c e9 81 f5 75 71 7c fd
    ## [126] 3d 1d 83 fe fc b8 0a ac b9 0b e7 66 10 0a 1c 85 a4 fc 81 c9 f0 aa c2 55 a1
    ## [151] f1 1b d9 5a df 48 e8 65 fb c8 e4 99 86 a6 2a 74 30 12 1e c9 29 10 d3 19 be
    ## [176] b9 b1 b3 af 6e 89 14 b1 f5 ce b6 cb c0 e0 7d c2 91 d6 bf 33 30 38 22 c3 39
    ## [201] fc 91 c7 f8 98 fe ab 71 e9 49 22 4e 67 75 ec ad 3d 81 41 0b 1b 63 3f 2c df
    ## [226] 89 f3 e4 31 4a 26 db 4b d4 e0 86 df 9d 38 95 69 23 ab e5 34 42 fb 77 1d 9f
    ## [251] 3e 02 ec ea 29 0c

You can see that the hash of the public key is the same as name of the
stored file here (which is used to prevent collisions when multiple
people request access at the same time).

``` r

h
```

    ## [1] "50a94e830f5cf6fa5985fdc493bdf727ae2ca0bd5d42154fe7c9c7b8f5ed34aa"

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

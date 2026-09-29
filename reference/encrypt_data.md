# Encrypt and decrypt data and other things

Encrypt and decrypt raw data, objects, strings and files. The core
functions here are `encrypt_data` and `decrypt_data` which take raw data
and decrypt it, writing either to file or returning a raw vector. The
other functions encrypt and decrypt arbitrary R objects
(`encrypt_object`, `decrypt_object`), strings (`encrypt_string`,
`decrypt_string`) and files (`encrypt_file`, `decrypt_file`).

## Usage

``` r
encrypt_data(data, key, dest = NULL)

encrypt_object(object, key, dest = NULL, rds_version = NULL)

encrypt_string(string, key, dest = NULL)

encrypt_file(path, key, dest = NULL)

decrypt_data(data, key, dest = NULL)

decrypt_object(data, key)

decrypt_string(data, key)

decrypt_file(path, key, dest = NULL)
```

## Arguments

- data:

  (for `encrypt_data`, `decrypt_data`, `decrypt_object`,
  `decrypt_string`) a raw vector with the data to be encrypted or
  decrypted. For the decryption functions this must be data derived by
  encrypting something or you will get an error.

- key:

  A `cyphr_key` object describing the encryption approach to use.

- dest:

  The destination filename for the encrypted or decrypted data, or
  `NULL` to return a raw vector. This is not used by `decrypt_object` or
  `decrypt_string` which always return an object or string.

- object:

  (for `encrypt_object`) an arbitrary R object to encrypt. It will be
  serialised to raw first (see
  [serialize](https://rdrr.io/r/base/serialize.html)).

- rds_version:

  RDS serialisation version to use (see
  [serialize](https://rdrr.io/r/base/serialize.html). The default in R
  version 3.3 and below is version 2 - in the R 3.4 series version 3 was
  introduced and is becoming the default. Version 3 format serialisation
  is not understood by older versions so if you need to exchange data
  with older R versions, you will need to use `rds_version = 2`. The
  default argument here (`NULL`) will ensure the same serialisation is
  used as R would use by default.

- string:

  (for `encrypt_string`) a scalar character vector to encrypt. It will
  be converted to raw first with
  [charToRaw](https://rdrr.io/r/base/rawConversion.html).

- path:

  (for `encrypt_file`) the name of a file to encrypt. It will first be
  read into R as binary (see
  [readBin](https://rdrr.io/r/base/readBin.html)).

## Examples

``` r
key <- key_sodium(sodium::keygen())
# Some super secret data we want to encrypt:
x <- runif(10)
# Convert the data into a raw vector:
data <- serialize(x, NULL)
data
#>   [1] 58 0a 00 00 00 03 00 04 06 00 00 03 05 00 00 00 00 05 55 54 46 2d 38 00 00
#>  [26] 00 0e 00 00 00 0a 3f b4 ac 0a 80 00 00 00 3f ea b2 db 32 a0 00 00 3f e3 39
#>  [51] 6e e4 e0 00 00 3f c4 1f 67 fd 80 00 00 3f 7e 4e e0 60 00 00 00 3f dd d9 64
#>  [76] 1c 80 00 00 3f df db 95 b1 40 00 00 3f d2 8b 8b e9 c0 00 00 3f e7 73 c4 ec
#> [101] c0 00 00 3f e8 b8 7f 08 40 00 00
# Encrypt the data; without the key above we will never be able to
# decrypt this.
data_enc <- encrypt_data(data, key)
data_enc
#>   [1] ba 22 80 10 db b8 7d b8 48 fa e4 c8 cb 83 94 87 4c ca 74 9c 00 db f1 f5 7b
#>  [26] f8 0d c8 8c f6 7e 49 3b 31 7b d1 29 25 21 1f b6 5f 78 8a da 7d a3 2b 30 08
#>  [51] fc 55 82 ac f3 e1 49 4f 63 ff 21 6e 48 f6 67 e6 77 e6 37 55 35 a7 55 d8 b7
#>  [76] 2c db a3 a7 f1 b3 35 15 9b 3a 69 ab ca 2b 75 86 59 b0 d8 98 ef 96 0b ca e2
#> [101] b6 e9 ef 30 25 96 d7 12 1f ec 34 3b ae 60 ed cd 18 fb 4a 73 71 71 f7 d7 7d
#> [126] 81 e7 e0 d4 1b e2 b3 80 4e 8b a2 4f 99 75 3a 20 59 d3 45 da 82 e6 87 ab 67
#> [151] 37
# Our random numbers:
unserialize(decrypt_data(data_enc, key))
#>  [1] 0.080750138 0.834333037 0.600760886 0.157208442 0.007399441 0.466393497
#>  [7] 0.497777389 0.289767245 0.732881987 0.772521511
# Same as the never-encrypted version:
x
#>  [1] 0.080750138 0.834333037 0.600760886 0.157208442 0.007399441 0.466393497
#>  [7] 0.497777389 0.289767245 0.732881987 0.772521511

# This can be achieved more easily using `encrypt_object`:
data_enc <- encrypt_object(x, key)
identical(decrypt_object(data_enc, key), x)
#> [1] TRUE

# Encrypt strings easily:
str_enc <- encrypt_string("secret message", key)
str_enc
#>  [1] 58 17 eb f2 2d 05 fb f0 c3 fd 45 eb 5b 37 23 24 0c 29 8f 47 cb be 90 a8 2a
#> [26] af 56 0b f2 fd 79 03 dc b0 72 9a 8b 10 54 ad 29 53 d7 55 e9 27 6d 0d cf f2
#> [51] 1b 9d 83 59
decrypt_string(str_enc, key)
#> [1] "secret message"
```

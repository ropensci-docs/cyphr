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
#>   [1] 90 69 06 08 ae 64 19 95 3f b5 66 85 07 c0 14 6f a7 1c 89 85 62 76 43 82 bc
#>  [26] f0 34 c3 ba 8a ce f9 a0 f4 e8 a6 b2 4d 87 be da f9 62 d5 22 51 25 b4 33 62
#>  [51] cb f4 2b 86 29 88 2d 14 9c 36 fc 34 66 21 78 48 05 35 73 25 5e 49 d3 a5 fe
#>  [76] f4 4e 2c 76 0b 3b c3 e8 7f bb cd d9 b3 3b 81 04 43 dc 2a f0 a5 e5 78 cf 0a
#> [101] 6d 52 09 ec ae 4b 38 6a da 93 b1 ed 3f 68 a4 81 8c 9e 0a 1c ee c0 c5 5e 2e
#> [126] 4f df b3 37 e2 d1 bc 57 94 6c a0 6f 12 79 87 dc b7 c4 e7 4f c9 0d 1c 5b 60
#> [151] fb
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
#>  [1] f0 09 2e fc 40 de 56 cb 9e 2f 74 08 1f 4e de 52 00 d2 e3 77 90 c8 e5 9f a1
#> [26] 57 c9 ae 63 5f cd 71 09 6b 12 8b 57 9d f5 c2 7b c6 df fa 8a 09 3e 4e 6d f7
#> [51] 06 00 cc 4d
decrypt_string(str_enc, key)
#> [1] "secret message"
```

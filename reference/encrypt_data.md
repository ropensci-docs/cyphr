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
#>   [1] 19 77 b8 03 b8 75 57 48 3d cd d3 ea 35 ef 23 21 01 27 9b 3b 4b a4 e4 35 39
#>  [26] 1c d3 f8 5e d2 b2 b0 8d 29 05 df e7 c9 d1 c2 cf 3f bc 75 08 5c 30 d0 b1 6f
#>  [51] 0d 99 a9 dd d0 5a aa a7 cc b8 03 5b ca d6 81 18 60 bb 16 40 a9 0e af 1a d3
#>  [76] 09 5a 5a 45 c8 f1 db 5e 3f 40 cb fc f6 ff 5a 65 00 63 e5 d1 38 96 5c 76 89
#> [101] 17 b8 37 f7 99 12 99 08 d9 4c 4a 49 2b 27 b9 ce 53 51 5e 13 6c 43 de 2a 6b
#> [126] 25 a7 0d 4d 4d 45 fe c6 d0 86 ab 67 21 9a cb b4 52 6e 47 ad 8b e1 ad 4c e6
#> [151] c4
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
#>  [1] 94 b8 de f1 7f 83 17 fa 6d 10 b1 02 63 56 b7 8c f8 b0 ce e5 9b d5 78 d1 32
#> [26] 16 25 96 9a 22 ae f7 7a 91 e2 96 c0 27 a3 92 e5 ab b1 b0 9b e7 ec b2 08 ec
#> [51] 82 0f 6b a3
decrypt_string(str_enc, key)
#> [1] "secret message"
```

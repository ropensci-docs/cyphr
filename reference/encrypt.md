# Easy encryption and decryption

Wrapper functions for encryption. These functions wrap expressions that
produce or consume a file and arrange to encrypt (for producing
functions) or decrypt (for consuming functions). The forms with a
trailing underscore (`encrypt_`, `decrypt_`) do not use any non-standard
evaluation and may be more useful for programming.

## Usage

``` r
encrypt(expr, key, file_arg = NULL, envir = parent.frame())

decrypt(expr, key, file_arg = NULL, envir = parent.frame())

encrypt_(expr, key, file_arg = NULL, envir = parent.frame())

decrypt_(expr, key, file_arg = NULL, envir = parent.frame())
```

## Arguments

- expr:

  A single expression representing a function call that would be called
  for the side effect of creating or reading a file.

- key:

  A `cyphr_key` object describing the encryption approach to use.

- file_arg:

  Optional hint indicating which argument to `expr` is the filename.
  This is done automatically for some built-in functions.

- envir:

  Environment in which `expr` is to be evaluated.

## Details

These functions will not work for all functions. For example
`pdf`/`dev.off` will create a file but we can't wrap those up (yet!).
Functions that *modify* a file (e.g., appending) also will not work and
may cause data loss.

## Examples

``` r
# To do anything we first need a key:
key <- cyphr::key_sodium(sodium::keygen())

# Encrypted write.csv - note how any number of arguments to
# write.csv will be passed along
path <- tempfile(fileext = ".csv")
cyphr::encrypt(write.csv(iris, path, row.names = FALSE), key)

# The new file now exists, but you would not be able to read it
# with read.csv because it is now binary data.
file.exists(path)
#> [1] TRUE

# Wrap the read.csv call with cyphr::decrypt()
dat <- cyphr::decrypt(read.csv(path, stringsAsFactors = FALSE), key)
head(dat)
#>   Sepal.Length Sepal.Width Petal.Length Petal.Width Species
#> 1          5.1         3.5          1.4         0.2  setosa
#> 2          4.9         3.0          1.4         0.2  setosa
#> 3          4.7         3.2          1.3         0.2  setosa
#> 4          4.6         3.1          1.5         0.2  setosa
#> 5          5.0         3.6          1.4         0.2  setosa
#> 6          5.4         3.9          1.7         0.4  setosa

file.remove(path)
#> [1] TRUE

# If you have a function that is not supported you can specify the
# filename argument directly.  For example, with "write.dcf" the
# filename argument is called "file"; we can pass that along
path <- tempfile()
cyphr::encrypt(write.dcf(list(a = 1), path), key, file_arg = "file")

# Similarly for decryption:
cyphr::decrypt(read.dcf(path), key, file_arg = "file")
#>      a  
#> [1,] "1"
```

# Register functions to work with encrypt/decrypt

Add information about argument rewriting so that they can be used with
[encrypt](https://docs.ropensci.org/cyphr/reference/encrypt.md) and
[decrypt](https://docs.ropensci.org/cyphr/reference/encrypt.md).

## Usage

``` r
rewrite_register(package, name, arg, fn = NULL)
```

## Arguments

- package:

  The name of the package with the function to support (as a scalar
  character). If your function has no package (e.g., a function you are
  working on outside of a package, use "" as the name).

- name:

  The name of the function to support.

- arg:

  The name of the argument in the target function that refers to the
  file that should be encrypted or decrypted. This is the value you
  would pass through to `file_arg` in
  [encrypt](https://docs.ropensci.org/cyphr/reference/encrypt.md).

- fn:

  Optional (and should be rare) argument used to work around functions
  that pass all their arguments through to a second function as dots.
  This is how `read.csv` works. If needed this function is a length-2
  character vector in the form "package", "name" with the actual
  function that is used. But this should be very rare!

## Details

If your package uses cyphr, it might be useful to add this as an
`.onLoad()` hook.

## Examples

``` r
# The saveRDS function is already supported.  But if we wanted to
# support it we could look at the arguments for the function:
args(saveRDS)
#> function (object, file = "", ascii = FALSE, version = NULL, compress = TRUE, 
#>     refhook = NULL) 
#> NULL
# The 'file' argument is the one that refers to the filename, so
# we'd write:
cyphr::rewrite_register("base", "saveRDS", "file")
# It's non-API but you can see what is supported in the package by
# looking at
ls(cyphr:::db)
#>  [1] "base::load"          "base::readLines"     "base::readRDS"      
#>  [4] "base::save"          "base::saveRDS"       "base::writeLines"   
#>  [7] "readxl::read_excel"  "readxl::read_xls"    "readxl::read_xlsx"  
#> [10] "utils::read.csv"     "utils::read.csv2"    "utils::read.delim"  
#> [13] "utils::read.delim2"  "utils::read.table"   "utils::write.csv"   
#> [16] "utils::write.csv2"   "utils::write.table"  "writexl::write_xlsx"
```

# qs [![Build Status](https://travis-ci.org/traversc/qs.svg)](https://travis-ci.org/traversc/qs) [![CRAN\_Status\_Badge](http://www.r-pkg.org/badges/version/qs)](https://cran.r-project.org/package=qs)

*Quick serialization of R objects*

`qs` provides an interface for quickly saving (serializing) and reading (de-serializing) objects to and from disk. The goal of this package is to provide a lightning-fast and complete replacement for the `saveRDS` and `readRDS` functions in R.

Inspired by the `fst` package, `qs` uses a similar block-compression design using either the `lz4` or `zstd` compression libraries. It differs in that it applies a more general approach for attributes and object references.  

`saveRDS` and `readRDS` are the standard for serializing R data, but these functions are not optimized for speed.  On the other hand, `fst` is extremely fast, but only works on `data.frame`'s and certain column types.  

`qs` is both  extremely fast and general: it can serialize any R object like `saveRDS` and is just as fast and sometimes faster than `fst`.

## Usage

```r
library(qs)
df1 <- data.frame(x=rnorm(5e6), y=sample(5e6), z=sample(letters,5e6))
qsave(df1, "myfile.q")
df2 <- qread("myfile.q")
```
## Installation:
For R version 3.5 or higher:

```r
install.packages("qs") ## CRAN version
remotes::install_github("traversc/qs") ## Latest version
```
For R version 3.4 and lower:

```r
remotes::install_github("traversc/qs", ref = "qs34")
```
For Windows users, I recommend installing from source rather than using the pre-compiled library. Adding `PKG_CPPFLAGS+=-mavx2` to the R makefile (`C:\Users\<user>\Documents\.R\Makevars`) will increase serialization speed. 
## Features
The table below compares the features of different serialization approaches in R.

|                    | qs         | fst           | saveRDS  |
|--------------------|:-----------:|:---------------:|:----------:|
| Not Slow             | &#10004;   | &#10004;       | &#10060; |
| Numeric Vectors    | &#10004;   | &#10004;       | &#10004;  |
| Integer Vectors    | &#10004;   | &#10004;       | &#10004;  |
| Logical Vectors    | &#10004;   | &#10004;       | &#10004;  |
| Character Vectors  | &#10004;   | &#10004;       | &#10004;  |
| Character Encoding | &#10004;   | (vector-wide only) | &#10004;  |
| Complex Vectors    | &#10004;   | &#10060;      | &#10004;  |
| Data.Frames        | &#10004;   | &#10004;       | &#10004;  |
| On disk row access | &#10060;  | &#10004;       | &#10060; |
| Attributes         | &#10004;   | Some          | &#10004;  |
| Lists / Nested Lists| &#10004;   |  &#10060;     | &#10004;  |
| Multi-threaded     | &#10060; (Not Yet) | &#10004;      |  &#10060;   |


`qs` also includes a number of advanced features:

* For character vectors, qs also has the option of using the new alt-rep system (R version 3.5+) to quickly read in string data.
* For numerical data (numeric, integer, logical and complex vectors) `qs` implements byte shuffling filters (adopted from the Blosc meta-compression library).  These filters utilize extended CPU instruction sets (either SSE2 or AVX2).

Both of these features have the possibility of additionally increasing performance by orders of magnitude, for certain types of data.  See sections below for more details.  
## Summary Benchmarks

These benchmarks were performed on a desktop using various data types (detailed below). `qs` was comapared with `saveRDS`/`readRDS` in base R and the `fst` package for serializing `data.frame`'s.  In terms of speed, `qs` is  comparable to `fst` for the data types `fst` supports, and both packages are orders of magnitude faster than `saveRDS`.  

`qs` is highly parameterized and can be tuned by the user to extract as much speed and compression as possible.  For simplicity, `qs` comes with 3 presets, which trades speed for compression: "fast", "balanced" and "high".  The benchmark below uses the "fast" preset.  

![](headline_bench.png "headline_bench"){width=500px}

* character: `randomStrings(1e6)`
* numeric:`rnorm(1e7)`
* integer: `sample(1e7)`
* list: `map(1:1e4, ~sample(100))`
* environment: `map(1:1e4, ~sample(100)) %>% setNames(1:1e4) %>% as.environment`


Benchmarking write and read speed is a bit tricky and depends highly on a number of factors, such as which operating system one uses, the hardware being run on, the distribution of the data, or even whether R is restarted before each measurement.  Reading data is also subject to various hardware and software memory caches, and so may not be fully representative of reading data straight from disk.  

## Data.Frame benchmark

For a more complete comparison with `fst`, a number of different parameters were evaluated for each method.  For `fst`, compression level and number of threads were varied and plotted.  For `qs`, byte shuffle settings (see the "Byte Shuffling" section below) and algorithm used were evaluated.  A `data.frame` with 5 million rows was used for the purpose of this evaluation. 

![](dataframe_bench.png "dataframe_bench"){width=500px}

The `data.frame` used is as follows:

```r
data.frame(a=rnorm(5e6), b=rpois(100,5e6),
           c=sample(starnames[["IAU Name"]],5e6,T), 
           stringsAsFactors = F)
```

This benchmark also includes materialization of alt-rep data, for an apples-to-apples comparison.  

## Byte Shuffle

Byte shuffling (adapted from the Blosc meta-compression library) is a way of re-organizing data to be more ammenable to compression.  Consider integer data: an integer contains four bytes and the limits of an integer in R are +/- 2^31-1.  However, most real data doesn't use anywhere near the range of possible integer values.  For example, if the data were representing percentages, 0% to 100%, the first three out of the four bytes would be unused and zero.    

Byte shuffling rearranges the data such that all of the first bytes are blocked together, the second bytes are blocked together, etc., which makes it very easy for compression algorithms to find repeated patterns.  This often improves compression by orders of magnitude; in the example below, shuffle compression achieves a compression ratio of over 1000x.  See `?qsave` for more details. 

Example:

```r
# With byte shuffling
x <- 1:1e8
qsave(x, "mydat.q", preset="custom", shuffle_control=15, algorithm="zstd")
cat( "Compression Ratio: ", as.numeric(object.size(x)) / file.info("mydat.q")$size, "\n" )
# Compression Ratio:  1389.164

# Without byte shuffling
x <- 1:1e8
qsave(x, "mydat.q", preset="custom", shuffle_control=0, algorithm="zstd")
cat( "Compression Ratio: ", as.numeric(object.size(x)) / file.info("mydat.q")$size, "\n" )
# Compression Ratio:  1.479294 
```

## Alt-rep character vectors

The alt-rep system was introduced in R version 3.5.  Briefly, alt-rep vectors are objects that are not represented by R internal data, but have accesor functions which promise to "materialize" elements within the vector on the fly.  The amazing thing about the system is that it's completely seamless to the user.  

Only alt-rep character vectors are implemented in `qs` because it is often the mostly costly of data types to read into R.  Numeric and integer data are already fast enough and do not benefit.  An example use case: if you have a large `data.frame`, and you are only interested in processing certain columns, it is wasted computatoin to materialize the whole `data.frame`.  The alt-rep system solves this problem.  


```r
df1 <- data.frame(x = randomStrings(1e6), y = randomStrings(1e6), stringsAsFactors = F)
qsave(df1, "temp.q")
rm(df1); gc() ## remove df1 and call gc for proper benchmarking

# With alt-rep
system.time(qread("temp.q", use_alt_rep=T))[1]
#     0.109 seconds


# Without alt-rep
gc(verbose=F)
system.time(qread("temp.q", use_alt_rep=F))[1]
#     1.703 seconds
```

## Future Developments

* Multi-threading
* Parameter optimization

Future versions will be backwards compatible with the current version.  

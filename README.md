---
title: "Modules in R"
author: "Sebastian Warnholz"
date: "2015-11-12"
output: rmarkdown::html_vignette
vignette: >
  %\VignetteIndexEntry{Modules in R}
  %\VignetteEngine{knitr::rmarkdown}
  %\VignetteEncoding{UTF-8}
---

[![Build Status](https://travis-ci.org/wahani/module.png?branch=master)](https://travis-ci.org/wahani/module)
[![codecov.io](https://codecov.io/github/wahani/module/coverage.svg?branch=master)](https://codecov.io/github/wahani/module?branch=master)
[![CRAN](http://www.r-pkg.org/badges/version/module)](http://cran.rstudio.com/package=module)
[![Downloads](http://cranlogs.r-pkg.org/badges/module?color=brightgreen)](http://www.r-pkg.org/pkg/module)

Provides modules as an organizational unit for source code. Modules enforce to be more rigerous when defining dependencies and have something like a local search path. They are to be used as a sub unit within packages or in scripts.

## Installation

From this GitHub:


```r
devtools::install_github("wahani/aoos")
```

# Modules Outside of Package Development

## General Behviour

The key idea of this package is to provide a unit which is self contained, i.e.
has it's own scope. The main and most reliable infrastructure for such
organizational units of source code in the R ecosystem is a package. Compared to
a pacakge modules can be considered ad-hoc, but - in the sense of an R-package -
self contained. 

There are two use cases. First when you use modules to devolop scripts, which is
subject of this section. And then inside of packages where the scope is a bit
different by default. Outside of packages modules know only of the base
environment, i.e. the search path ends there. Also they are allways represented
as a list inside R.


```r
library(module)
m <- module({
  boringFunction <- function() cat("boring output")
})

m$boringFunction()
```

```
## boring output
```

Since they are isolated from the `.GlobalEnv` the following object `hey` can not
be found:


```r
hey <- "hey"
m <- module({
  isolatedFunction <- function() hey
})
m$isolatedFunction()
```

```
## Error in m$isolatedFunction(): object 'hey' not found
```

## Imports

If you rely on exported objects of packages you can refer to them explicitly
using `::`:


```r
m <- module({
  functionWithDep <- function(x) stats::median(x)
})
m$functionWithDep(1:10)
```

```
## [1] 5.5
```

Or you can use one of several options to import objects in the local scope:


```r
m <- module({
 
  import(stats, median) # make median from package stats available
  
  functionWithDep <- function(x) median(x)

})
m$functionWithDep(1:10)
```

```
## [1] 5.5
```


```r
m <- module({
  
  use("stats") # a package or a list, i.e. a module
  
  functionWithDep <- function(x) median(x)

})
m$functionWithDep(1:10)
```

```
## [1] 5.5
```

## Exports

It may also be of interest to control which objects are visible for the client.
You can do that with the `export` function. Note that export accepts regular
expressions. If you supply more than one argument they are connected with '|'.
Functions which begin with a `.` are never exported.


```r
m <- module({
  
  export("fun")
  
  .privateFunction <- identity
  privateFunction <- identity
  fun <- identity
  
})

names(m)
```

```
## [1] "fun"
```

## Example: Modules as Parallel Process

One example where you may want to have more control of the enclosing environment 
of a function is when you parallelize your code. First consider the case when a 
*naive* implementation fails.


```r
library(parallel)
dependency <- identity
fun <- function(x) dependency(x) 

cl <- makeCluster(2)
clusterMap(cl, fun, replicate(2, 1:10, simplify = FALSE))
```

```
## Error in checkForRemoteErrors(val): 2 nodes produced errors; first error: could not find function "dependency"
```

```r
stopCluster(cl)
```

To make the function `fun` self contained we can define it in a module. 


```r
m <- module({
  dependency <- identity
  fun <- function(x) dependency(x) 
})

cl <- makeCluster(2)
clusterMap(cl, m$fun, replicate(2, 1:10, simplify = FALSE))
```

```
## [[1]]
##  [1]  1  2  3  4  5  6  7  8  9 10
## 
## [[2]]
##  [1]  1  2  3  4  5  6  7  8  9 10
```

```r
stopCluster(cl)
```


# Modules with Object Orientation

I have trouble with S3, default S4. Most of syntactic sugar in aoos works:


```r
m <- module({
  use("methods")
  use("aoos")
  gen(x) %g% cat("default method")
  gen(x ~ numeric) %m% cat("method for x ~ numerc")
})
m$gen("Hej")
```

```
## default method
```

```r
m$gen(1)
```

```
## method for x ~ numerc
```

S4 classes or types in aoos are not working correctly because S4 makes it
terrible hard to understand the side effects of the implementation. Until now I
was not able to isolate that. `setOldClass` with an appropriate where statement
seems to work though:


```r
m <- module({
  use("methods")
  use("aoos")
  gen(x) %g% cat("default method")
  gen(x ~ numeric) %m% cat("method for x ~ numerc")
  NewType <- function(x) {
    retList("NewType")
  }
  setOldClass("NewType", where = environment())
  gen(x ~ NewType) %m% cat("method for x ~ NewType")
})

cl <- makeCluster(1)
clusterMap(cl, m$gen, m$NewType(NULL))
```

```
## $x
## NULL
```

```r
stopCluster(cl)
```




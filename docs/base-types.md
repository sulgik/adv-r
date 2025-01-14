# Base types {#base-types}

## Introduction
\index{base objects}
\index{OO objects}



To talk about objects and OOP in R we first need to clear up a fundamental confusion about two uses of the word "object". So far in this book, we've used the word in the general sense captured by John Chambers' pithy quote: "Everything that exists in R is an object". However, while everything _is_ an object, not everything is object-oriented. This confusion arises because the base objects come from S, and were developed before anyone thought that S might need an OOP system. The tools and nomenclature evolved organically over many years without a single guiding principle.

Most of the time, the distinction between objects and object-oriented objects is not important. But here we need to get into the nitty gritty details so we'll use the terms __base objects__ and __OO objects__ to distinguish them.


\begin{center}\includegraphics[width=2.21in]{diagrams/oo-venn} \end{center}

### Outline {-} 

* Section \@ref(base-vs-oo) shows you how to identify base and OO objects.

* Section \@ref(base-types-2) gives a complete set of the base types used to build all objects.

## Base versus OO objects {#base-vs-oo}
\indexc{is.object()}
\indexc{otype()}
\index{attributes!class}
\indexc{class()}

To tell the difference between a base and OO object, use `is.object()` or `sloop::otype()`:


```r
# A base object:
is.object(1:10)
#> [1] FALSE
sloop::otype(1:10)
#> [1] "base"

# An OO object
is.object(mtcars)
#> [1] TRUE
sloop::otype(mtcars)
#> [1] "S3"
```

Technically, the difference between base and OO objects is that OO objects have a "class" attribute:


```r
attr(1:10, "class")
#> NULL

attr(mtcars, "class")
#> [1] "data.frame"
```

You may already be familiar with the `class()` function. This function is safe to apply to S3 and S4 objects, but it returns misleading results when applied to base objects. It's safer to use `sloop::s3_class()`, which returns the implicit class that the S3 and S4 systems will use to pick methods. You'll learn more about `s3_class()` in Section \@ref(implicit-class).


```r
x <- matrix(1:4, nrow = 2)
class(x)
#> [1] "matrix" "array"
sloop::s3_class(x)
#> [1] "matrix"  "integer" "numeric"
```

## Base types {#base-types-2}
\indexc{typeof()}
\index{base type|see {\texttt{typeof()}}}

While only OO objects have a class attribute, every object has a __base type__:


```r
typeof(1:10)
#> [1] "integer"

typeof(mtcars)
#> [1] "list"
```

Base types do not form an OOP system because functions that behave differently for different base types are primarily written in C code that uses switch statements. This means that only R-core can create new types, and creating a new type is a lot of work because every switch statement needs to be modified to handle a new case. As a consequence, new base types are rarely added. The most recent change, in 2011, added two exotic types that you never see in R itself, but are needed for diagnosing memory problems. Prior to that, the last type added was a special base type for S4 objects added in 2005.

<!-- 
https://github.com/wch/r-source/blob/f5bb85782509ddadbcec94ab7648886c2d008bda/src/main/util.c#L185-L211-->

In total, there are 25 different base types. They are listed below, loosely grouped according to where they're discussed in this book. These types are most important in C code, so you'll often see them called by their C type names. I've included those in parentheses.

*   Vectors, Chapter \@ref(vectors-chap), include types `NULL` (`NILSXP`), 
    `logical` (`LGLSXP`), `integer` (`INTSXP`), `double` (`REALSXP`), `complex`
    (`CPLXSXP`), `character` (`STRSXP`), `list` (`VECSXP`), and `raw` (`RAWSXP`).
    
    
    ```r
    typeof(NULL)
    #> [1] "NULL"
    typeof(1L)
    #> [1] "integer"
    typeof(1i)
    #> [1] "complex"
    ```

*   Functions, Chapter \@ref(functions), include types `closure` (regular R 
    functions, `CLOSXP`), `special` (internal functions, `SPECIALSXP`), and 
    `builtin` (primitive functions, `BUILTINSXP`).
    
    
    ```r
    typeof(mean)
    #> [1] "closure"
    typeof(`[`)
    #> [1] "special"
    typeof(sum)    
    #> [1] "builtin"
    ```
    
    Internal and primitive functions are described in Section 
    \@ref(primitive-functions).

*   Environments, Chapter \@ref(environments), have type `environment` 
    (`ENVSXP`).

    
    ```r
    typeof(globalenv())
    #> [1] "environment"
    ```

*   The `S4` type (`S4SXP`), Chapter \@ref(s4), is used for S4 classes that 
    don't inherit from an existing base type.
   
    
    ```r
    mle_obj <- stats4::mle(function(x = 1) (x - 2) ^ 2)
    typeof(mle_obj)
    #> [1] "S4"
    ```

*   Language components, Chapter \@ref(expressions), include `symbol` (aka 
    name, `SYMSXP`), `language` (usually called calls, `LANGSXP`), and 
    `pairlist` (used for function arguments, `LISTSXP`) types.

    
    ```r
    typeof(quote(a))
    #> [1] "symbol"
    typeof(quote(a + 1))
    #> [1] "language"
    typeof(formals(mean))
    #> [1] "pairlist"
    ```
 
    `expression` (`EXPRSXP`) is a special purpose type that's only returned by
    `parse()` and `expression()`. Expressions are generally not needed in user 
    code.
 
*   The remaining types are esoteric and rarely seen in R. They are important 
    primarily for C code: `externalptr` (`EXTPTRSXP`), `weakref` (`WEAKREFSXP`), 
    `bytecode` (`BCODESXP`), `promise` (`PROMSXP`), `...` (`DOTSXP`), and 
    `any` (`ANYSXP`).

\indexc{mode()}
You may have heard of `mode()` and `storage.mode()`. Do not use these functions: they exist only to provide type names that are compatible with S. 

### Numeric type {#numeric-type}
\index{numeric vectors}
\index{vectors!numeric|see {numeric vectors}}

Be careful when talking about the numeric type, because R uses "numeric" to mean three slightly different things:

1.  In some places numeric is used as an alias for the double type. For 
    example `as.numeric()` is identical to `as.double()`, and `numeric()` is
    identical to `double()`.
    
    (R also occasionally uses real instead of double; `NA_real_` is the one 
    place that you're likely to encounter this in practice.)
    
1.  In the S3 and S4 systems, numeric is used as a shorthand for either 
    integer or double type, and is used when picking methods:

    
    ```r
    sloop::s3_class(1)
    #> [1] "double"  "numeric"
    sloop::s3_class(1L)
    #> [1] "integer" "numeric"
    ```

1.  `is.numeric()` tests for objects that _behave_ like numbers. For example, 
    factors have type "integer" but don't behave like numbers (i.e. it doesn't
    make sense to take the mean of factor).

    
    ```r
    typeof(factor("x"))
    #> [1] "integer"
    is.numeric(factor("x"))
    #> [1] FALSE
    ```

In this book, I consistently use numeric to mean an object of type integer or double.

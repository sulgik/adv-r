# Evaluation



## Introduction

The user-facing inverse of quotation is unquotation: it gives the _user_ the ability to selectively evaluate parts of an otherwise quoted argument. The developer-facing complement of quotation is evaluation: this gives the _developer_ the ability to evaluate quoted expressions in custom environments to achieve specific goals.

This chapter begins with a discussion of evaluation in its purest form. You'll learn how `eval()` evaluates an expression in an environment, and then how it can be used to implement a number of important base R functions. Once you have the basics under your belt, you'll learn extensions to evaluation that are needed for robustness. There are two big new ideas:

*   The quosure: a data structure that captures an expression along with its
    associated environment, as found in function arguments.

*   The data mask, which makes it easier to evaluate an expression in the
    context of a data frame. This introduces potential evaluation ambiguity
    which we'll then resolve with data pronouns.

Together, quasiquotation, quosures, and data masks form what we call __tidy evaluation__, or tidy eval for short. Tidy eval provides a principled approach to non-standard evaluation that makes it possible to use such functions both interactively and embedded with other functions. Tidy evaluation is the most important practical implication of all this theory so we'll spend a little time exploring the implications. The chapter finishes off with a discussion of the closest related approaches in base R, and how you can program around their drawbacks.

### Outline {-}

* Section \@ref(eval) discusses the basics of evaluation using `eval()`,
  and shows how you can use it to implement key functions like `local()`
  and `source()`.

* Section \@ref(quosures) introduces a new data structure, the quosure, which
  combines an expression with an environment. You'll learn how to capture
  quosures from promises, and evaluate them using `rlang::eval_tidy()`.

* Section \@ref(data-masks) extends evaluation with the data mask, which
  makes it trivial to intermingle symbols bound in an environment with
  variables found in a data frame.

* Section \@ref(tidy-evaluation) shows how to use tidy evaluation in practice,
  focussing on the common pattern of quoting and unquoting, and how to
  handle ambiguity with pronouns.

* Section \@ref(base-evaluation) circles back to evaluation in base R,
  discusses some of the downsides, and shows how to use quasiquotation and
  evaluation to wrap functions that use NSE.

### Prerequisites {-}

You'll need to be familiar with the content of Chapter \@ref(expressions) and Chapter \@ref(quasiquotation), as well as the environment data structure (Section \@ref(env-basics)) and the caller environment (Section \@ref(call-stack)).

We'll continue to use [rlang](https://rlang.r-lib.org) and [purrr](https://purrr.tidyverse.org).


```r
library(rlang)
library(purrr)
```

## Evaluation basics {#eval}
\index{evaluation!basics}
\indexc{eval\_bare()}

Here we'll explore the details of `eval()` which we briefly mentioned in the last chapter. It has two key arguments: `expr` and `envir`. The first argument, `expr`, is the object to evaluate, typically a symbol or expression[^non-expr]. None of the evaluation functions quote their inputs, so you'll usually use them with `expr()` or similar:

[^non-expr]: All other objects yield themselves when evaluated; i.e. `eval(x)` yields `x`, except when `x` is a symbol or expression.


```r
x <- 10
eval(expr(x))
#> [1] 10

y <- 2
eval(expr(x + y))
#> [1] 12
```

The second argument, `env`, gives the environment in which the expression should be evaluated, i.e. where to look for the values of `x`, `y`, and `+`. By default, this is the current environment, i.e. the calling environment of `eval()`, but you can override it if you want:


```r
eval(expr(x + y), env(x = 1000))
#> [1] 1002
```

The first argument is evaluated, not quoted, which can lead to confusing results once if you use a custom environment and forget to manually quote:


```r
eval(print(x + 1), env(x = 1000))
#> [1] 11
#> [1] 11

eval(expr(print(x + 1)), env(x = 1000))
#> [1] 1001
```

Now that you've seen the basics, let's explore some applications. We'll focus primarily on base R functions that you might have used before, reimplementing the underlying principles using rlang.

### Application: `local()`
\indexc{local()}

Sometimes you want to perform a chunk of calculation that creates some intermediate variables. The intermediate variables have no long-term use and could be quite large, so you'd rather not keep them around. One approach is to clean up after yourself using `rm()`; another is to wrap the code in a function and just call it once. A more elegant approach is to use `local()`:


```r
# Clean up variables created earlier
rm(x, y)

foo <- local({
  x <- 10
  y <- 200
  x + y
})

foo
#> [1] 210
x
#> Error in eval(expr, envir, enclos): object 'x' not found
y
#> Error in eval(expr, envir, enclos): object 'y' not found
```

The essence of `local()` is quite simple and re-implemented below. We capture the input expression, and create a new environment in which to evaluate it. This is a new environment (so assignment doesn't affect the existing environment) with the caller environment as parent (so that `expr` can still access variables in that environment). This effectively emulates running `expr` as if it was inside a function (i.e. it's lexically scoped, Section \@ref(lexical-scoping)).


```r
local2 <- function(expr) {
  env <- env(caller_env())
  eval(enexpr(expr), env)
}

foo <- local2({
  x <- 10
  y <- 200
  x + y
})

foo
#> [1] 210
x
#> Error in eval(expr, envir, enclos): object 'x' not found
y
#> Error in eval(expr, envir, enclos): object 'y' not found
```

Understanding how `base::local()` works is harder, as it uses `eval()` and `substitute()` together in rather complicated ways. Figuring out exactly what's going on is good practice if you really want to understand the subtleties of `substitute()` and the base `eval()` functions, so they are included in the exercises below.

### Application: `source()`
\indexc{source()}

We can create a simple version of `source()` by combining `eval()` with `parse_expr()` from Section \@ref(parsing). We read in the file from disk, use `parse_expr()` to parse the string into a list of expressions, and then use `eval()` to evaluate each element in turn. This version evaluates the code in the caller environment, and invisibly returns the result of the last expression in the file just like `base::source()`.


```r
source2 <- function(path, env = caller_env()) {
  file <- paste(readLines(path, warn = FALSE), collapse = "\n")
  exprs <- parse_exprs(file)

  res <- NULL
  for (i in seq_along(exprs)) {
    res <- eval(exprs[[i]], env)
  }

  invisible(res)
}
```

The real `source()` is considerably more complicated because it can `echo` input and output, and has many other settings that control its behaviour.


::: sidebar
**Expression vectors**
\index{expression vectors}

`base::eval()` has special behaviour for expression _vectors_, evaluating each component in turn. This makes for a very compact implementation of `source2()` because `base::parse()` also returns an expression object:


```r
source3 <- function(file, env = parent.frame()) {
  lines <- parse(file)
  res <- eval(lines, envir = env)
  invisible(res)
}
```

While `source3()` is considerably more concise than `source2()`, this is the only advantage to expression vectors. Overall I don't believe this benefit outweighs the cost of introducing a new data structure, and hence this book avoids the use of expression vectors.
:::


### Gotcha: `function()`
\index{evaluation!functions}
\indexc{srcref}

There's one small gotcha that you should be aware of if you're using `eval()` and `expr()` to generate functions:


```r
x <- 10
y <- 20
f <- eval(expr(function(x, y) !!x + !!y))
f
#> function(x, y) !!x + !!y
```

This function doesn't look like it will work, but it does:


```r
f()
#> [1] 30
```

This is because, if available, functions print their `srcref` attribute (Section \@ref(fun-components)), and because `srcref` is a base R feature it's unaware of quasiquotation. 

To work around this problem, either use `new_function()` (Section \@ref(new-function)) or remove the `srcref` attribute:


```r
attr(f, "srcref") <- NULL
f
#> function (x, y) 
#> 10 + 20
```

### Exercises

1.  Carefully read the documentation for `source()`. What environment does it
    use by default? What if you supply `local = TRUE`? How do you provide
    a custom environment?

1.  Predict the results of the following lines of code:

    
    ```r
    eval(expr(eval(expr(eval(expr(2 + 2))))))
    eval(eval(expr(eval(expr(eval(expr(2 + 2)))))))
    expr(eval(expr(eval(expr(eval(expr(2 + 2)))))))
    ```

1.  Fill in the function bodies below to re-implement `get()` using `sym()` 
    and `eval()`, and`assign()` using `sym()`, `expr()`, and `eval()`. Don't 
    worry about the multiple ways of choosing an environment that `get()` and
    `assign()` support; assume that the user supplies it explicitly.

    
    ```r
    # name is a string
    get2 <- function(name, env) {}
    assign2 <- function(name, value, env) {}
    ```

1.  Modify `source2()` so it returns the result of _every_ expression,
    not just the last one. Can you eliminate the for loop?

1.  We can make `base::local()` slightly easier to understand by spreading
    out over multiple lines:

    
    ```r
    local3 <- function(expr, envir = new.env()) {
      call <- substitute(eval(quote(expr), envir))
      eval(call, envir = parent.frame())
    }
    ```

    Explain how `local()` works in words. (Hint: you might want to `print(call)`
    to help understand what `substitute()` is doing, and read the documentation
    to remind yourself what environment `new.env()` will inherit from.)

## Quosures
\index{quosures}

Almost every use of `eval()` involves both an expression and environment. This coupling is so important that we need a data structure that can hold both pieces. Base R does not have such a structure[^formula] so rlang fills the gap with the __quosure__, an object that contains an expression and an environment. The name is a portmanteau of quoting and closure, because a quosure both quotes the expression and encloses the environment. Quosures reify the internal promise object (Section \@ref(promises)) into something that you can program with.

[^formula]: Technically a formula combines an expression and environment, but formulas are tightly coupled to modelling so a new data structure makes sense.

In this section, you'll learn how to create and manipulate quosures, and a little about how they are implemented.

### Creating
\index{quosures!creating}

There are three ways to create quosures:

*   Use `enquo()` and `enquos()` to capture user-supplied expressions.
    The vast majority of quosures should be created this way.

    
    ```r
    foo <- function(x) enquo(x)
    foo(a + b)
    #> <quosure>
    #> expr: ^a + b
    #> env:  global
    ```
    \indexc{enquo()}

*   `quo()` and `quos()` exist to match to `expr()` and `exprs()`, but
    they are included only for the sake of completeness and are needed very
    rarely. If you find yourself using them, think carefully if `expr()` and
    careful unquoting can eliminate the need to capture the environment.

    
    ```r
    quo(x + y + z)
    #> <quosure>
    #> expr: ^x + y + z
    #> env:  global
    ```
    \index{quosures!quo()@\texttt{quo()}}

*   `new_quosure()` create a quosure from its components: an expression and
    an environment. This is rarely needed in practice, but is useful for
    learning, so is used a lot in this chapter.

    
    ```r
    new_quosure(expr(x + y), env(x = 1, y = 10))
    #> <quosure>
    #> expr: ^x + y
    #> env:  0x556277fd8d80
    ```

### Evaluating
\index{evaluation!tidy}
\index{quosures!evaluating}
\indexc{eval\_tidy()}

Quosures are paired with a new evaluation function `eval_tidy()` that takes a single quosure instead of an expression-environment pair. It is straightforward to use:


```r
q1 <- new_quosure(expr(x + y), env(x = 1, y = 10))
eval_tidy(q1)
#> [1] 11
```

For this simple case, `eval_tidy(q1)` is basically a shortcut for `eval(get_expr(q1), get_env(q1))`. However, it has two important features that you'll learn about later in the chapter: it supports nested quosures (Section \@ref(nested-quosures)) and pronouns (Section \@ref(pronouns)).

### Dots {#quosure-dots}
\indexc{...}

Quosures are typically just a convenience: they make code cleaner because you only have one object to pass around, instead of two. They are, however, essential when it comes to working with `...` because it's possible for each argument passed to `...` to be associated with a different environment. In the following example note that both quosures have the same expression, `x`, but a different environment:


```r
f <- function(...) {
  x <- 1
  g(..., f = x)
}
g <- function(...) {
  enquos(...)
}

x <- 0
qs <- f(global = x)
qs
#> <list_of<quosure>>
#> 
#> $global
#> <quosure>
#> expr: ^x
#> env:  global
#> 
#> $f
#> <quosure>
#> expr: ^x
#> env:  0x556277e4eb20
```

That means that when you evaluate them, you get the correct results:


```r
map_dbl(qs, eval_tidy)
#> global      f 
#>      0      1
```

Correctly evaluating the elements of `...` was one of the original motivations for the development of quosures.

### Under the hood {#quosure-impl}
\index{quosures!internals}
\index{formulas}

Quosures were inspired by R's formulas, because formulas capture an expression and an environment:


```r
f <- ~runif(3)
str(f)
#> Class 'formula'  language ~runif(3)
#>   ..- attr(*, ".Environment")=<environment: R_GlobalEnv>
```

An early version of tidy evaluation used formulas instead of quosures, as an attractive feature of `~` is that it provides quoting with a single keystroke. Unfortunately, however, there is no clean way to make `~` a quasiquoting function.

Quosures are a subclass of formulas:


```r
q4 <- new_quosure(expr(x + y + z))
class(q4)
#> [1] "quosure" "formula"
```

which means that under the hood, quosures, like formulas, are call objects:


```r
is_call(q4)
#> [1] TRUE

q4[[1]]
#> Warning: Subsetting quosures with `[[` is deprecated as of rlang 0.4.0
#> Please use `quo_get_expr()` instead.
#> This warning is displayed once per session.
#> `~`
q4[[2]]
#> x + y + z
```

with an attribute that stores the environment:


```r
attr(q4, ".Environment")
#> <environment: R_GlobalEnv>
```

If you need to extract the expression or environment, don't rely on these implementation details. Instead use `get_expr()` and `get_env()`:


```r
get_expr(q4)
#> x + y + z
get_env(q4)
#> <environment: R_GlobalEnv>
```

### Nested quosures
\index{quosures!nested}

It's possible to use quasiquotation to embed a quosure in an expression. This is an advanced tool, and most of the time you don't need to think about it because it just works, but I talk about it here so you can spot nested quosures in the wild and not be confused. Take this example, which inlines two quosures into an expression:


```r
q2 <- new_quosure(expr(x), env(x = 1))
q3 <- new_quosure(expr(x), env(x = 10))

x <- expr(!!q2 + !!q3)
```

It evaluates correctly with `eval_tidy()`:


```r
eval_tidy(x)
#> [1] 11
```

However, if you print it, you only see the `x`s, with their formula heritage leaking through:


```r
x
#> (~x) + ~x
```

You can get a better display with `rlang::expr_print()` (Section \@ref(non-standard-ast)):


```r
expr_print(x)
#> (^x) + (^x)
```

When you use `expr_print()` in the console, quosures are coloured according to their environment, making it easier to spot when symbols are bound to different variables.

### Exercises

1.  Predict what each of the following quosures will return if
    evaluated.

    
    ```r
    q1 <- new_quosure(expr(x), env(x = 1))
    q1
    #> <quosure>
    #> expr: ^x
    #> env:  0x55627a6be3e8
    
    q2 <- new_quosure(expr(x + !!q1), env(x = 10))
    q2
    #> <quosure>
    #> expr: ^x + (^x)
    #> env:  0x55627a7e3d88
    
    q3 <- new_quosure(expr(x + !!q2), env(x = 100))
    q3
    #> <quosure>
    #> expr: ^x + (^x + (^x))
    #> env:  0x55627a9b55c0
    ```

1.  Write an `enenv()` function that captures the environment associated
    with an argument. (Hint: this should only require two function calls.)

## Data masks
\index{data masks}

In this section, you'll learn about the __data mask__, a data frame where the evaluated code will look first for variable definitions. The data mask is the key idea that powers base functions like `with()`, `subset()` and `transform()`, and is used throughout the tidyverse in packages like dplyr and ggplot2.

### Basics
\indexc{eval\_tidy()}

The data mask allows you to mingle variables from an environment and a data frame in a single expression. You supply the data mask as the second argument to `eval_tidy()`:


```r
q1 <- new_quosure(expr(x * y), env(x = 100))
df <- data.frame(y = 1:10)

eval_tidy(q1, df)
#>  [1]  100  200  300  400  500  600  700  800  900 1000
```

This code is a little hard to follow because there's so much syntax as we're creating every object from scratch. It's easier to see what's going on if we make a little wrapper. I call this `with2()` because it's equivalent to `base::with()`.
\indexc{with()}


```r
with2 <- function(data, expr) {
  expr <- enquo(expr)
  eval_tidy(expr, data)
}
```

We can now rewrite the code above as below:


```r
x <- 100
with2(df, x * y)
#>  [1]  100  200  300  400  500  600  700  800  900 1000
```

`base::eval()` has similar functionality, although it doesn't call it a data mask. Instead you can supply a data frame to the second argument and an environment to the third. That gives the following implementation of `with()`:


```r
with3 <- function(data, expr) {
  expr <- substitute(expr)
  eval(expr, data, caller_env())
}
```

### Pronouns
\index{pronouns}
\index{.Data@\texttt{.data}}
\index{.Env@\texttt{.env}}

Using a data mask introduces ambiguity. For example, in the following code you can't know whether `x` will come from the data mask or the environment, unless you know what variables are found in `df`.


```r
with2(df, x)
```

That makes code harder to reason about (because you need to know more context), which can introduce bugs. To resolve that issue, the data mask provides two pronouns: `.data` and `.env`.

* `.data$x` always refers to `x` in the data mask.
* `.env$x`  always refers to `x` in the environment.


```r
x <- 1
df <- data.frame(x = 2)

with2(df, .data$x)
#> [1] 2
with2(df, .env$x)
#> [1] 1
```

You can also subset `.data` and `.env` using `[[`, e.g. `.data[["x"]]`. Otherwise the pronouns are special objects and you shouldn't expect them to behave like data frames or environments. In particular, they throw an error if the object isn't found:


```r
with2(df, .data$y)
#> Error: Column `y` not found in `.data`
```

### Application: `subset()` {#subset}
\indexc{subset()}
\indexc{filter()}

We'll explore tidy evaluation in the context of `base::subset()`, because it's a simple yet powerful function that makes a common data manipulation challenge easier. If you haven't used it before, `subset()`, like `dplyr::filter()`, provides a convenient way of selecting rows of a data frame. You give it some data, along with an expression that is evaluated in the context of that data. This considerably reduces the number of times you need to type the name of the data frame:


```r
sample_df <- data.frame(a = 1:5, b = 5:1, c = c(5, 3, 1, 4, 1))

# Shorthand for sample_df[sample_df$a >= 4, ]
subset(sample_df, a >= 4)
#>   a b c
#> 4 4 2 4
#> 5 5 1 1

# Shorthand for sample_df[sample_df$b == sample_df$c, ]
subset(sample_df, b == c)
#>   a b c
#> 1 1 5 5
#> 5 5 1 1
```

The core of our version of `subset()`, `subset2()`, is quite simple. It takes two arguments: a data frame, `data`, and an expression, `rows`. We evaluate `rows` using `df` as a data mask, then use the results to subset the data frame with `[`. I've included a very simple check to ensure the result is a logical vector; real code would do more to create an informative error.


```r
subset2 <- function(data, rows) {
  rows <- enquo(rows)
  rows_val <- eval_tidy(rows, data)
  stopifnot(is.logical(rows_val))

  data[rows_val, , drop = FALSE]
}

subset2(sample_df, b == c)
#>   a b c
#> 1 1 5 5
#> 5 5 1 1
```

### Application: transform
\indexc{transform()}

A more complicated situation is `base::transform()` which allows you to add new variables to a data frame, evaluating their expressions in the context of the existing variables:


```r
df <- data.frame(x = c(2, 3, 1), y = runif(3))
transform(df, x = -x, y2 = 2 * y)
#>    x      y    y2
#> 1 -2 0.0808 0.162
#> 2 -3 0.8343 1.669
#> 3 -1 0.6008 1.202
```

Again, our own `transform2()` requires little code. We capture the unevaluated `...`  with `enquos(...)`, and then evaluate each expression using a for loop. Real code would do more error checking to ensure that each input is named and evaluates to a vector the same length as `data`.


```r
transform2 <- function(.data, ...) {
  dots <- enquos(...)

  for (i in seq_along(dots)) {
    name <- names(dots)[[i]]
    dot <- dots[[i]]

    .data[[name]] <- eval_tidy(dot, .data)
  }

  .data
}

transform2(df, x2 = x * 2, y = -y)
#>   x       y x2
#> 1 2 -0.0808  4
#> 2 3 -0.8343  6
#> 3 1 -0.6008  2
```

NB: I named the first argument `.data` to avoid problems if users tried to create a variable called `data`. They will still have problems if they attempt to create a variable called `.data`, but this is much less likely. This is the same reasoning that leads to the `.x` and `.f` arguments to `map()` (Section \@ref(argument-names)).

### Application: `select()` {#select}
\indexc{select()}

A data mask will typically be a data frame, but it's sometimes useful to provide a list filled with more exotic contents. This is basically how the `select` argument in `base::subset()` works. It allows you to refer to variables as if they were numbers:


```r
df <- data.frame(a = 1, b = 2, c = 3, d = 4, e = 5)
subset(df, select = b:d)
#>   b c d
#> 1 2 3 4
```

The key idea is to create a named list where each component gives the position of the corresponding variable:


```r
vars <- as.list(set_names(seq_along(df), names(df)))
str(vars)
#> List of 5
#>  $ a: int 1
#>  $ b: int 2
#>  $ c: int 3
#>  $ d: int 4
#>  $ e: int 5
```

Then implementation is again only a few lines of code:


```r
select2 <- function(data, ...) {
  dots <- enquos(...)

  vars <- as.list(set_names(seq_along(data), names(data)))
  cols <- unlist(map(dots, eval_tidy, vars))

  data[, cols, drop = FALSE]
}
select2(df, b:d)
#>   b c d
#> 1 2 3 4
```

`dplyr::select()` takes this idea and runs with it, providing a number of helpers that allow you to select variables based on their names (e.g. `starts_with("x")` or `ends_with("_a"`)).

### Exercises

1.  Why did I use a for loop in `transform2()` instead of `map()`? 
    Consider `transform2(df, x = x * 2, x = x * 2)`.

1.  Here's an alternative implementation of `subset2()`:

    
    ```r
    subset3 <- function(data, rows) {
      rows <- enquo(rows)
      eval_tidy(expr(data[!!rows, , drop = FALSE]), data = data)
    }
    
    df <- data.frame(x = 1:3)
    subset3(df, x == 1)
    ```

    Compare and contrast `subset3()` to `subset2()`. What are its advantages
    and disadvantages?

1.  The following function implements the basics of `dplyr::arrange()`.
    Annotate each line with a comment explaining what it does. Can you
    explain why `!!.na.last` is strictly correct, but omitting the `!!`
    is unlikely to cause problems?

    
    ```r
    arrange2 <- function(.df, ..., .na.last = TRUE) {
      args <- enquos(...)
    
      order_call <- expr(order(!!!args, na.last = !!.na.last))
    
      ord <- eval_tidy(order_call, .df)
      stopifnot(length(ord) == nrow(.df))
    
      .df[ord, , drop = FALSE]
    }
    ```

## Using tidy evaluation {#tidy-evaluation}

While it's important to understand how `eval_tidy()` works, most of the time you won't call it directly. Instead, you'll usually use it indirectly by calling a function that uses `eval_tidy()`. This section will give a few practical examples of wrapping functions that use tidy evaluation.

### Quoting and unquoting
\index{quoting!in practice}
\index{unquoting!in practice}
\index{bootstrapping}




Imagine we have written a function that resamples a dataset:


```r
resample <- function(df, n) {
  idx <- sample(nrow(df), n, replace = TRUE)
  df[idx, , drop = FALSE]
}
```

We want to create a new function that allows us to resample and subset in a single step. Our naive approach doesn't work:


```r
subsample <- function(df, cond, n = nrow(df)) {
  df <- subset2(df, cond)
  resample(df, n)
}

df <- data.frame(x = c(1, 1, 1, 2, 2), y = 1:5)
subsample(df, x == 1)
#> Error in eval_tidy(rows, data): object 'x' not found
```

`subsample()` doesn't quote any arguments so `cond` is evaluated normally (not in a data mask), and we get an error when it tries to find a binding for  `x`. To fix this problem we need to quote `cond`, and then unquote it when we pass it on ot `subset2()`:


```r
subsample <- function(df, cond, n = nrow(df)) {
  cond <- enquo(cond)

  df <- subset2(df, !!cond)
  resample(df, n)
}

subsample(df, x == 1)
#>   x y
#> 3 1 3
#> 1 1 1
#> 2 1 2
```

This is a very common pattern; whenever you call a quoting function with arguments from the user, you need to quote them and then unquote.

<!-- GVW: I really, really want a diagram here to show the various objects in play at each step - it took me a long time to figure out why quote/unquote was needed, and I still have to go back and review it each time I run into it. -->

### Handling ambiguity
\index{pronouns}

In the case above, we needed to think about tidy evaluation because of quasiquotation. We also need to think about tidy evaluation even when the wrapper doesn't need to quote any arguments. Take this wrapper around `subset2()`:


```r
threshold_x <- function(df, val) {
  subset2(df, x >= val)
}
```

This function can silently return an incorrect result in two situations:

*   When `x` exists in the calling environment, but not in `df`:

    
    ```r
    x <- 10
    no_x <- data.frame(y = 1:3)
    threshold_x(no_x, 2)
    #>   y
    #> 1 1
    #> 2 2
    #> 3 3
    ```

*   When `val` exists in `df`:

    
    ```r
    has_val <- data.frame(x = 1:3, val = 9:11)
    threshold_x(has_val, 2)
    #> [1] x   val
    #> <0 rows> (or 0-length row.names)
    ```

These failure modes arise because tidy evaluation is ambiguous: each variable can be found in __either__ the data mask __or__ the environment. To make this function safe we need to remove the ambiguity using the `.data` and `.env` pronouns:


```r
threshold_x <- function(df, val) {
  subset2(df, .data$x >= .env$val)
}

x <- 10
threshold_x(no_x, 2)
#> Error: Column `x` not found in `.data`
threshold_x(has_val, 2)
#>   x val
#> 2 2  10
#> 3 3  11
```

Generally, whenever you use the `.env` pronoun, you can use unquoting instead:


```r
threshold_x <- function(df, val) {
  subset2(df, .data$x >= !!val)
}
```

There are subtle differences in when `val` is evaluated. If you unquote, `val` will be early evaluated by `enquo()`; if you use a pronoun, `val` will be lazily evaluated by `eval_tidy()`. These differences are usually unimportant, so pick the form that looks most natural.

### Quoting and ambiguity

To finish our discussion let's consider the case where we have both quoting and potential ambiguity. I'll generalise `threshold_x()` slightly so that the user can pick the variable used for thresholding. Here I used `.data[[var]]` because it makes the code a little simpler; in the exercises you'll have a chance to explore how you might use `$` instead.


```r
threshold_var <- function(df, var, val) {
  var <- as_string(ensym(var))
  subset2(df, .data[[var]] >= !!val)
}

df <- data.frame(x = 1:10)
threshold_var(df, x, 8)
#>     x
#> 8   8
#> 9   9
#> 10 10
```

It is not always the responsibility of the function author to avoid ambiguity. Imagine we generalise further to allow thresholding based on any expression:


```r
threshold_expr <- function(df, expr, val) {
  expr <- enquo(expr)
  subset2(df, !!expr >= !!val)
}
```

It's not possible to evaluate `expr` only in the data mask, because the data mask doesn't include any functions like `+` or `==`. Here, it's the user's responsibility to avoid ambiguity. As a general rule of thumb, as a function author it's your responsibility to avoid ambiguity with any expressions that you create; it's the user's responsibility to avoid ambiguity in expressions that they create.

### Exercises

1.  I've included an alternative implementation of `threshold_var()` below.
    What makes it different to the approach I used above? What makes it harder?

    
    ```r
    threshold_var <- function(df, var, val) {
      var <- ensym(var)
      subset2(df, `$`(.data, !!var) >= !!val)
    }
    ```

## Base evaluation
\index{evaluation!base R}

Now that you understand tidy evaluation, it's time to come back to the alternative approaches taken by base R. Here I'll explore the two most common uses in base R:

* `substitute()` and evaluation in the caller environment, as used by
  `subset()`. I'll use this technique to demonstrate why this technique is not
  programming friendly, as warned about in the `subset()` documentation.

* `match.call()`, call manipulation, and evaluation in the caller environment,
  as used by `write.csv()` and `lm()`. I'll use this technique to demonstrate how
  quasiquotation and (regular) evaluation can help you write wrappers around 
  such functions.

These two approaches are common forms of non-standard evaluation (NSE).

### `substitute()`
\indexc{substitute()}

The most common form of NSE in base R is `substitute()` + `eval()`.  The following code shows how you might write the core of `subset()` in this style using `substitute()` and `eval()` rather than `enquo()` and `eval_tidy()`. I repeat the code introduced in Section \@ref(subset) so you can compare easily. The main difference is the evaluation environment: in `subset_base()` the argument is evaluated in the caller environment, while in `subset_tidy()`, it's evaluated in the environment where it was defined.


```r
subset_base <- function(data, rows) {
  rows <- substitute(rows)
  rows_val <- eval(rows, data, caller_env())
  stopifnot(is.logical(rows_val))

  data[rows_val, , drop = FALSE]
}

subset_tidy <- function(data, rows) {
  rows <- enquo(rows)
  rows_val <- eval_tidy(rows, data)
  stopifnot(is.logical(rows_val))

  data[rows_val, , drop = FALSE]
}
```

#### Programming with `subset()`
\indexc{subset()}

The documentation of `subset()` includes the following warning:

> This is a convenience function intended for use interactively. For
> programming it is better to use the standard subsetting functions like `[`,
> and in particular the non-standard evaluation of argument `subset` can have
> unanticipated consequences.

There are three main problems:

*   `base::subset()` always evaluates `rows` in the calling environment, but
    if `...` has been used, then the expression might need to be evaluated
    elsewhere:

    
    ```r
    f1 <- function(df, ...) {
      xval <- 3
      subset_base(df, ...)
    }
    
    my_df <- data.frame(x = 1:3, y = 3:1)
    xval <- 1
    f1(my_df, x == xval)
    #>   x y
    #> 3 3 1
    ```

    This may seems like an esoteric concern, but it means that `subset_base()`
    cannot reliably work with functionals like `map()` or `lapply()`:

    
    ```r
    local({
      zzz <- 2
      dfs <- list(data.frame(x = 1:3), data.frame(x = 4:6))
      lapply(dfs, subset_base, x == zzz)
    })
    #> Error in eval(rows, data, caller_env()): object 'zzz' not found
    ```

*   Calling `subset()` from another function requires some care: you have
    to use `substitute()` to capture a call to `subset()` complete expression,
    and then evaluate. I think this code is hard to understand because 
    `substitute()` doesn't use a syntactic marker for unquoting. Here I print 
    the generated call to make it a little easier to see what's happening.

    
    ```r
    f2 <- function(df1, expr) {
      call <- substitute(subset_base(df1, expr))
      expr_print(call)
      eval(call, caller_env())
    }
    
    my_df <- data.frame(x = 1:3, y = 3:1)
    f2(my_df, x == 1)
    #> subset_base(my_df, x == 1)
    #>   x y
    #> 1 1 3
    ```

*   `eval()` doesn't provide any pronouns so there's no way to require part of
    the expression to come from the data. As far as I can tell, there's no
    way to make the following function safe except by manually checking for the
    presence of `z` variable in `df`.

    
    ```r
    f3 <- function(df) {
      call <- substitute(subset_base(df, z > 0))
      expr_print(call)
      eval(call, caller_env())
    }
    
    my_df <- data.frame(x = 1:3, y = 3:1)
    z <- -1
    f3(my_df)
    #> subset_base(my_df, z > 0)
    #> [1] x y
    #> <0 rows> (or 0-length row.names)
    ```

#### What about `[`?

Given that tidy evaluation is quite complex, why not simply use `[` as `?subset` recommends? Primarily, it seems unappealing to me to have functions that can only be used interactively, and never inside another function. 

Additionally, even the simple `subset()` function provides two useful features compared to `[`:

* It sets `drop = FALSE` by default, so it's guaranteed to return a data frame.

* It drops rows where the condition evaluates to `NA`.

That means `subset(df, x == y)` is not equivalent to `df[x == y,]` as you might expect. Instead, it is equivalent to `df[x == y & !is.na(x == y), , drop = FALSE]`: that's a lot more typing! Real-life alternatives to `subset()`, like `dplyr::filter()`, do even more. For example, `dplyr::filter()` can translate R expressions to SQL so that they can be executed in a database. This makes programming with `filter()` relatively more important.

### `match.call()`
\indexc{match.call()}

Another common form of NSE is to capture the complete call with `match.call()`, modify it, and evaluate the result. `match.call()` is similar to `substitute()`, but instead of capturing a single argument, it captures the complete call. It doesn't have an equivalent in rlang. 


```r
g <- function(x, y, z) {
  match.call()
}
g(1, 2, z = 3)
#> g(x = 1, y = 2, z = 3)
```

One prominent user of `match.call()` is `write.csv()`, which basically works by transforming the call into a call to `write.table()` with the appropriate arguments set. The following code shows the heart of `write.csv()`:


```r
write.csv <- function(...) {
  call <- match.call(write.table, expand.dots = TRUE)

  call[[1]] <- quote(write.table)
  call$sep <- ","
  call$dec <- "."

  eval(call, parent.frame())
}
```

I don't think this technique is a good idea because you can achieve the same result without NSE:


```r
write.csv <- function(...) {
  write.table(..., sep = ",", dec = ".")
}
```

Nevertheless, it's important to understand this technique because it's commonly used in modelling functions. These functions also prominently print the captured call, which poses some special challenges, as you'll see next.

#### Wrapping modelling functions
\indexc{lm()}

To begin, consider the simplest possible wrapper around `lm()`:


```r
lm2 <- function(formula, data) {
  lm(formula, data)
}
```

This wrapper works, but is suboptimal because `lm()` captures its call and displays it when printing.


```r
lm2(mpg ~ disp, mtcars)
#> 
#> Call:
#> lm(formula = formula, data = data)
#> 
#> Coefficients:
#> (Intercept)         disp  
#>     29.5999      -0.0412
```

Fixing this is important because this call is the chief way that you see the model specification when printing the model. To overcome this problem, we need to capture the arguments, create the call to `lm()` using unquoting, then evaluate that call. To make it easier to see what's going on, I'll also print the expression we generate. This will become more useful as the calls get more complicated.


```r
lm3 <- function(formula, data, env = caller_env()) {
  formula <- enexpr(formula)
  data <- enexpr(data)

  lm_call <- expr(lm(!!formula, data = !!data))
  expr_print(lm_call)
  eval(lm_call, env)
}

lm3(mpg ~ disp, mtcars)
#> lm(mpg ~ disp, data = mtcars)
#> 
#> Call:
#> lm(formula = mpg ~ disp, data = mtcars)
#> 
#> Coefficients:
#> (Intercept)         disp  
#>     29.5999      -0.0412
```

There are three pieces that you'll use whenever wrapping a base NSE function in this way:

* You capture the unevaluated arguments using `enexpr()`, and capture the caller
  environment using `caller_env()`. 

* You generate a new expression using `expr()` and unquoting.

* You evaluate that expression in the caller environment. You have to accept 
  that the function will not work correctly if the arguments are not defined 
  in the caller environment. Providing the `env` argument at least provides
  a hook that experts can use if the default environment isn't correct.

The use of `enexpr()` has a nice side-effect: we can use unquoting to generate formulas dynamically:


```r
resp <- expr(mpg)
disp1 <- expr(vs)
disp2 <- expr(wt)
lm3(!!resp ~ !!disp1 + !!disp2, mtcars)
#> lm(mpg ~ vs + wt, data = mtcars)
#> 
#> Call:
#> lm(formula = mpg ~ vs + wt, data = mtcars)
#> 
#> Coefficients:
#> (Intercept)           vs           wt  
#>       33.00         3.15        -4.44
```

#### Evaluation environment

What if you want to mingle objects supplied by the user with objects that you create in the function?  For example, imagine you want to make an auto-resampling version of `lm()`. You might write it like this:


```r
resample_lm0 <- function(formula, data, env = caller_env()) {
  formula <- enexpr(formula)
  resample_data <- resample(data, n = nrow(data))

  lm_call <- expr(lm(!!formula, data = resample_data))
  expr_print(lm_call)
  eval(lm_call, env)
}

df <- data.frame(x = 1:10, y = 5 + 3 * (1:10) + round(rnorm(10), 2))
resample_lm0(y ~ x, data = df)
#> lm(y ~ x, data = resample_data)
#> Error in is.data.frame(data): object 'resample_data' not found
```

Why doesn't this code work? We're evaluating `lm_call` in the caller environment, but `resample_data` exists in the execution environment. We could instead evaluate in the execution environment of `resample_lm0()`, but there's no guarantee that `formula` could be evaluated in that environment.

There are two basic ways to overcome this challenge:

1.  Unquote the data frame into the call. This means that no lookup has
    to occur, but has all the problems of inlining expressions (Section
    \@ref(non-standard-ast)). For modelling functions this means that the 
    captured call is suboptimal:

    
    ```r
    resample_lm1 <- function(formula, data, env = caller_env()) {
      formula <- enexpr(formula)
      resample_data <- resample(data, n = nrow(data))
    
      lm_call <- expr(lm(!!formula, data = !!resample_data))
      expr_print(lm_call)
      eval(lm_call, env)
    }
    resample_lm1(y ~ x, data = df)$call
    #> lm(y ~ x, data = <data.frame>)
    #> lm(formula = y ~ x, data = list(x = c(3L, 7L, 4L, 4L, 
    #> 2L, 7L, 2L, 1L, 8L, 9L), y = c(13.21, 27.04, 18.63, 
    #> 18.63, 10.99, 27.04, 10.99, 7.83, 28.14, 32.72)))
    ```

1.  Alternatively you can create a new environment that inherits from the
    caller, and bind variables that you've created inside the
    function to that environment.

    
    ```r
    resample_lm2 <- function(formula, data, env = caller_env()) {
      formula <- enexpr(formula)
      resample_data <- resample(data, n = nrow(data))
    
      lm_env <- env(env, resample_data = resample_data)
      lm_call <- expr(lm(!!formula, data = resample_data))
      expr_print(lm_call)
      eval(lm_call, lm_env)
    }
    resample_lm2(y ~ x, data = df)
    #> lm(y ~ x, data = resample_data)
    #> 
    #> Call:
    #> lm(formula = y ~ x, data = resample_data)
    #> 
    #> Coefficients:
    #> (Intercept)            x  
    #>        4.42         3.11
    ```

    This is more work, but gives the cleanest specification.

### Exercises

1.  Why does this function fail?

    
    ```r
    lm3a <- function(formula, data) {
      formula <- enexpr(formula)
    
      lm_call <- expr(lm(!!formula, data = data))
      eval(lm_call, caller_env())
    }
    lm3a(mpg ~ disp, mtcars)$call
    #> Error in as.data.frame.default(data, optional = TRUE): 
    #> cannot coerce class ‘"function"’ to a data.frame
    ```

1.  When model building, typically the response and data are relatively
    constant while you rapidly experiment with different predictors. Write a
    small wrapper that allows you to reduce duplication in the code below.

    
    ```r
    lm(mpg ~ disp, data = mtcars)
    lm(mpg ~ I(1 / disp), data = mtcars)
    lm(mpg ~ disp * cyl, data = mtcars)
    ```

1.  Another way to write `resample_lm()` would be to include the
    resample expression (`data[sample(nrow(data), replace = TRUE), , drop = FALSE]`)
    in the data argument. Implement that approach. What are the advantages?
    What are the disadvantages?

# 함수 연산자 {#function-operators}



## 들어가기
\index{function operators}

이 장에서는 함수 연산자들에 대해 배울 것이다. __함수 연산자__ 는 하나 이상의 함수를 입력을 취하여 함수를 출력으로 반환하는 함수이다. 다음의 코드는 단순한 함수 연산자 `chatty()`를 보여준다. It wraps a function, making a new function that prints out its first argument. You might create a function like this because it gives you a window to see how functionals, like `map_int()`, work.


```r
chatty <- function(f) {
  force(f)
  
  function(x, ...) {
    res <- f(x, ...)
    cat("Processing ", x, "\n", sep = "")
    res
  }
}
f <- function(x) x ^ 2
s <- c(3, 2, 1)

purrr::map_dbl(s, chatty(f))
#> Processing 3
#> Processing 2
#> Processing 1
#> [1] 9 4 1
```

Function operators are closely related to function factories; indeed they're just a function factory that takes a function as input. Like factories, there's nothing you can't do without them, but they often allow you to factor out complexity in order to make your code more readable and reusable. 

Function operators are typically paired with functionals. If you're using a for-loop, there's rarely a reason to use a function operator, as it will make your code more complex for little gain.

If you're familiar with Python, decorators is just another name for function operators.

### Outline {-}

* Section \@ref(existing-fos) introduces you to two extremely useful existing 
  function operators, and shows you how to use them to solve real problems.
  
* Section \@ref(fo-case-study) works through a problem amenable to solution
  with function operators: downloading many web pages.

### Prerequisites {-}

Function operators are a type of function factory, so make sure you're familiar with at least Section \@ref(function-fundamentals) before you go on. 

We'll use [purrr](https://purrr.tidyverse.org) for a couple of functionals that you learned about in Chapter \@ref(functionals), and some function operators that you'll learn about below. We'll also use the [memoise package](https://memoise.r-lib.org) [@memoise] for the `memoise()` operator.


```r
library(purrr)
library(memoise)
```

<!--
### In other languages

Function operators are used extensively in FP languages like Haskell, and commonly in Lisp, Scheme and Clojure. They are also an important part of modern JavaScript programming, like in the [underscore.js](http://underscorejs.org/) library. They are particularly common in CoffeeScript because its syntax for anonymous functions is so concise. In stack-based languages like Forth and Factor, function operators are used almost exclusively because it's rare to refer to variables by name. Python's decorators are just function operators by a [different name](http://stackoverflow.com/questions/739654/). In Java, they are very rare because it's difficult to manipulate functions (although possible if you wrap them up in strategy-type objects). They are also rare in C++ because, while it's possible to create objects that work like functions ("functors") by overloading the `()` operator, modifying these objects with other functions is not a common programming technique. That said, C++ 11 includes partial application (`std::bind`) as part of the standard library.
-->

## Existing function operators {#existing-fos}

There are two very useful function operators that will both help you solve common recurring problems, and give you a sense for what function operators can do: `purrr::safely()` and `memoise::memoise()`.

### Capturing errors with `purrr::safely()` {#safely}
\indexc{safely()}
\index{errors!handling}

One advantage of for-loops is that if one of the iterations fails, you can still access all the results up to the failure:


```r
x <- list(
  c(0.512, 0.165, 0.717),
  c(0.064, 0.781, 0.427),
  c(0.890, 0.785, 0.495),
  "oops"
)

out <- rep(NA_real_, length(x))
for (i in seq_along(x)) {
  out[[i]] <- sum(x[[i]])
}
#> Error in sum(x[[i]]): invalid 'type' (character) of argument
out
#> [1] 1.39 1.27 2.17   NA
```

If you do the same thing with a functional, you get no output, making it hard to figure out where the problem lies:


```r
map_dbl(x, sum)
#> Error in .Primitive("sum")(..., na.rm = na.rm): invalid 'type' (character) of argument
```

`purrr::safely()` provides a tool to help with this problem. `safely()` is a function operator that transforms a function to turn errors into data. (You can learn the basic idea that makes it work in Section \@ref(try-success-failure).) Let's start by taking a look at it outside of `map_dbl()`:


```r
safe_sum <- safely(sum)
safe_sum
#> function (...) 
#> capture_error(.f(...), otherwise, quiet)
#> <bytecode: 0x55dd0415f418>
#> <environment: 0x55dd0415ef80>
```

Like all function operators, `safely()` takes a function and returns a wrapped function which we can call as usual:


```r
str(safe_sum(x[[1]]))
#> List of 2
#>  $ result: num 1.39
#>  $ error : NULL
str(safe_sum(x[[4]]))
#> List of 2
#>  $ result: NULL
#>  $ error :List of 2
#>   ..$ message: chr "invalid 'type' (character) of argument"
#>   ..$ call   : language .Primitive("sum")(..., na.rm = na.rm)
#>   ..- attr(*, "class")= chr [1:3] "simpleError" "error" "condition"
```

You can see that a function transformed by `safely()` always returns a list with two elements, `result` and `error`. If the function runs successfully, `error` is `NULL` and `result` contains the result; if the function fails, `result` is `NULL` and `error` contains the error.

Now lets use `safely()` with a functional:


```r
out <- map(x, safely(sum))
str(out)
#> List of 4
#>  $ :List of 2
#>   ..$ result: num 1.39
#>   ..$ error : NULL
#>  $ :List of 2
#>   ..$ result: num 1.27
#>   ..$ error : NULL
#>  $ :List of 2
#>   ..$ result: num 2.17
#>   ..$ error : NULL
#>  $ :List of 2
#>   ..$ result: NULL
#>   ..$ error :List of 2
#>   .. ..$ message: chr "invalid 'type' (character) of argument"
#>   .. ..$ call   : language .Primitive("sum")(..., na.rm = na.rm)
#>   .. ..- attr(*, "class")= chr [1:3] "simpleError" "error" "condition"
```

The output is in a slightly inconvenient form, since we have four lists, each of which is a list containing the `result` and the `error`. We can make the output easier to use by turning it "inside-out" with `purrr::transpose()`, so that we get a list of `result`s and a list of `error`s:


```r
out <- transpose(map(x, safely(sum)))
str(out)
#> List of 2
#>  $ result:List of 4
#>   ..$ : num 1.39
#>   ..$ : num 1.27
#>   ..$ : num 2.17
#>   ..$ : NULL
#>  $ error :List of 4
#>   ..$ : NULL
#>   ..$ : NULL
#>   ..$ : NULL
#>   ..$ :List of 2
#>   .. ..$ message: chr "invalid 'type' (character) of argument"
#>   .. ..$ call   : language .Primitive("sum")(..., na.rm = na.rm)
#>   .. ..- attr(*, "class")= chr [1:3] "simpleError" "error" "condition"
```

Now we can easily find the results that worked, or the inputs that failed:


```r
ok <- map_lgl(out$error, is.null)
ok
#> [1]  TRUE  TRUE  TRUE FALSE

x[!ok]
#> [[1]]
#> [1] "oops"

out$result[ok]
#> [[1]]
#> [1] 1.39
#> 
#> [[2]]
#> [1] 1.27
#> 
#> [[3]]
#> [1] 2.17
```

You can use this same technique in many different situations. For example, imagine you're fitting a generalised linear model (GLM) to a list of data frames. GLMs can sometimes fail because of optimisation problems, but you still want to be able to try to fit all the models, and later look back at those that failed: 


```r
fit_model <- function(df) {
  glm(y ~ x1 + x2 * x3, data = df)
}

models <- transpose(map(datasets, safely(fit_model)))
ok <- map_lgl(models$error, is.null)

# which data failed to converge?
datasets[!ok]

# which models were successful?
models[ok]
```

I think this is a great example of the power of combining functionals and function operators: `safely()` lets you succinctly express what you need to solve a common data analysis problem. 

purrr comes with three other function operators in a similar vein:

* `possibly()`: returns a default value when there's an error.
  It provides no way to tell if an error occured or not, so it's best
  reserved for cases when there's some obvious sentinel value (like `NA`).

* `quietly()`: turns output, messages, and warning side-effects into
  `output`, `message`, and `warning` components of the output.

* `auto_browser()`: automatically executes `browser()` inside the 
  function when there's an error.

See their documentation for more details.

### Caching computations with `memoise::memoise()` {#memoise}
\index{memoisation}
\index{Fibonacci series}

Another handy function operator is `memoise::memoise()`. It __memoises__ a function, meaning that the function will remember previous inputs and return cached results. Memoisation is an example of the classic computer science tradeoff of memory versus speed. A memoised function can run much faster, but because it stores all of the previous inputs and outputs, it uses more memory.

Let's explore this idea with a toy function that simulates an expensive operation:


```r
slow_function <- function(x) {
  Sys.sleep(1)
  x * 10 * runif(1)
}
system.time(print(slow_function(1)))
#> [1] 0.808
#>    user  system elapsed 
#>       0       0       1

system.time(print(slow_function(1)))
#> [1] 8.34
#>    user  system elapsed 
#>   0.003   0.000   1.004
```

When we memoise this function, it's slow when we call it with new arguments. But when we call it with arguments that it's seen before it's instantaneous: it retrieves the previous value of the computation.


```r
fast_function <- memoise::memoise(slow_function)
system.time(print(fast_function(1)))
#> [1] 6.01
#>    user  system elapsed 
#>   0.001   0.000   1.002

system.time(print(fast_function(1)))
#> [1] 6.01
#>    user  system elapsed 
#>   0.017   0.000   0.017
```

A relatively realistic use of memoisation is computing the Fibonacci series. The Fibonacci series is defined recursively: the first two values are defined by convention, $f(0) = 0$, $f(1) = 1$, and then $f(n) = f(n - 1) + f(n - 2)$ (for any positive integer). A naive version is slow because, for example, `fib(10)` computes `fib(9)` and `fib(8)`, and `fib(9)` computes `fib(8)` and `fib(7)`, and so on. 


```r
fib <- function(n) {
  if (n < 2) return(1)
  fib(n - 2) + fib(n - 1)
}
system.time(fib(23))
#>    user  system elapsed 
#>   0.047   0.000   0.047
system.time(fib(24))
#>    user  system elapsed 
#>    0.08    0.00    0.08
```

Memoising `fib()` makes the implementation much faster because each value is computed only once:


```r
fib2 <- memoise::memoise(function(n) {
  if (n < 2) return(1)
  fib2(n - 2) + fib2(n - 1)
})
system.time(fib2(23))
#>    user  system elapsed 
#>   0.007   0.000   0.007
```

And future calls can rely on previous computations:


```r
system.time(fib2(24))
#>    user  system elapsed 
#>   0.001   0.000   0.001
```

This is an example of __dynamic programming__, where a complex problem can be broken down into many overlapping subproblems, and remembering the results of a subproblem considerably improves performance. 

Think carefully before memoising a function. If the function is not __pure__, i.e. the output does not depend only on the input, you will get misleading and confusing results. I created a subtle bug in devtools because I memoised the results of `available.packages()`, which is rather slow because it has to download a large file from CRAN. The available packages don't change that frequently, but if you have an R process that's been running for a few days, the changes can become important, and because the problem only arose in long-running R processes, the bug was very painful to find.

### Exercises

1.  Base R provides a function operator in the form of `Vectorize()`. 
    What does it do? When might you use it?
    
1.  Read the source code for `possibly()`. How does it work?

1.  Read the source code for `safely()`. How does it work?


## Case study: Creating your own function operators {#fo-case-study}
\index{loops}

`meomoise()` and `safely()` are very useful but also quite complex. In this case study you'll learn how to create your own simpler function operators. Imagine you have a named vector of URLs and you'd like to download each one to disk. That's pretty simple with `walk2()` and `file.download()`:


```r
urls <- c(
  "adv-r" = "https://adv-r.hadley.nz", 
  "r4ds" = "http://r4ds.had.co.nz/"
  # and many many more
)
path <- paste(tempdir(), names(urls), ".html")

walk2(urls, path, download.file, quiet = TRUE)
```

This approach is fine for a handful of URLs, but as the vector gets longer, you might want to add a couple more features:

* Add a small delay between each request to avoid hammering the server.

* Display a `.` every few URLs so that we know that the function is still 
  working. 

It's relatively easy to add these extra features if we're using a for loop:


```r
for(i in seq_along(urls)) {
  Sys.sleep(0.1)
  if (i %% 10 == 0) cat(".")
  download.file(urls[[i]], paths[[i]])
}
```

I think this for loop is suboptimal because it interleaves different concerns: pausing, showing progress, and downloading. This makes the code harder to read, and it makes it harder to reuse the components in new situations. Instead, let's see if we can use function operators to extract out pausing and showing progress and make them reusable.

First, let's write a function operator that adds a small delay. I'm going to call it `delay_by()` for reasons that will be more clear shortly, and it has two arguments: the function to wrap, and the amount of delay to add. The actual implementation is quite simple. The main trick is forcing evaluation of all arguments as described in Section \@ref(factory-pitfalls), because function operators are a special type of function factory:


```r
delay_by <- function(f, amount) {
  force(f)
  force(amount)
  
  function(...) {
    Sys.sleep(amount)
    f(...)
  }
}
system.time(runif(100))
#>    user  system elapsed 
#>       0       0       0
system.time(delay_by(runif, 0.1)(100))
#>    user  system elapsed 
#>     0.0     0.0     0.1
```

And we can use it with the original `walk2()`:


```r
walk2(urls, path, delay_by(download.file, 0.1), quiet = TRUE)
```

Creating a function to display the occasional dot is a little harder, because we can no longer rely on the index from the loop. We could pass the index along as another argument, but that breaks encapsulation: a concern of the progress function now becomes a problem that the higher level wrapper needs to handle. Instead, we'll use another function factory trick (from Section \@ref(stateful-funs)), so that the progress wrapper can manage its own internal counter:


```r
dot_every <- function(f, n) {
  force(f)
  force(n)
  
  i <- 0
  function(...) {
    i <<- i + 1
    if (i %% n == 0) cat(".")
    f(...)
  }
}
walk(1:100, runif)
walk(1:100, dot_every(runif, 10))
#> ..........
```

Now we can express our original for loop as:


```r
walk2(
  urls, path, 
  dot_every(delay_by(download.file, 0.1), 10), 
  quiet = TRUE
)
```

This is starting to get a little hard to read because we are composing many function calls, and the arguments are getting spread out. One way to resolve that is to use the pipe:


```r
walk2(
  urls, path, 
  download.file %>% dot_every(10) %>% delay_by(0.1), 
  quiet = TRUE
)
```

The pipe works well here because I've carefully chosen the function names to yield an (almost) readable sentence: take `download.file` then (add) a dot every 10 iterations, then delay by 0.1s. The more clearly you can express the intent of your code through function names, the more easily others (including future you!) can read and understand the code.

### Exercises

1.  Weigh the pros and cons of 
    `download.file %>% dot_every(10) %>% delay_by(0.1)` versus
    `download.file %>% delay_by(0.1) %>% dot_every(10)`.
    
1.  Should you memoise `file.download()`? Why or why not?

1.  Create a function operator that reports whenever a file is created or 
    deleted in the working directory, using `dir()` and `setdiff()`. What other 
    global function effects might you want to track?

1.  Write a function operator that logs a timestamp and message to a file 
    every time a function is run.

1.  Modify `delay_by()` so that instead of delaying by a fixed amount of time, 
    it ensures that a certain amount of time has elapsed since the function 
    was last called. That is, if you called 
    `g <- delay_by(1, f); g(); Sys.sleep(2); g()` there shouldn't be an 
    extra delay.

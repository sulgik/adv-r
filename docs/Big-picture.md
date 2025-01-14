# Big picture {#meta-big-picture}


## Introduction

Metaprogramming is the hardest topic in this book because it brings together many formerly unrelated topics and forces you grapple with issues that you probably haven't thought about before. You'll also need to learn a lot of new vocabulary, and at first it will seem like every new term is defined by three other terms that you haven't heard of. Even if you're an experienced programmer in another language, your existing skills are unlikely to be much help as few modern popular languages expose the level of metaprogramming that R provides. So don't be surprised if you're frustrated or confused at first; this is a natural part of the process that happens to everyone! 

But I think it's easier to learn metaprogramming now than ever before. Over the last few years, the theory and practice have matured substantially, providing a strong foundation paired with tools that allow you to solve common problems. In this chapter, you'll get the big picture of all the main pieces and how they fit together. 

### Outline {-}

Each section in this chapter introduces one big new idea:

* Section \@ref(code-data) shows that code is data and teaches you how to create and modify expressions by capturing code.

* Section \@ref(code-tree) describes the tree-like structure of code, called an 
abstract syntax tree.

* Section \@ref(coding-code) shows how to create new expressions programmatically.

* Section \@ref(eval-intro) shows how to execute expressions by evaluating them in an environment.

* Section \@ref(eval-funs) illustrates how to customise evaluation by supplying custom functions in a new environment.

* Section \@ref(eval-data) extends that customisation to data masks, which blur the line between environments and data frames.

* Section \@ref(quosure-intro) introduces a new data structure called the quosure that makes all this simpler and more correct. 

### Prerequisites {-}

This chapter introduces the big ideas using rlang; you'll learn the base equivalents in later chapters. We'll also use the lobstr package to explore the tree structure of code.


```r
library(rlang)
library(lobstr)
```

Make sure that you're also familiar with the environment (Section \@ref(env-basics)) and data frame (Section \@ref(tibble)) data structures.

## Code is data {#code-data}

The first big idea is that code is data: you can capture code and compute on it as you can with any other type of data. The first way you can capture code is with `rlang::expr()`. You can think of `expr()` as returning exactly what you pass in:


```r
expr(mean(x, na.rm = TRUE))
#> mean(x, na.rm = TRUE)
expr(10 + 100 + 1000)
#> 10 + 100 + 1000
```

More formally, captured code is called an __expression__. An expression isn't a single type of object, but is a collective term for any of four types (call, symbol, constant, or pairlist), which you'll learn more about in Chapter \@ref(expressions).

`expr()` lets you capture code that you've typed. You need a different tool to capture code passed to a function because `expr()` doesn't work:


```r
capture_it <- function(x) {
  expr(x)
}
capture_it(a + b + c)
#> x
```

Here you need to use a function specifically designed to capture user input in a function argument: `enexpr()`. Think of the "en" in the context of "enrich": `enexpr()` takes a lazily evaluated argument and turns it into an expression:


```r
capture_it <- function(x) {
  enexpr(x)
}
capture_it(a + b + c)
#> a + b + c
```

Because `capture_it()` uses `enexpr()` we say that it automatically quotes its first argument. You'll learn more about this term in Section \@ref(vocabulary).

Once you have captured an expression, you can inspect and modify it. Complex expressions behave much like lists. That means you can modify them using `[[` and `$`:


```r
f <- expr(f(x = 1, y = 2))

# Add a new argument
f$z <- 3
f
#> f(x = 1, y = 2, z = 3)

# Or remove an argument:
f[[2]] <- NULL
f
#> f(y = 2, z = 3)
```

The first element of the call is the function to be called, which means the first argument is in the second position. You'll learn the full details in Section \@ref(expression-details).

## Code is a tree {#code-tree}

To do more complex manipulation with expressions, you need to fully understand their structure. Behind the scenes, almost every programming language represents code as a tree, often called the __abstract syntax tree__, or AST for short. R is unusual in that you can actually inspect and manipulate this tree.

A very convenient tool for understanding the tree-like structure is `lobstr::ast()`. Given some code, this function displays the underlying tree structure. Function calls form the branches of the tree, and are shown by rectangles. The leaves of the tree are symbols (like `a`) and constants (like `"b"`).


```r
lobstr::ast(f(a, "b"))
#> █─f 
#> ├─a 
#> └─"b"
```

Nested function calls create more deeply branching trees:


```r
lobstr::ast(f1(f2(a, b), f3(1, f4(2))))
#> █─f1 
#> ├─█─f2 
#> │ ├─a 
#> │ └─b 
#> └─█─f3 
#>   ├─1 
#>   └─█─f4 
#>     └─2
```

Because all function forms can be written in prefix form (Section \@ref(prefix-form)), every R expression can be displayed in this way:


```r
lobstr::ast(1 + 2 * 3)
#> █─`+` 
#> ├─1 
#> └─█─`*` 
#>   ├─2 
#>   └─3
```

Displaying the AST in this way is a useful tool for exploring R's grammar, the topic of Section \@ref(grammar).

## Code can generate code {#coding-code}

As well as seeing the tree from code typed by a human, you can also use code to create new trees. There are two main tools: `call2()` and unquoting.

`rlang::call2()` constructs a function call from its components: the function to call, and the arguments to call it with.


```r
call2("f", 1, 2, 3)
#> f(1, 2, 3)
call2("+", 1, call2("*", 2, 3))
#> 1 + 2 * 3
```

`call2()` is often convenient to program with, but is a bit clunky for interactive use. An alternative technique is to build complex code trees by combining simpler code trees with a template. `expr()` and `enexpr()` have built-in support for this idea via `!!` (pronounced bang-bang), the __unquote operator__.

The precise details are the topic of Section \@ref(unquoting), but basically `!!x` inserts the code tree stored in `x` into the expression. This makes it easy to build complex trees from simple fragments:


```r
xx <- expr(x + x)
yy <- expr(y + y)

expr(!!xx / !!yy)
#> (x + x)/(y + y)
```

Notice that the output preserves the operator precedence so we get `(x + x) / (y + y)` not `x + x / y + y` (i.e. `x + (x / y) + y`). This is important, particularly if you've been wondering if it wouldn't be easier to just paste strings together.

Unquoting gets even more useful when you wrap it up into a function, first using `enexpr()` to capture the user's expression, then `expr()` and `!!` to create a new expression using a template. The example below shows how you can generate an expression that computes the coefficient of variation:


```r
cv <- function(var) {
  var <- enexpr(var)
  expr(sd(!!var) / mean(!!var))
}

cv(x)
#> sd(x)/mean(x)
cv(x + y)
#> sd(x + y)/mean(x + y)
```

(This isn't very useful here, but being able to create this sort of building block is very useful when solving more complex problems.)

Importantly, this works even when given weird variable names:


```r
cv(`)`)
#> sd(`)`)/mean(`)`)
```

Dealing with weird names[^non-syntactic] is another good reason to avoid `paste()` when generating R code. You might think this is an esoteric concern, but not worrying about it when generating SQL code in web applications led to SQL injection attacks that have collectively cost billions of dollars.

[^non-syntactic]: More technically, these are called non-syntactic names and are the topic of Section \@ref(non-syntactic).

## Evaluation runs code {#eval-intro}

Inspecting and modifying code gives you one set of powerful tools. You get another set of powerful tools when you __evaluate__, i.e. execute or run, an expression. Evaluating an expression requires an environment, which tells R what the symbols in the expression mean. You'll learn the details of evaluation in Chapter \@ref(evaluation).

The primary tool for evaluating expressions is `base::eval()`, which takes an expression and an environment:


```r
eval(expr(x + y), env(x = 1, y = 10))
#> [1] 11
eval(expr(x + y), env(x = 2, y = 100))
#> [1] 102
```

If you omit the environment, `eval` uses the current environment:


```r
x <- 10
y <- 100
eval(expr(x + y))
#> [1] 110
```

One of the big advantages of evaluating code manually is that you can tweak the environment. There are two main reasons to do this:

* To temporarily override functions to implement a domain specific language.
* To add a data mask so you can refer to variables in a data frame as if
  they are variables in an environment.

## Customising evaluation with functions {#eval-funs}

The above example used an environment that bound `x` and `y` to vectors. It's less obvious that you also bind names to functions, allowing you to override the behaviour of existing functions. This is a big idea that we'll come back to in Chapter \@ref(translation) where I explore generating HTML and LaTeX from R. The example below gives you a taste of the power. Here I evaluate code in a special environment where `*` and `+` have been overridden to work with strings instead of numbers:


```r
string_math <- function(x) {
  e <- env(
    caller_env(),
    `+` = function(x, y) paste0(x, y),
    `*` = function(x, y) strrep(x, y)
  )

  eval(enexpr(x), e)
}

name <- "Hadley"
string_math("Hello " + name)
#> [1] "Hello Hadley"
string_math(("x" * 2 + "-y") * 3)
#> [1] "xx-yxx-yxx-y"
```

dplyr takes this idea to the extreme, running code in an environment that generates SQL for execution in a remote database:


```r
library(dplyr)
#> 
#> Attaching package: 'dplyr'
#> The following objects are masked from 'package:stats':
#> 
#>     filter, lag
#> The following objects are masked from 'package:base':
#> 
#>     intersect, setdiff, setequal, union

con <- DBI::dbConnect(RSQLite::SQLite(), filename = ":memory:")
mtcars_db <- copy_to(con, mtcars)

mtcars_db %>%
  filter(cyl > 2) %>%
  select(mpg:hp) %>%
  head(10) %>%
  show_query()
#> <SQL>
#> SELECT `mpg`, `cyl`, `disp`, `hp`
#> FROM `mtcars`
#> WHERE (`cyl` > 2.0)
#> LIMIT 10

DBI::dbDisconnect(con)
```

## Customising evaluation with data {#eval-data}

Rebinding functions is an extremely powerful technique, but it tends to require a lot of investment. A more immediately practical application is modifying evaluation to look for variables in a data frame instead of an environment. This idea powers the base `subset()` and `transform()` functions, as well as many tidyverse functions like `ggplot2::aes()` and `dplyr::mutate()`. It's possible to use `eval()` for this, but there are a few potential pitfalls (Section \@ref(base-evaluation)), so we'll switch to `rlang::eval_tidy()` instead.

As well as expression and environment, `eval_tidy()` also takes a __data mask__, which is typically a data frame:


```r
df <- data.frame(x = 1:5, y = sample(5))
eval_tidy(expr(x + y), df)
#> [1] 6 6 4 6 8
```

Evaluating with a data mask is a useful technique for interactive analysis because it allows you to write `x + y` rather than `df$x + df$y`. However, that convenience comes at a cost: ambiguity. In Section \@ref(data-masks) you'll learn how to deal with ambiguity using special `.data` and `.env` pronouns.

We can wrap this pattern up into a function by using `enexpr()`. This gives us a function very similar to `base::with()`:


```r
with2 <- function(df, expr) {
  eval_tidy(enexpr(expr), df)
}

with2(df, x + y)
#> [1] 6 6 4 6 8
```

Unfortunately, this function has a subtle bug and we need a new data structure to help deal with it.

## Quosures {#quosure-intro}

To make the problem more obvious, I'm going to modify `with2()`. The basic problem still occurs without this modification but it's much harder to see.


```r
with2 <- function(df, expr) {
  a <- 1000
  eval_tidy(enexpr(expr), df)
}
```

We can see the problem when we use `with2()` to refer to a variable called `a`. We want the value of `a` to come from the binding we can see (10), not the binding internal to the function (1000):


```r
df <- data.frame(x = 1:3)
a <- 10
with2(df, x + a)
#> [1] 1001 1002 1003
```

The problem arises because we need to evaluate the captured expression in the environment where it was written (where `a` is 10), not the environment inside of `with2()` (where `a` is 1000).

Fortunately we can solve this problem by using a new data structure: the __quosure__ which bundles an expression with an environment. `eval_tidy()` knows how to work with quosures so all we need to do is switch out `enexpr()` for `enquo()`:


```r
with2 <- function(df, expr) {
  a <- 1000
  eval_tidy(enquo(expr), df)
}

with2(df, x + a)
#> [1] 11 12 13
```

Whenever you use a data mask, you must always use `enquo()` instead of `enexpr()`. This is the topic of Chapter \@ref(evaluation).

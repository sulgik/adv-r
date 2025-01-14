# Conditions {#conditions}



## Introduction

The __condition__ system provides a paired set of tools that allow the author of a function to indicate that something unusual is happening, and the user of that function to deal with it. The function author __signals__ conditions with functions like `stop()` (for errors), `warning()` (for warnings), and `message()` (for messages), then the function user can handle them with functions like `tryCatch()` and `withCallingHandlers()`. Understanding the condition system is important because you'll often need to play both roles: signalling conditions from the functions you create, and handle conditions signalled by the functions you call.

R offers a very powerful condition system based on ideas from Common Lisp. Like R's approach to object-oriented programming, it is rather different to currently popular programming languages so it is easy to misunderstand, and there has been relatively little written about how to use it effectively. Historically, this has meant that few people (myself included) have taken full advantage of its power. The goal of this chapter is to remedy that situation. Here you will learn about the big ideas of R's condition system, as well as learning a bunch of practical tools that will make your code stronger.

I found two resources particularly useful when writing this chapter. You may also want to read them if you want to learn more about the inspirations and motivations for the system:

* [_A prototype of a condition system for R_][prototype] by Robert Gentleman 
  and Luke Tierney. This describes an early version of R's condition system. 
  While the implementation has changed somewhat since this document was 
  written, it provides a good overview of how the pieces fit together, and 
  some motivation for its design.

* [_Beyond exception handling: conditions and restarts_][beyond-handling]
  by Peter Seibel. This describes exception handling in Lisp, which happens 
  to be very similar to R's approach. It provides useful motivation and 
  more sophisticated examples. I have provided an R translation of the 
  chapter at <http://adv-r.had.co.nz/beyond-exception-handling.html>.

I also found it helpful to work through the underlying C code that implements these ideas. If you're interested in understanding how it all works, you might find [my notes](https://gist.github.com/hadley/4278d0a6d3a10e42533d59905fbed0ac) to be useful.

### Quiz {-}

Want to skip this chapter? Go for it, if you can answer the questions below. Find the answers at the end of the chapter in Section \@ref(conditions-answers).

1. What are the three most important types of condition?

1. What function do you use to ignore errors in block of code?

1. What's the main difference between `tryCatch()` and `withCallingHandlers()`?

1. Why might you want to create a custom error object?

### Outline {-}

* Section \@ref(signalling-conditions) introduces the basic tools for 
  signalling conditions, and discusses when it is appropriate to use each type. 

* Section \@ref(ignoring-conditions) teaches you about the simplest tools for
  handling conditions: functions like `try()` and `supressMessages()` that 
  swallow conditions and prevent them from getting to the top level.

* Section \@ref(handling-conditions) introduces the condition __object__, and 
  the two fundamental tools of condition handling: `tryCatch()` for error 
  conditions, and `withCallingHandlers()` for everything else.

* Section \@ref(custom-conditions) shows you how to extend the built-in 
  condition objects to store useful data that condition handlers can use to 
  make more informed decisions.

* Section \@ref(condition-applications) closes out the chapter with a grab bag 
  of practical applications based on the low-level tools found in earlier 
  sections.

### Prerequisites

As well as base R functions, this chapter uses condition signalling and handling functions from [rlang](https://rlang.r-lib.org).


```r
library(rlang)
```

## Signalling conditions
\index{conditions!signalling}
\index{interrupts}

There are three conditions that you can signal in code: errors, warnings, and messages. 

* Errors are the most severe; they indicate that there is no way for a function 
  to continue and execution must stop. 

* Warnings fall somewhat in between errors and message, and typically indicate 
  that something has gone wrong but the function has been able to at least 
  partially recover. 
  
* Messages are the mildest; they are way of informing users that some action 
  has been performed on their behalf. 
  
There is a final condition that can only be generated interactively: an interrupt, which indicates that the user has interrupted execution by pressing Escape, Ctrl + Break, or Ctrl + C (depending on the platform).

Conditions are usually displayed prominently, in a bold font or coloured red, depending on the R interface. You can tell them apart because errors always start with "Error", warnings with "Warning" or "Warning message", and messages with nothing.


```r
stop("This is what an error looks like")
#> Error in eval(expr, envir, enclos): This is what an error looks like

warning("This is what a warning looks like")
#> Warning: This is what a warning looks like

message("This is what a message looks like")
#> This is what a message looks like
```

The following three sections describe errors, warnings, and messages in more detail.

### Errors
\index{errors}
\indexc{stop()}
\indexc{abort()}

In base R, errors are signalled, or __thrown__, by `stop()`:


```r
f <- function() g()
g <- function() h()
h <- function() stop("This is an error!")

f()
#> Error in h(): This is an error!
```

By default, the error message includes the call, but this is typically not useful (and recapitulates information that you can easily get from `traceback()`), so I think it's good practice to use `call. = FALSE`[^trailing-dot]:

[^trailing-dot]: The trailing `.` in `call.` is a peculiarity of `stop()`; don't read anything into it.


```r
h <- function() stop("This is an error!", call. = FALSE)
f()
#> Error: This is an error!
```

The rlang equivalent to `stop()`, `rlang::abort()`, does this automatically. We'll use `abort()` throughout this chapter, but we won't get to its most compelling feature, the ability to add additional metadata to the condition object, until we're near the end of the chapter.


```r
h <- function() abort("This is an error!")
f()
#> Error: This is an error!
```

(NB: `stop()` pastes together multiple inputs, while `abort()` does not. To create complex error messages with abort, I recommend using `glue::glue()`. This allows us to use other arguments to `abort()` for useful features that you'll learn about in Section \@ref(custom-conditions).)

The best error messages tell you what is wrong and point you in the right direction to fix the problem. Writing good error messages is hard because errors usually occur when the user has a flawed mental model of the function. As a developer, it's hard to imagine how the user might be thinking incorrectly about your function, and thus it's hard to write a message that will steer the user in the correct direction. That said, the tidyverse style guide discusses a few general principles that we have found useful: <http://style.tidyverse.org/error-messages.html>.

### Warnings
\index{warnings}

Warnings, signalled by `warning()`, are weaker than errors: they signal that something has gone wrong, but the code has been able to recover and continue. Unlike errors, you can have multiple warnings from a single function call:


```r
fw <- function() {
  cat("1\n")
  warning("W1")
  cat("2\n")
  warning("W2")
  cat("3\n")
  warning("W3")
}
```

By default, warnings are cached and printed only when control returns to the top level:


```r
fw()
#> 1
#> 2
#> 3
#> Warning messages:
#> 1: In f() : W1
#> 2: In f() : W2
#> 3: In f() : W3
```

\index{options!warn@\texttt{warn}}
You can control this behaviour with the `warn` option:

* To make warnings appear immediately, set `options(warn = 1)`. 

* To turn warnings into errors, set `options(warn = 2)`. This is usually
  the easiest way to debug a warning, as once it's an error you can
  use tools like `traceback()` to find the source. 

* Restore the default behaviour with `options(warn = 0)`.

Like `stop()`, `warning()` also has a call argument. It is slightly more useful (since warnings are often more distant from their source), but I still generally suppress it with `call. = FALSE`. Like `rlang::abort()`, the rlang equivalent of `warning()`, `rlang::warn()`, also suppresses the `call.` by default.

Warnings occupy a somewhat challenging place between messages ("you should know about this") and errors ("you must fix this!"), and it's hard to give precise advice on when to use them. Generally, be restrained, as warnings are easy to miss if there's a lot of other output, and you don't want your function to recover too easily from clearly invalid input. In my opinion, base R tends to overuse warnings, and many warnings in base R would be better off as errors. For example, I think these warnings would be more helpful as errors:


```r
formals(1)
#> Warning in formals(fun): argument is not a function
#> NULL

file.remove("this-file-doesn't-exist")
#> Warning in file.remove("this-file-doesn't-exist"): cannot remove file
#> 'this-file-doesn't-exist', reason 'No such file or directory'
#> [1] FALSE

lag(1:3, k = 1.5)
#> Warning in lag.default(1:3, k = 1.5): 'k' is not an integer
#> [1] 1 2 3
#> attr(,"tsp")
#> [1] -1  1  1

as.numeric(c("18", "30", "50+", "345,678"))
#> Warning: NAs introduced by coercion
#> [1] 18 30 NA NA
```

There are only a couple of cases where using a warning is clearly appropriate:

* When you __deprecate__ a function you want to allow older code to continue
  to work (so ignoring the warning is OK) but you want to encourage the user
  to switch to a new function.

* When you are reasonably certain you can recover from a problem:
  If you were 100% certain that you could fix the problem, you wouldn't need 
  any message; if you were more uncertain that you could correctly fix the 
  issue, you'd throw an error.

Otherwise use warnings with restraint, and carefully consider if an error would be more appropriate.

### Messages
\index{messages}
\indexc{cat()}
\indexc{packageStartupMessage()}

Messages, signalled by `message()`, are informational; use them to tell the user that you've done something on their behalf. Good messages are a balancing act: you want to provide just enough information so the user knows what's going on, but not so much that they're overwhelmed.

`message()`s are displayed immediately and do not have a `call.` argument:


```r
fm <- function() {
  cat("1\n")
  message("M1")
  cat("2\n")
  message("M2")
  cat("3\n")
  message("M3")
}

fm()
#> 1
#> M1
#> 2
#> M2
#> 3
#> M3
```

Good places to use a message are:

* When a default argument requires some non-trivial amount of computation
  and you want to tell the user what value was used. For example, ggplot2
  reports the number of bins used if you don't supply a `binwidth`.
  
* In functions that are called primarily for their side-effects which would 
  otherwise be silent. For example, when writing files to disk, calling a web 
  API, or writing to a database, it's useful to provide regular status messages 
  telling the user what's happening.

* When you're about to start a long running process with no 
  intermediate output. A progress bar (e.g. with 
  [progress](https://github.com/r-lib/progress)) is better, but a message 
  is a good place to start.

* When writing a package, you sometimes want to display a message when
  your package is loaded (i.e. in `.onAttach()`); here you must use 
  `packageStartupMessage()`.

Generally any function that produces a message should have some way to suppress it, like a `quiet = TRUE` argument. It is possible to suppress all messages with `suppressMessages()`, as you'll learn shortly, but it is nice to also give finer grained control.

It's important to compare `message()` to the closely related `cat()`. In terms of usage and result, they appear quite similar[^cat]:

[^cat]: But note that `cat()` requires an explicit trailing `"\n"` to print a new line.


```r
cat("Hi!\n")
#> Hi!

message("Hi!")
#> Hi!
```

However, the _purposes_ of `cat()` and `message()` are different. Use `cat()` when the primary role of the function is to print to the console, like `print()` or `str()` methods. Use `message()` as a side-channel to print to the console when the primary purpose of the function is something else.  In other words, `cat()` is for when the user _asks_ for something to be printed and `message()` is for when the developer _elects_ to print something.

### Exercises

1.  Write a wrapper around `file.remove()` that throws an error if the file
    to be deleted does not exist.
   
1.  What does the `appendLF` argument to `message()` do? How is it related to
    `cat()`?

## Ignoring conditions 
\index{conditions!muffling}
\indexc{try()} 
\indexc{suppressWarnings()} 
\indexc{suppressMessages()}  

The simplest way of handling conditions in R is to simply ignore them:

* Ignore errors with `try()`.
* Ignore warnings with `suppressWarnings()`.
* Ignore messages with `suppressMessages()`.

These functions are heavy handed as you can't use them to suppress a single type of condition that you know about, while allowing everything else to pass through. We'll come back to that challenge later in the chapter. 

`try()` allows execution to continue even after an error has occurred. Normally if you run a function that throws an error, it terminates immediately and doesn't return a value: 


```r
f1 <- function(x) {
  log(x)
  10
}
f1("x")
#> Error in log(x): non-numeric argument to mathematical function
```

However, if you wrap the statement that creates the error in `try()`, the error message will be displayed[^silent] but execution will continue:

[^silent]: You can suppress the message with `try(..., silent = TRUE)`.


```r
f2 <- function(x) {
  try(log(x))
  10
}
f2("a")
#> Error in log(x) : non-numeric argument to mathematical function
#> [1] 10
```

It is possible, but not recommended, to save the result of `try()` and perform different actions based on whether or not the code succeeded or failed[^try-error]. Instead, it is better to use `tryCatch()` or a higher-level helper; you'll learn about those shortly. 

A simple, but useful, pattern is to do assignment inside the call: this lets you define a default value to be used if the code does not succeed. This works because the argument is evaluated in the calling environment, not inside the function. (See Section \@ref(promises) for more details.)

[^try-error]: You can tell if the expression failed because the result will have class `try-error`.


```r
default <- NULL
try(default <- read.csv("possibly-bad-input.csv"), silent = TRUE)
```

`suppressWarnings()` and `suppressMessages()` suppress all warnings and messages. Unlike errors, messages and warnings don't terminate execution, so there may be multiple warnings and messages signalled in a single block.


```r
suppressWarnings({
  warning("Uhoh!")
  warning("Another warning")
  1
})
#> [1] 1

suppressMessages({
  message("Hello there")
  2
})
#> [1] 2

suppressWarnings({
  message("You can still see me")
  3
})
#> You can still see me
#> [1] 3
```

## Handling conditions
\index{errors!catching}
\index{conditions!handling}
\indexc{tryCatch()}
\indexc{withCallingHandlers()}

Every condition has default behaviour: errors stop execution and return to the top level, warnings are captured and displayed in aggregate, and messages are immediately displayed. Condition __handlers__ allow us to temporarily override or supplement the default behaviour. 

Two functions, `tryCatch()` and `withCallingHandlers()`, allow us to register handlers, functions that take the signalled condition as their single argument. The registration functions have the same basic form:


```r
tryCatch(
  error = function(cnd) {
    # code to run when error is thrown
  },
  code_to_run_while_handlers_are_active
)

withCallingHandlers(
  warning = function(cnd) {
    # code to run when warning is signalled
  },
  message = function(cnd) {
    # code to run when message is signalled
  },
  code_to_run_while_handlers_are_active
)
```

They differ in the type of handlers that they create:

*  `tryCatch()` defines __exiting__ handlers; after the condition is handled, 
   control returns to the context where `tryCatch()` was called. This makes 
   `tryCatch()` most suitable for working with errors and interrupts, as these 
   have to exit anyway.
  
*  `withCallingHandlers()` defines __calling__ handlers; after the condition
   is captured control returns to the context where the condition was signalled.
   This makes it most suitable for working with non-error conditions.

But before we can learn about and use these handlers, we need to talk a little bit about condition __objects__. These are created implicitly whenever you signal a condition, but become explicit inside the handler.

### Condition objects
\index{conditions!objects}
\indexc{catch\_cnd()}

So far we've just signalled conditions, and not looked at the objects that are created behind the scenes. The easiest way to see a condition object is to catch one from a signalled condition. That's the job of `rlang::catch_cnd()`:


```r
cnd <- catch_cnd(stop("An error"))
str(cnd)
#> List of 2
#>  $ message: chr "An error"
#>  $ call   : language force(expr)
#>  - attr(*, "class")= chr [1:3] "simpleError" "error" "condition"
```

Built-in conditions are lists with two elements: 

* `message`, a length-1 character vector containing the text to display to a user.
  To extract the message, use `conditionMessage(cnd)`.

* `call`, the call which triggered the condition. As described above, we don't
  use the call, so it will often be `NULL`. To extract it, use 
  `conditionCall(cnd)`.

Custom conditions may contain other components, which we'll discuss in Section \@ref(custom-conditions).

Conditions also have a `class` attribute, which makes them S3 objects. We won't discuss S3 until Chapter \@ref(s3), but fortunately, even if you don't know about S3, condition objects are quite simple. The most important thing to know is that the `class` attribute is a character vector, and it determines which handlers will match the condition.

### Exiting handlers
\indexc{tryCatch()} 
\index{handlers!exiting}

`tryCatch()` registers exiting handlers, and is typically used to handle error conditions. It allows you to override the default error behaviour. For example, the following code will return `NA` instead of throwing an error:


```r
f3 <- function(x) {
  tryCatch(
    error = function(cnd) NA,
    log(x)
  )
}

f3("x")
#> [1] NA
```

If no conditions are signalled, or the class of the signalled condition does not match the handler name, the code executes normally:


```r
tryCatch(
  error = function(cnd) 10,
  1 + 1
)
#> [1] 2

tryCatch(
  error = function(cnd) 10,
  {
    message("Hi!")
    1 + 1
  }
)
#> Hi!
#> [1] 2
```

The handlers set up by `tryCatch()` are called __exiting__ handlers because after the condition is signalled, control passes to the handler and never returns to the original code, effectively meaning that the code exits:


```r
tryCatch(
  message = function(cnd) "There",
  {
    message("Here")
    stop("This code is never run!")
  }
)
#> [1] "There"
```

The protected code is evaluated in the environment of `tryCatch()`, but the handler code is not, because the handlers are functions. This is important to remember if you're trying to modify objects in the parent environment.

The handler functions are called with a single argument, the condition object. I call this argument `cnd`, by convention. This value is only moderately useful for the base conditions because they contain relatively little data. It's more useful when you make your own custom conditions, as you'll see shortly.


```r
tryCatch(
  error = function(cnd) {
    paste0("--", conditionMessage(cnd), "--")
  },
  stop("This is an error")
)
#> [1] "--This is an error--"
```

`tryCatch()` has one other argument: `finally`. It specifies a block of code (not a function) to run regardless of whether the initial expression succeeds or fails. This can be useful for clean up, like  deleting files, or closing connections. This is functionally equivalent to using `on.exit()` (and indeed that's how it's implemented) but it can wrap smaller chunks of code than an entire function. 
\indexc{on.exit()}


```r
path <- tempfile()
tryCatch(
  {
    writeLines("Hi!", path)
    # ...
  },
  finally = {
    # always run
    unlink(path)
  }
)
```

### Calling handlers
\index{handlers!calling}
\indexc{withCallingHandlers()}

The handlers set up by `tryCatch()` are called exiting handlers, because they cause code to exit once the condition has been caught. By contrast, `withCallingHandlers()` sets up __calling__  handlers: code execution continues normally once the handler returns. This tends to make `withCallingHandlers()` a more natural pairing with the non-error conditions. Exiting and calling handlers use "handler" in slighty different senses: 

* An exiting handler handles a signal like you handle a problem; it makes the 
  problem go away. 
  
* A calling handler handles a signal like you handle a car; the car still 
  exists.

Compare the results of `tryCatch()` and `withCallingHandlers()` in the example below. The messages are not printed in the first case, because the code is terminated once the exiting handler completes. They are printed in the second case, because a calling handler does not exit.


```r
tryCatch(
  message = function(cnd) cat("Caught a message!\n"), 
  {
    message("Someone there?")
    message("Why, yes!")
  }
)
#> Caught a message!

withCallingHandlers(
  message = function(cnd) cat("Caught a message!\n"), 
  {
    message("Someone there?")
    message("Why, yes!")
  }
)
#> Caught a message!
#> Someone there?
#> Caught a message!
#> Why, yes!
```

Handlers are applied in order, so you don't need to worry about getting caught in an infinite loop. In the following example, the `message()` signalled by the handler doesn't also get caught:


```r
withCallingHandlers(
  message = function(cnd) message("Second message"),
  message("First message")
)
#> Second message
#> First message
```

(But beware if you have multiple handlers, and some handlers signal conditions that could be captured by another handler: you'll need to think through the order carefully.)

The return value of a calling handler is ignored because the code continues to execute after the handler completes; where would the return value go? That means that calling handlers are only useful for their side-effects. 

\index{conditions!muffling}
\indexc{cnd\_muffle()}

One important side-effect unique to calling handlers is the ability to __muffle__ the signal. By default, a condition will continue to propagate to parent handlers, all the way up to the default handler (or an exiting handler, if provided):


```r
# Bubbles all the way up to default handler which generates the message
withCallingHandlers(
  message = function(cnd) cat("Level 2\n"),
  withCallingHandlers(
    message = function(cnd) cat("Level 1\n"),
    message("Hello")
  )
)
#> Level 1
#> Level 2
#> Hello

# Bubbles up to tryCatch
tryCatch(
  message = function(cnd) cat("Level 2\n"),
  withCallingHandlers(
    message = function(cnd) cat("Level 1\n"),
    message("Hello")
  )
)
#> Level 1
#> Level 2
```

If you want to prevent the condition "bubbling up" but still run the rest of the code in the block, you need to explicitly muffle it with `rlang::cnd_muffle()`:


```r
# Muffles the default handler which prints the messages
withCallingHandlers(
  message = function(cnd) {
    cat("Level 2\n")
    cnd_muffle(cnd)
  },
  withCallingHandlers(
    message = function(cnd) cat("Level 1\n"),
    message("Hello")
  )
)
#> Level 1
#> Level 2

# Muffles level 2 handler and the default handler
withCallingHandlers(
  message = function(cnd) cat("Level 2\n"),
  withCallingHandlers(
    message = function(cnd) {
      cat("Level 1\n")
      cnd_muffle(cnd)
    },
    message("Hello")
  )
)
#> Level 1
```

### Call stacks
\index{call stacks}
\indexc{cnd\_muffle()}

To complete the section, there are some important differences between the call stacks of exiting and calling handlers. These differences are generally not important but I'm including them here because I've occasionally found them useful, and don't want to forget about them!

It's easiest to see the difference by setting up a small example that uses `lobstr::cst()`:


```r
f <- function() g()
g <- function() h()
h <- function() message("!")
```

Calling handlers are called in the context of the call that signalled the condition:
  

```r
withCallingHandlers(f(), message = function(cnd) {
  lobstr::cst()
  cnd_muffle(cnd)
})
#>      █
#>   1. ├─base::withCallingHandlers(...)
#>   2. ├─global::f()
#>   3. │ └─global::g()
#>   4. │   └─global::h()
#>   5. │     └─base::message("!")
#>   6. │       ├─base::withRestarts(...)
#>   7. │       │ └─base:::withOneRestart(expr, restarts[[1L]])
#>   8. │       │   └─base:::doWithOneRestart(return(expr), restart)
#>   9. │       └─base::signalCondition(cond)
#>  10. └─(function (cnd) ...
#>  11.   └─lobstr::cst()
```

Whereas exiting handlers are called in the context of the call to `tryCatch()`:


```r
tryCatch(f(), message = function(cnd) lobstr::cst())
#>     █
#>  1. └─base::tryCatch(f(), message = function(cnd) lobstr::cst())
#>  2.   └─base:::tryCatchList(expr, classes, parentenv, handlers)
#>  3.     └─base:::tryCatchOne(expr, names, parentenv, handlers[[1L]])
#>  4.       └─value[[3L]](cond)
#>  5.         └─lobstr::cst()
```

### Exercises

1.  What extra information does the condition generated by `abort()` contain
    compared to the condition generated by `stop()` i.e. what's the difference
    between these two objects? Read the help for `?abort` to learn more.
    
    
    ```r
    catch_cnd(stop("An error"))
    catch_cnd(abort("An error"))
    ```

1.  Predict the results of evaluating the following code

    
    ```r
    show_condition <- function(code) {
      tryCatch(
        error = function(cnd) "error",
        warning = function(cnd) "warning",
        message = function(cnd) "message",
        {
          code
          NULL
        }
      )
    }
    
    show_condition(stop("!"))
    show_condition(10)
    show_condition(warning("?!"))
    show_condition({
      10
      message("?")
      warning("?!")
    })
    ```

1.  Explain the results of running this code:

    
    ```r
    withCallingHandlers(
      message = function(cnd) message("b"),
      withCallingHandlers(
        message = function(cnd) message("a"),
        message("c")
      )
    )
    #> b
    #> a
    #> b
    #> c
    ```

1.  Read the source code for `catch_cnd()` and explain how it works.

1.  How could you rewrite `show_condition()` to use a single handler?

## Custom conditions {#custom-conditions}
\index{conditions!custom}
\indexc{abort()}

One of the challenges of error handling in R is that most functions generate one of the built-in conditions, which contain only a `message` and a `call`. That means that if you want to detect a specific type of error, you can only work with the text of the error message. This is error prone, not only because the message might change over time, but also because messages can be translated into other languages. 

Fortunately R has a powerful, but little used feature: the ability to create custom conditions that can contain additional metadata. Creating custom conditions is a little fiddly in base R, but `rlang::abort()` makes it very easy as you can supply a custom `.subclass` and additional metadata. 

The following example shows the basic pattern. I recommend using the following call structure for custom conditions. This takes advantage of R's flexible argument matching so that the name of the type of error comes first, followed by the user-facing text, followed by custom metadata.


```r
abort(
  "error_not_found",
  message = "Path `blah.csv` not found", 
  path = "blah.csv"
)
#> Error: Path `blah.csv` not found
```

Custom conditions work just like regular conditions when used interactively, but allow handlers to do much more.

### Motivation

To explore these ideas in more depth, let's take `base::log()`. It does the minimum when throwing errors  caused by invalid arguments:


```r
log(letters)
#> Error in log(letters): non-numeric argument to mathematical function
log(1:10, base = letters)
#> Error in log(1:10, base = letters): non-numeric argument to mathematical function
```

I think we can do better by being explicit about which argument is the problem (i.e. `x` or `base`), and saying what the problematic input is (not just what it isn't).


```r
my_log <- function(x, base = exp(1)) {
  if (!is.numeric(x)) {
    abort(paste0(
      "`x` must be a numeric vector; not ", typeof(x), "."
    ))
  }
  if (!is.numeric(base)) {
    abort(paste0(
      "`base` must be a numeric vector; not ", typeof(base), "."
    ))
  }

  base::log(x, base = base)
}
```

This gives us:


```r
my_log(letters)
#> Error: `x` must be a numeric vector; not character.
my_log(1:10, base = letters)
#> Error: `base` must be a numeric vector; not character.
```

This is an improvement for interactive usage as the error messages are more likely to guide the user towards a correct fix. However, they're no better if you want to programmatically handle the errors: all the useful metadata about the error is jammed into a single string.

### Signalling
\index{conditions!signalling}

Let's build some infrastructure to improve this situation, We'll start by providing a custom `abort()` function for bad arguments. This is a little over-generalised for the example at hand, but it reflects common patterns that I've seen across other functions. The pattern is fairly simple. We create a nice error message for the user, using `glue::glue()`, and store metadata in the condition call for the developer. 


```r
abort_bad_argument <- function(arg, must, not = NULL) {
  msg <- glue::glue("`{arg}` must {must}")
  if (!is.null(not)) {
    not <- typeof(not)
    msg <- glue::glue("{msg}; not {not}.")
  }
  
  abort("error_bad_argument", 
    message = msg, 
    arg = arg, 
    must = must, 
    not = not
  )
}
```

\newpage
::: base

If you want to throw a custom error without adding a dependency on rlang, you can create a condition object "by hand" and then pass it to `stop()`:


```r
stop_custom <- function(.subclass, message, call = NULL, ...) {
  err <- structure(
    list(
      message = message,
      call = call,
      ...
    ),
    class = c(.subclass, "error", "condition")
  )
  stop(err)
}

err <- catch_cnd(
  stop_custom("error_new", "This is a custom error", x = 10)
)
class(err)
err$x
```

:::

We can now rewrite `my_log()` to use this new helper:


```r
my_log <- function(x, base = exp(1)) {
  if (!is.numeric(x)) {
    abort_bad_argument("x", must = "be numeric", not = x)
  }
  if (!is.numeric(base)) {
    abort_bad_argument("base", must = "be numeric", not = base)
  }

  base::log(x, base = base)
}
```

`my_log()` itself is not much shorter, but is a little more meangingful, and it ensures that error messages for bad arguments are consistent across functions. It yields the same interactive error messages as before:


```r
my_log(letters)
#> Error: `x` must be numeric; not character.
my_log(1:10, base = letters)
#> Error: `base` must be numeric; not character.
```

### Handling
\index{conditions!handling}

These structured condition objects are much easier to program with. The first place you might want to use this capability is when testing your function. Unit testing is not a subject of this book (see [R packages](http://r-pkgs.had.co.nz/) for details), but the basics are easy to understand. The following code captures the error, and then asserts it has the structure that we expect.


```r
library(testthat)

err <- catch_cnd(my_log("a"))
expect_s3_class(err, "error_bad_argument")
expect_equal(err$arg, "x")
expect_equal(err$not, "character")
```

We can also use the class (`error_bad_argument`) in `tryCatch()` to only handle that specific error:


```r
tryCatch(
  error_bad_argument = function(cnd) "bad_argument",
  error = function(cnd) "other error",
  my_log("a")
)
#> [1] "bad_argument"
```

When using `tryCatch()` with multiple handlers and custom classes, the first handler to match any class in the signal's class vector is called, not the best match. For this reason, you need to make sure to put the most specific handlers first. The following code does not do what you might hope:


```r
tryCatch(
  error = function(cnd) "other error",
  error_bad_argument = function(cnd) "bad_argument",
  my_log("a")
)
#> [1] "other error"
```

### Exercises

1.  Inside a package, it's occasionally useful to check that a package is 
    installed before using it. Write a function that checks if a package is 
    installed (with `requireNamespace("pkg", quietly = FALSE))` and if not,
    throws a custom condition that includes the package name in the metadata.
    
1.  Inside a package you often need to stop with an error when something
    is not right. Other packages that depend on your package might be
    tempted to check these errors in their unit tests. How could you help
    these packages to avoid relying on the error message which is part of
    the user interface rather than the API and might change without notice?

## Applications {#condition-applications}

Now that you've learned the basic tools of R's condition system, it's time to dive into some applications. The goal of this section is not to show every possible usage of `tryCatch()` and `withCallingHandlers()` but to illustrate some common patterns that frequently crop up. Hopefully these will get your creative juices flowing, so when you encounter a new problem you can come up with a useful solution.

### Failure value
\indexc{try()}

There are a few simple, but useful, `tryCatch()` patterns based on returning a value from the error handler. The simplest case is a wrapper to return a default value if an error occurs:


```r
fail_with <- function(expr, value = NULL) {
  tryCatch(
    error = function(cnd) value,
    expr
  )
}

fail_with(log(10), NA_real_)
#> [1] 2.3
fail_with(log("x"), NA_real_)
#> [1] NA
```

A more sophisticated application is `base::try()`. Below, `try2()` extracts the essence of `base::try()`; the real function is more complicated in order to make the error message look more like what you'd see if `tryCatch()` wasn't used. 


```r
try2 <- function(expr, silent = FALSE) {
  tryCatch(
    error = function(cnd) {
      msg <- conditionMessage(cnd)
      if (!silent) {
        message("Error: ", msg)
      }
      structure(msg, class = "try-error")
    },
    expr
  )
}

try2(1)
#> [1] 1
try2(stop("Hi"))
#> Error: Hi
#> [1] "Hi"
#> attr(,"class")
#> [1] "try-error"
try2(stop("Hi"), silent = TRUE)
#> [1] "Hi"
#> attr(,"class")
#> [1] "try-error"
```

### Success and failure values {#try-success-failure}

We can extend this pattern to return one value if the code evaluates successfully (`success_val`), and another if it fails (`error_val`). This pattern just requires one small trick: evaluating the user supplied code, then `success_val`. If the code throws an error, we'll never get to `success_val` and will instead return `error_val`.


```r
foo <- function(expr) {
  tryCatch(
    error = function(cnd) error_val,
    {
      expr
      success_val
    }
  )
}
```

We can use this to determine if an expression fails:


```r
does_error <- function(expr) {
  tryCatch(
    error = function(cnd) TRUE,
    {
      expr
      FALSE
    }
  )
}
```

Or to capture any condition, like just `rlang::catch_cnd()`:


```r
catch_cnd <- function(expr) {
  tryCatch(
    condition = function(cnd) cnd, 
    {
      expr
      NULL
    }
  )
}
```

We can also use this pattern to create a `try()` variant. One challenge with `try()` is that it's slightly challenging to determine if the code succeeded or failed. Rather than returning an object with a special class, I think it's slightly nicer to return a list with two components `result` and `error`.


```r
safety <- function(expr) {
  tryCatch(
    error = function(cnd) {
      list(result = NULL, error = cnd)
    },
    list(result = expr, error = NULL)
  )
}

str(safety(1 + 10))
#> List of 2
#>  $ result: num 11
#>  $ error : NULL
str(safety(stop("Error!")))
#> List of 2
#>  $ result: NULL
#>  $ error :List of 2
#>   ..$ message: chr "Error!"
#>   ..$ call   : language doTryCatch(return(expr), name, parentenv, h..
#>   ..- attr(*, "class")= chr [1:3] "simpleError" "error" "condition"
```

(This is closely related to `purrr::safely()`, a function operator, which we'll come back to in Section \@ref(safely).)

### Resignal
\index{options!warn@\texttt{warn}}

As well as returning default values when a condition is signalled, handlers can be used to make more informative error messages. One simple application is to make a function that works like `options(warn = 2)` for a single block of code. The idea is simple: we handle warnings by throwing an error:


```r
warning2error <- function(expr) {
  withCallingHandlers(
    warning = function(cnd) abort(conditionMessage(cnd)),
    expr
  )
}
```


```r
warning2error({
  x <- 2 ^ 4
  warn("Hello")
})
#> Error: Hello
```

You could write a similar function if you were trying to find the source of an annoying message. More on this in Section \@ref(non-error-failures).

<!-- 
Another common place where it's useful to add additional context dependent information. For example, you might have a function to download data from a remote website:


```r
download_data <- function(name) {
  src <- paste0("http://awesomedata.com/", name, ".csv")
  dst <- paste0("data/", name, ".csv")
  
  tryCatch(
    curl::curl_download(src, dst),
    error = function(cnd) {
      abort(
        glue::glue("Failed to download remote data `{name}`"), 
        parent = c
      )
    }
  )
}
```

There are two important ideas here:

* We rewrap `curl_download()`, which downloads the file, to provide context
  specific to our function.
  
* We include the original error as the `parent` so that the original context is
  still available.

-->

### Record

Another common pattern is to record conditions for later investigation. The new challenge here is that calling handlers are called only for their side-effects so we can't return values, but instead need to modify some object in place.


```r
catch_cnds <- function(expr) {
  conds <- list()
  add_cond <- function(cnd) {
    conds <<- append(conds, list(cnd))
    cnd_muffle(cnd)
  }
  
  withCallingHandlers(
    message = add_cond,
    warning = add_cond,
    expr
  )
  
  conds
}

catch_cnds({
  inform("a")
  warn("b")
  inform("c")
})
#> [[1]]
#> <message: a
#> >
#> 
#> [[2]]
#> <warning: b>
#> 
#> [[3]]
#> <message: c
#> >
```

What if you also want to capture errors? You'll need to wrap the `withCallingHandlers()` in a `tryCatch()`. If an error occurs, it will be the last condition.


```r
catch_cnds <- function(expr) {
  conds <- list()
  add_cond <- function(cnd) {
    conds <<- append(conds, list(cnd))
    cnd_muffle(cnd)
  }
  
  tryCatch(
    error = function(cnd) {
      conds <<- append(conds, list(cnd))
    },
    withCallingHandlers(
      message = add_cond,
      warning = add_cond,
      expr
    )
  )
  
  conds
}

catch_cnds({
  inform("a")
  warn("b")
  abort("C")
})
#> [[1]]
#> <message: a
#> >
#> 
#> [[2]]
#> <warning: b>
#> 
#> [[3]]
#> <error/rlang_error>
#> C
#> Backtrace:
#>  1. global::catch_cnds(...)
#>  6. base::withCallingHandlers(...)
```

This is the key idea underlying the evaluate package [@evaluate] which powers knitr: it captures every output into a special data structure so that it can be later replayed. As a whole, the evaluate package is quite a lot more complicated than the code here because it also needs to handle plots and text output.

### No default behaviour
\indexc{signal()}
\indexc{cnd\_muffle()}

A final useful pattern is to signal a condition that doesn't inherit from `message`, `warning` or `error`. Because there is no default behaviour, this means the condition has no effect unless the user specifically requests it. For example, you could imagine a logging system based on conditions:


```r
log <- function(message, level = c("info", "error", "fatal")) {
  level <- match.arg(level)
  signal(message, "log", level = level)
}
```

When you call `log()` a condition is signalled, but nothing happens because it has no default handler:


```r
log("This code was run")
```

To activate logging you need a handler that does something with the `log` condition. Below I define a `record_log()` function that will record all logging messages to a file:


```r
record_log <- function(expr, path = stdout()) {
  withCallingHandlers(
    log = function(cnd) {
      cat(
        "[", cnd$level, "] ", cnd$message, "\n", sep = "",
        file = path, append = TRUE
      )
    },
    expr
  )
}

record_log(log("Hello"))
#> [info] Hello
```

You could even imagine layering with another function that allows you to selectively suppress some logging levels.


```r
ignore_log_levels <- function(expr, levels) {
  withCallingHandlers(
    log = function(cnd) {
      if (cnd$level %in% levels) {
        cnd_muffle(cnd)
      }
    },
    expr
  )
}

record_log(ignore_log_levels(log("Hello"), "info"))
```

:::base
If you create a condition object by hand, and signal it with `signalCondition()`,  `cnd_muffle()` will not work. Instead you need to call it with a muffle restart defined, like this:

```R
withRestarts(signalCondition(cond), muffle = function() NULL)
```

Restarts are currently beyond the scope of the book, but I suspect will be included in the third edition.
:::


### Exercises

1.  Create `suppressConditions()` that works like `suppressMessages()` and 
    `suppressWarnings()` but suppresses everything. Think carefully about how you
    should handle errors.

1.  Compare the following two implementations of `message2error()`. What is the
    main advantage of `withCallingHandlers()` in this scenario? (Hint: look
    carefully at the traceback.)

    
    ```r
    message2error <- function(code) {
      withCallingHandlers(code, message = function(e) stop(e))
    }
    message2error <- function(code) {
      tryCatch(code, message = function(e) stop(e))
    }
    ```
    
1.  How would you modify the `catch_cnds()` definition if you wanted to recreate
    the original intermingling of warnings and messages?

1.  Why is catching interrupts dangerous? Run this code to find out.

    
    ```r
    bottles_of_beer <- function(i = 99) {
      message(
        "There are ", i, " bottles of beer on the wall, ", 
        i, " bottles of beer."
      )
      while(i > 0) {
        tryCatch(
          Sys.sleep(1),
          interrupt = function(err) {
            i <<- i - 1
            if (i > 0) {
              message(
                "Take one down, pass it around, ", i, 
                " bottle", if (i > 1) "s", " of beer on the wall."
              )
            }
          }
        )
      }
      message(
        "No more bottles of beer on the wall, ", 
        "no more bottles of beer."
      )
    }
    ```


## Quiz answers {#conditions-answers}

1. `error`, `warning`, and `message`.

1. You could use `try()` or `tryCatch()`.

1. `tryCatch()` creates exiting handlers which will terminate the execution
   of wrapped code; `withCallingHandlers()` creates calling handlers which 
   don't affect the execution of wrapped code.

1. Because you can then capture specific types of error with `tryCatch()`,
   rather than relying on the comparison of error strings, which is risky,
   especially when messages are translated.

[prototype]: http://homepage.stat.uiowa.edu/~luke/R/exceptions/simpcond.html
[beyond-handling]: http://www.gigamonkeys.com/book/beyond-exception-handling-conditions-and-restarts.html

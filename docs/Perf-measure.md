# Measuring performance {#perf-measure}
\index{performance!measuring}



## Introduction

> Programmers waste enormous amounts of time thinking about, or worrying 
> about, the speed of noncritical parts of their programs, and these attempts 
> at efficiency actually have a strong negative impact when debugging and 
> maintenance are considered.
>
> --- Donald Knuth.

Before you can make your code faster, you first need to figure out what's making it slow. This sounds easy, but it's not. Even experienced programmers have a hard time identifying bottlenecks in their code. So instead of relying on your intuition, you should __profile__ your code: measure the run-time of each line of code using realistic inputs.

Once you've identified bottlenecks you'll need to carefully experiment with alternatives to find faster code that is still equivalent. In Chapter  \@ref(perf-improve) you'll learn a bunch of ways to speed up code, but first you need to learn how to __microbenchmark__ so that you can precisely measure the difference in performance.

### Outline {-}

* Section \@ref(profiling) shows you how to use profiling tools to dig into
  exactly what is making code slow.
  
* Section \@ref(microbenchmarking) shows how to use microbenchmarking to 
  explore alternative implementations and figure out exactly which one is 
  fastest.

### Prerequisites {-}

We'll use [profvis](https://rstudio.github.io/profvis/) for profiling, and [bench](https://bench.r-lib.org/) for microbenchmarking.


```r
library(profvis)
library(bench)
```

## Profiling {#profiling}
\index{profiling}
\indexc{RProf()}

Across programming languages, the primary tool used to understand code performance is the profiler. There are a number of different types of profilers, but R uses a fairly simple type called a sampling or statistical profiler. A sampling profiler stops the execution of code every few milliseconds and records the call stack (i.e. which function is currently executing, and the function that called the function, and so on). For example, consider `f()`, below: 


```r
f <- function() {
  pause(0.1)
  g()
  h()
}
g <- function() {
  pause(0.1)
  h()
}
h <- function() {
  pause(0.1)
}
```

(I use `profvis::pause()` instead of `Sys.sleep()` because `Sys.sleep()` does not appear in profiling outputs because as far as R can tell, it doesn't use up any computing time.) \indexc{pause()}

If we profiled the execution of `f()`, stopping the execution of code every 0.1 s, we'd see a profile like this:


```r
"pause" "f" 
"pause" "g" "f"
"pause" "h" "g" "f"
"pause" "h" "f"
```

Each line represents one "tick" of the profiler (0.1 s in this case), and function calls are recorded from right to left: the first line shows `f()` calling `pause()`. It shows that the code spends 0.1 s running `f()`, then 0.2 s running `g()`, then 0.1 s running `h()`.

If we actually profile `f()`, using `utils::Rprof()` as in the code below, we're unlikely to get such a clear result.


```r
tmp <- tempfile()
Rprof(tmp, interval = 0.1)
f()
Rprof(NULL)
writeLines(readLines(tmp))
#> sample.interval=100000
#> "pause" "g" "f" 
#> "pause" "h" "g" "f" 
#> "pause" "h" "f" 
```

That's because all profilers must make a fundamental trade-off between accuracy and performance. The compromise that makes, using a sampling profiler, only has minimal impact on performance, but is fundamentally stochastic because there's some variability in both the accuracy of the timer and in the time taken by each operation. That means each time that you profile you'll get a slightly different answer. Fortunately, the variability most affects functions that take very little time to run, which are also the functions of least interest.

### Visualising profiles
\indexc{profvis()}

The default profiling resolution is quite small, so if your function takes even a few seconds it will generate hundreds of samples. That quickly grows beyond our ability to look at directly, so instead of using `utils::Rprof()` we'll use the profvis package to visualise aggregates. profvis also connects profiling data back to the underlying source code, making it easier to build up a mental model of what you need to change. If you find profvis doesn't help for your code, you might try one of the other options like `utils::summaryRprof()` or the proftools package [@proftools].

There are two ways to use profvis:

*   From the Profile menu in RStudio.
  
*   With `profvis::profvis()`. I recommend storing your code in a separate 
    file and `source()`ing it in; this will ensure you get the best connection 
    between profiling data and source code.

    
    ```r
    source("profiling-example.R")
    profvis(f())
    ```

After profiling is complete, profvis will open an interactive HTML document that allows you to explore the results. There are two panes, as shown in Figure \@ref(fig:flamegraph). 

\begin{figure}

{\centering \includegraphics[width=1\linewidth]{screenshots/performance/flamegraph} 

}

\caption{profvis output showing source on top and flame graph below.}(\#fig:flamegraph)
\end{figure}

The top pane shows the source code, overlaid with bar graphs for memory and execution time for each line of code. Here I'll focus on time, and we'll come back to memory shortly. This display gives you a good overall feel for the bottlenecks but doesn't always help you precisely identify the cause. Here, for example, you can see that `h()` takes 150 ms, twice as long as `g()`; that's not because the function is slower, but because it's called twice as often.

The bottom pane displays a __flame graph__ showing the full call stack. This allows you to see the full sequence of calls leading to each function, allowing you to see that `h()` is called from two different places. In this display you can mouse over individual calls to get more information, and see the corresponding line of source code, as in Figure \@ref(fig:perf-info).

\begin{figure}

{\centering \includegraphics[width=1\linewidth]{screenshots/performance/info} 

}

\caption{Hovering over a call in the flamegraph highlights the corresponding line of code, and displays additional information about performance.}(\#fig:perf-info)
\end{figure}

Alternatively, you can use the __data tab__, Figure \@ref(fig:perf-tree) lets you interactively dive into the tree of performance data. This is basically the same display as the flame graph (rotated 90 degrees), but it's more useful when you have very large or deeply nested call stacks because you can choose to interactively zoom into only selected components.

\begin{figure}

{\centering \includegraphics[width=1\linewidth]{screenshots/performance/tree} 

}

\caption{The data gives an interactive tree that allows you to selectively zoom into key components}(\#fig:perf-tree)
\end{figure}

### Memory profiling
\index{profiling!memory}
\index{garbage collector!performance}
\index{memory usage}

There is a special entry in the flame graph that doesn't correspond to your code: `<GC>`, which indicates that the garbage collector is running. If `<GC>` is taking a lot of time, it's usually an indication that you're creating many short-lived objects. For example, take this small snippet of code:


```r
x <- integer()
for (i in 1:1e4) {
  x <- c(x, i)
}
```

If you profile it, you'll see that most of the time is spent in the garbage collector, Figure \@ref(fig:perf-memory).

\begin{figure}

{\centering \includegraphics[width=1\linewidth]{screenshots/performance/memory} 

}

\caption{Profiling a loop that modifies an existing variable reveals that most time is spent in the garbage collector (<GC>).}(\#fig:perf-memory)
\end{figure}

When you see the garbage collector taking up a lot of time in your own code, you can often figure out the source of the problem by looking at the memory column: you'll see a line where large amounts of memory are being allocated (the bar on the right) and freed (the bar on the left). Here the problem arises because of copy-on-modify (Section \@ref(copy-on-modify)): each iteration of the loop creates another copy of `x`. You'll learn strategies to resolve this type of problem in Section \@ref(avoid-copies).

### Limitations
\index{profiling!limitations}

There are some other limitations to profiling:

*   Profiling does not extend to C code. You can see if your R code calls C/C++
    code but not what functions are called inside of your C/C++ code. 
    Unfortunately, tools for profiling compiled code are beyond the scope of
    this book; start by looking at <https://github.com/r-prof/jointprof>.

*   If you're doing a lot of functional programming with anonymous functions,
    it can be hard to figure out exactly which function is being called.
    The easiest way to work around this is to name your functions.

*   Lazy evaluation means that arguments are often evaluated inside another 
    function, and this complicates the call stack (Section 
    \@ref(lazy-call-stack)). Unfortunately R's profiler doesn't store enough
    information to disentangle lazy evaluation so that in the following code, 
    profiling would  make it seem like `i()` was called by `j()` because the 
    argument isn't evaluated until it's needed by `j()`. 

    
    ```r
    i <- function() {
      pause(0.1)
      10
    }
    j <- function(x) {
      x + 10
    }
    j(i())
    ```
    
    If this is confusing, use `force()` (Section \@ref(forcing-evaluation)) to 
    force computation to happen earlier.

### Exercises

<!-- The explanation of `torture = TRUE` was removed in https://github.com/hadley/adv-r/commit/ea63f1e48fb523c013fb3df1860b7e0c227e1512 -->

1.  Profile the following function with `torture = TRUE`. What is 
    surprising? Read the source code of `rm()` to figure out what's going on.

    
    ```r
    f <- function(n = 1e5) {
      x <- rep(1, n)
      rm(x)
    }
    ```

## Microbenchmarking {#microbenchmarking}
\index{microbenchmarking|see {benchmarking}}
\index{benchmarking}
 
A __microbenchmark__ is a measurement of the performance of a very small piece of code, something that might take milliseconds (ms), microseconds (µs), or nanoseconds (ns) to run. Microbenchmarks are useful for comparing small snippets of code for specific tasks. Be very wary of generalising the results of microbenchmarks to real code: the observed differences in microbenchmarks will typically be dominated by higher-order effects in real code; a deep understanding of subatomic physics is not very helpful when baking.

A great tool for microbenchmarking in R is the bench package [@bench]. The bench package uses a high precision timer, making it possible to compare operations that only take a tiny amount of time. For example, the following code compares the speed of two approaches to computing a square root.


```r
x <- runif(100)
(lb <- bench::mark(
  sqrt(x),
  x ^ 0.5
))
#> # A tibble: 2 × 6
#>   expression      min   median `itr/sec` mem_alloc `gc/sec`
#>   <bch:expr> <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
#> 1 sqrt(x)     299.9ns    500ns  1724459.      848B        0
#> 2 x^0.5         6.8µs      7µs   140974.      848B        0
```

By default, `bench::mark()` runs each expression at least once (`min_iterations = 1`), and at most enough times to take 0.5 s (`min_time = 0.5`). It checks that each run returns the same value which is typically what you want microbenchmarking; if you want to compare the speed of expressions that return different values, set `check = FALSE`.

### `bench::mark()` results
\indexc{mark()}

`bench::mark()` returns the results as a tibble, with one row for each input expression, and the following columns:

*   `min`, `mean`, `median`, `max`, and `itr/sec` summarise the time taken by the 
    expression. Focus on the minimum (the best possible running time) and the
    median (the typical time). In this example, you can see that using the 
    special purpose `sqrt()` function is faster than the general exponentiation 
    operator. 

    You can visualise the distribution of the individual timings with `plot()`:

    
    ```r
    plot(lb)
    #> Loading required namespace: tidyr
    ```
    
    
    
    \begin{center}\includegraphics[width=0.7\linewidth]{Perf-measure_files/figure-latex/unnamed-chunk-9-1} \end{center}

    The distribution tends to be heavily right-skewed (note that the x-axis is 
    already on a log scale!), which is why you should avoid comparing means. 
    You'll also often see multimodality because your computer is running
    something else in the background.

*   `mem_alloc` tells you the amount of memory allocated by the first run,
    and `n_gc()` tells you the total number of garbage collections over all
    runs. These are useful for assessing the memory usage of the expression.
  
*   `n_itr` and `total_time` tells you how many times the expression was 
    evaluated and how long that took in total. `n_itr` will always be
    greater than the `min_iteration` parameter, and `total_time` will always
    be greater than the `min_time` parameter.

*   `result`, `memory`, `time`, and `gc` are list-columns that store the 
    raw underlying data.

Because the result is a special type of tibble, you can use `[` to select just the most important columns. I'll do that frequently in the next chapter.


```r
lb[c("expression", "min", "median", "itr/sec", "n_gc")]
#> # A tibble: 2 × 4
#>   expression      min   median `itr/sec`
#>   <bch:expr> <bch:tm> <bch:tm>     <dbl>
#> 1 sqrt(x)     299.9ns    500ns  1724459.
#> 2 x^0.5         6.8µs      7µs   140974.
```

### Interpreting results



As with all microbenchmarks, pay careful attention to the units: here, each computation takes about 300 ns, 300 billionths of a second. To help calibrate the impact of a microbenchmark on run time, it's useful to think about how many times a function needs to run before it takes a second. If a microbenchmark takes:

* 1 ms, then one thousand calls take a second.
* 1 µs, then one million calls take a second.
* 1 ns, then one billion calls take a second.

The `sqrt()` function takes about 300 ns, or 0.3 µs, to compute the square roots of 100 numbers. That means if you repeated the operation a million times, it would take 0.3 s, and hence changing the way you compute the square root is unlikely to significantly affect real code. This is the reason you need to exercise care when generalising microbenchmarking results.

### Exercises

1. Instead of using `bench::mark()`, you could use the built-in function
   `system.time()`. But `system.time()` is much less precise, so you'll
   need to repeat each operation many times with a loop, and then divide
   to find the average time of each operation, as in the code below.

    
    ```r
    n <- 1e6
    system.time(for (i in 1:n) sqrt(x)) / n
    system.time(for (i in 1:n) x ^ 0.5) / n
    ```
    
    How do the estimates from `system.time()` compare to those from
    `bench::mark()`? Why are they different?

1.  Here are two other ways to compute the square root of a vector. Which
    do you think will be fastest? Which will be slowest? Use microbenchmarking
    to test your answers.

    
    ```r
    x ^ (1 / 2)
    exp(log(x) / 2)
    ```

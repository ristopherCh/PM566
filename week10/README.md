PM 566 Week 10 Lab
================
Chris Hanson
10/29/2021

# Question 1:

Re-write fun1 in a vectorized way (without the for loop) so that it
could later utilize parallel processing.

fun1 generates a dataset of n rows and k columns using the rpois
function - the random generator for the Poisson distrubtion (lambda is
mean of poisson distribution). rbind combines vectors, matrices, or
dataframe arguments by row or column.

``` r
fun1 <- function(n = 100, k = 4, lambda = 4) {
  x <- NULL
  for (i in 1:n)
    x <- rbind(x, rpois(k, lambda))
  
  # return(x)
  x
}
```

The above code is not vectorized, it’s in a for loop. This does not
allow us to use parallel computing.

Here we re-write the code to make it vectorized:

``` r
fun1alt <- function(n = 100, k = 4, lambda = 4) {
  x <- matrix(rpois(n * k, lambda), nrow = n, ncol = k)
  x
}
```

In the above code, we generate the entire n \* k poisson values at once,
then use “matrix” to organize this vector into a matrix. It is
presumably faster, as for loops are notoriously slow in R.

Let’s compare the speed of these two function using the microbenchmark
library:

``` r
# Benchmarking
microbenchmark::microbenchmark(
  fun1(n = 1000),
  fun1alt(n = 1000), unit="relative"
)
```

    ## Unit: relative
    ##               expr      min       lq     mean   median       uq     max neval
    ##     fun1(n = 1000) 35.66984 35.11493 34.41188 34.38972 34.29885 8.30394   100
    ##  fun1alt(n = 1000)  1.00000  1.00000  1.00000  1.00000  1.00000 1.00000   100

As we set the microbenchmark to “relative,” it appears that it sets the
quicker function to 1, and reports the relative speed of the slower
function. The for loop function is 33x slower.

# Question 2:

Find the maximum value in a column. Again, fun2 is non-optimized for
parallel processing.

rnorm is a random number generator for the normal distribution with mean
automatically set equal to 0 and std to 1. Here we generate 10k values
in a normal distribution and arrange them in a matrix with 10 rows and
thus 1000 columns. We then write the fun2 function which uses apply -
returns a list obtained by applying a function to an a matrix. The
function we apply is “max” which finds the maximum of an input value,
and the input is columns because MARGIN is = 2. Apparently apply uses a
for loop and is thus non vectorized.

``` r
set.seed(1234)
x <- matrix(rnorm(1e4), nrow=10)

# Find each column's max value
fun2 <- function(x) {
  apply(x, 2, max)
}
```

Here’s a faster alternative to the above code:

Confusingly, max.col finds the index of the maximum position for each
row of a matrix. To use it to find the index of the max of a column, we
have to transpose our matrix using t(x). We store these indices in idx.
We then use cbind to name the row number of x that is the maximum of
each column, for each of the 1000 columns, 1 at a time. Very creative.

``` r
fun2alt <- function(x) {
  # Position of the max value per row of x
  idx <- max.col(t(x))
  
  x[cbind(idx, 1:ncol(x))]
}
```

The code below tests to see whether the output of the two functions are
identical.

``` r
all(fun2(x) == fun2alt(x))
```

    ## [1] TRUE

``` r
microbenchmark::microbenchmark(
  fun2(x),
  fun2alt(x), unit = "relative"
)
```

    ## Unit: relative
    ##        expr      min       lq     mean   median       uq      max neval
    ##     fun2(x) 11.32694 11.83163 10.05782 11.76056 11.62901 1.243087   100
    ##  fun2alt(x)  1.00000  1.00000  1.00000  1.00000  1.00000 1.000000   100

We see that the alternative code is 11.44 times faster than the
original.

# Question 3:

Bootstrapping is random sampling with replacement, it is used to
generate a measure of accuracy - variance or confidence interval - to a
sample estimate. The code below implements the non-parametric bootstrap.
I edit it to make it work with parallel.

The bootstrap code is a bit too complicated for me to figure out right
now.

We’re following the process from page 12 of the lecture notes: Step 1:
Create a cluster, denoting names = 1L, the \# of copies of R run on
localhost. Step 2:

``` r
library(parallel)
my_boot <- function(dat, stat, R, ncpus = 1L) {
  
  # Getting the random indices
  n <- nrow(dat)
  idx <- matrix(sample.int(n, n*R, TRUE), nrow=n, ncol=R)
 
  # Making the cluster using `ncpus`
  # STEP 1: Create a PSOCK cluster
  cl <- makePSOCKcluster(ncpus)
  
  # STEP 2: Copy/prepare each R session
    # Equivalent to 'set.seed'
  clusterSetRNGStream(cl, 123)
    # Copy objects to forked environments with clusterExport
  clusterExport(cl, c("stat", "dat", "idx"), envir = environment())
  
  # STEP 3: Call with parLapply instead of lapply
    # you must designate the cluster, otherwise everything else is identical
  ans <- parLapply(cl = cl, seq_len(R), function(i) {
    stat(dat[idx[,i], , drop=FALSE])
  })
  
  # Previously step 3 looked like:
#  ans <- lapply(seq_len(R), function(i) {
#    stat(dat[idx[,i], , drop=FALSE])
#  })
  
  # Coercing the list into a matrix
  ans <- do.call(rbind, ans)
  
  # STEP 4: Stop the cluster
  stopCluster(cl)
  
  ans
  
}
```

Here is an example:

``` r
# Bootstrap of an OLS
my_stat <- function(d) coef(lm(y ~ x, data=d))

# DATA SIM
set.seed(1)
n <- 500; R <- 5e3

x <- cbind(rnorm(n)); y <- x*5 + rnorm(n)

# Checking if we get something similar as lm
ans0 <- confint(lm(y~x))
ans1 <- my_boot(dat = data.frame(x, y), my_stat, R = R, ncpus = 2L)

# You should get something like this
t(apply(ans1, 2, quantile, c(.025,.975)))
```

    ##                   2.5%      97.5%
    ## (Intercept) -0.1395732 0.05291612
    ## x            4.8686527 5.04503468

``` r
##                   2.5%      97.5%
## (Intercept) -0.1372435 0.05074397
## x            4.8680977 5.04539763
ans0
```

    ##                  2.5 %     97.5 %
    ## (Intercept) -0.1379033 0.04797344
    ## x            4.8650100 5.04883353

``` r
##                  2.5 %     97.5 %
## (Intercept) -0.1379033 0.04797344
## x            4.8650100 5.04883353
```

(No idea what this above code means or why it’s there)

Is it faster?

``` r
system.time(my_boot(dat = data.frame(x, y), my_stat, R = 4000, ncpus = 1L))
```

    ##    user  system elapsed 
    ##    0.08    0.00    3.13

``` r
system.time(my_boot(dat = data.frame(x, y), my_stat, R = 4000, ncpus = 2L))
```

    ##    user  system elapsed 
    ##    0.09    0.02    1.44

It is almost 2x faster to use ncpus = 2L than ncpus = 1L.

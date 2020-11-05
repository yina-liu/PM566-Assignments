Assignment 04 - HPC and SQL
================
Yina Liu
11/3/2020

The learning objecives are to conduct data scraping and perfrom text
mining.

``` r
library(parallel)
library(RSQLite)
library(DBI)
```

# HPC

## Problem 1: Make sure your code is nice

Rewrite the following R function to make them faster. It is OK (and
recommended) to take a look at Stackoverflow and Google.

``` r
# Total row sums
fun1 <- function(mat) {
  n <- nrow(mat)
  ans <- double(n) 
  for (i in 1:n) {
    ans[i] <- sum(mat[i, ])
  }
  ans
}
fun1alt <- function(mat) {
  # YOUR CODE HERE
  rowSums(mat)
}

# Cumulative sum by row
fun2 <- function(mat) {
  n <- nrow(mat)
  k <- ncol(mat)
  ans <- mat
  for (i in 1:n) {
    for (j in 2:k) {
      ans[i,j] <- mat[i, j] + ans[i, j - 1]
    }
  }
  ans
}

fun2alt <- function(mat) {
   # YOUR CODE HERE
  n <- nrow(mat)
  ans <- mat
  for (i in 1:n) {
    ans[i,] = cumsum(mat[i,])
  }
  ans
}

# Use the data with this code
set.seed(2315)
dat <- matrix(rnorm(200 * 100), nrow = 200)

# Test for the first
microbenchmark::microbenchmark(
  fun1(dat),
  fun1alt(dat), unit = "relative", check = "equivalent"
)
```

    ## Unit: relative
    ##          expr     min       lq     mean   median       uq      max neval
    ##     fun1(dat) 7.56649 10.66806 6.286334 12.83786 15.79011 1.216701   100
    ##  fun1alt(dat) 1.00000  1.00000 1.000000  1.00000  1.00000 1.000000   100

``` r
# Test for the second
microbenchmark::microbenchmark(
  fun2(dat),
  fun2alt(dat), unit = "relative", check = "equivalent"
)
```

    ## Unit: relative
    ##          expr      min       lq     mean   median       uq       max neval
    ##     fun2(dat) 6.621588 3.996583 3.184712 4.332653 3.742757 0.3544529   100
    ##  fun2alt(dat) 1.000000 1.000000 1.000000 1.000000 1.000000 1.0000000   100

The last arguement, check = “equivalent”, is included to make suree that
the functions return the same result.

## Problem 2: Make things run faster with parallel computing

The following function allows simulating PI

``` r
sim_pi <- function(n = 1000, i = NULL) {
  p <- matrix(runif(n*2), ncol = 2)
  mean(rowSums(p^2) < 1) * 4
}
# Here is an example of the run
set.seed(156)
sim_pi(1000) # 3.132
```

    ## [1] 3.132

In order to get accurate estimates, we can run this function multiple
times, with the following code:

``` r
# This runs the simulation a 4,000 times, each with 10,000 points
set.seed(1231)
system.time({
  ans <- unlist(lapply(1:4000, sim_pi, n = 10000))
  print(mean(ans))
})
```

    ## [1] 3.14124

    ##    user  system elapsed 
    ##   5.564   1.391  10.116

Rewrite the previous code using `parLapply()` to make it run faster.
Make sure you set the seed using `clusterSetRNGStream()`:

``` r
# YOUR CODE HERE
system.time({
  cl <- makePSOCKcluster(2L, setup_strategy = "sequential") 
  clusterSetRNGStream(cl, 1231) 
  ans <- unlist(parLapply(cl,1:4000, sim_pi, n = 10000))
  print(mean(ans))
  stopCluster(cl)
})
```

    ## [1] 3.141577

    ##    user  system elapsed 
    ##   0.015   0.010   6.123

# SQL

Setup a temporary datavase by running the following chunk

``` r
# Initialize a temporary in memory database
con <- dbConnect(SQLite(), ":memory:")
# Download tables
film <- read.csv(
  "https://raw.githubusercontent.com/ivanceras/sakila/master/csv-sakila-db/film.csv")
film_category <- read.csv(
  "https://raw.githubusercontent.com/ivanceras/sakila/master/csv-sakila-db/film_category.csv")
category <- read.csv(
  "https://raw.githubusercontent.com/ivanceras/sakila/master/csv-sakila-db/category.csv")
# Copy data.frames to database
dbWriteTable(con, "film", film)
dbWriteTable(con, "film_category", film_category)
dbWriteTable(con, "category", category)
```

When you write a new chunk, remember to replace the `r` with `sql,
connection = con`. Some of these questions will require you to use an
inner join. Read more about them here
<https://www.w3schools.com/sql/sql_join_inner.asp>

## Question 1: How many movies is there available in each rating category?

``` sql
SELECT rating, COUNT(*) AS count
FROM film
GROUP BY rating
```

<div class="knitsql-table">

| rating | count |
| :----- | ----: |
| G      |   180 |
| NC-17  |   210 |
| PG     |   194 |
| PG-13  |   223 |
| R      |   195 |

5 records

</div>

## Question 2: What is the average replacement cost and rental rate for each rating category.

``` sql
SELECT rating, avg(replacement_cost), avg(rental_rate)
FROM film
GROUP BY rating
```

<div class="knitsql-table">

| rating | avg(replacement\_cost) | avg(rental\_rate) |
| :----- | ---------------------: | ----------------: |
| G      |               20.12333 |          2.912222 |
| NC-17  |               20.13762 |          2.970952 |
| PG     |               18.95907 |          3.051856 |
| PG-13  |               20.40256 |          3.034843 |
| R      |               20.23103 |          2.938718 |

5 records

</div>

## Question 3: Use table film\_category together with film to find the how many films there are with each category ID

``` sql
SELECT a.film_id, count(b.film_id) as COUNT,a.category_id, b.film_id
FROM film_category AS a
  INNER JOIN film AS b on a.film_id = b.film_id
GROUP BY category_id
ORDER BY COUNT DESC 
```

<div class="knitsql-table">

| film\_id | COUNT | category\_id | film\_id |
| -------: | ----: | -----------: | -------: |
|       10 |    74 |           15 |       10 |
|        6 |    73 |            9 |        6 |
|        5 |    69 |            8 |        5 |
|        1 |    68 |            6 |        1 |
|       18 |    66 |            2 |       18 |
|       19 |    64 |            1 |       19 |
|       22 |    63 |           13 |       22 |
|       33 |    62 |            7 |       33 |
|       26 |    61 |           14 |       26 |
|       46 |    61 |           10 |       46 |

Displaying records 1 - 10

</div>

## Question 4: Incorporate table category into the answer to the previous question to find the name of the most popular category.

``` sql
SELECT name, a.category_id , count(a.film_id) as COUNT
FROM film_category a 
  JOIN film b on a.film_id=b.film_id
  JOIN category c on a.category_id=c.category_id 
GROUP BY a.category_id
ORDER BY COUNT DESC
```

<div class="knitsql-table">

| name        | category\_id | COUNT |
| :---------- | -----------: | ----: |
| Sports      |           15 |    74 |
| Foreign     |            9 |    73 |
| Family      |            8 |    69 |
| Documentary |            6 |    68 |
| Animation   |            2 |    66 |
| Action      |            1 |    64 |
| New         |           13 |    63 |
| Drama       |            7 |    62 |
| Sci-Fi      |           14 |    61 |
| Games       |           10 |    61 |

Displaying records 1 - 10

</div>

The most popular category is Sprorts (74 films) and Foreign (73 films).

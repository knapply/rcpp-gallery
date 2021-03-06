---
title: Detecting a Time Series Change Point
author: Joshua French
license: GPL (>= 2)
tags: modeling armadillo
summary: Detects how to detect a change point in a time series using Bayesian methodology.  
  Compares naive and vectorized R solutions to one implemented using Rcpp and 
  RcppArmadillo's sample function.
layout: post
src: 2014-01-04-bayesian-time-series-changepoint.Rmd
---

In this example we will detect the change point in a time series of counts
using Bayesian methodology.  A natural solution to this problem utilizes a Gibbs sampler.  We'll first implement the sampler in R naively, then create a vectorized R implementation, and lastly create an implementation of the sampler using Rcpp and RcppArmadillo.  We will compare these implementations using a famous coal mining data set.  We will see that the speed of the sampler greatly improves with each implementation.  

We will assume that we have a time series of `n` responses coming from a Poisson distribution.  The first `k` observations are i.i.d. and come from a Poisson distribution with mean `lambda`, while the remaining observations are i.i.d. and come from a Poisson distribution with mean `phi`.  We will assume the prior distribution for `lambda` is Gamma(`a`, `b`), the prior distribution for `phi` is Gamma(`c`, `d`), and the prior distribution for `k` is discrete uniform over the values 1,2,...,`n`.  From this information, one can arrive at closed form expressions for the full conditional distribution of `lambda`, `phi`, and `k` (see [here](https://github.com/jpfrench81/files/raw/master/changepoint.pdf)).  

We can sample values from the full conditional distributions of `lambda` and `phi` using the
`rgamma` function built into R.  We can sample from the discrete set
of values 1, 2, ..., `n` using the  `sample` function with the
appropriate sampling probability for each value.  Since the probability mass
function (pmf) for the full conditional distribution of `k` is a bit
complicated, we'll code it up as its own function taking the observed data
`y` and the parameters `lambda` and `phi`.  The functions for the pmf will be named `kprob*`.  Note
that in the pmf we will need to calculate the natural logarithm of a sum of
exponential values; this can sometimes create problems with numerical
underflow or overflow.  Consequently, we will also create a  function
`logsumexp` to do this efficiently.  The Gibbs samplers for the three implementations will be named `gibbsloop`, `gibbsvec`, and `gibbscpp`.  

The initial naive R implementation of the sampler is given below. 


{% highlight r %}
# Function for Gibbs sampler.  Takes the number of simulations desired,
# the vector of observed data, the values of the hyperparameters, the 
# possible values of k, a starting value for phi, and a starting
# value for k.

# Function takes:
# nsim: number of cycles to run
# y: vector of observed values
# a, b: parameters for prior distribution of lambda
# c, d: parameters for prior distribution of phi
# kposs: possible values of k
# phi, k: starting values of chain for phi and k

gibbsloop <- function(nsim, y, a, b, c, d, kposs, phi, k) {
    # matrix to store simulated values from each cycle
    out <- matrix(NA, nrow = nsim, ncol = 3)

    # number of observations
    n <- length(y)

    for(i in 1:nsim) {
	# Generate value from full conditional of phi based on
	# current values of other parameters
	lambda <- rgamma(1, a + sum(y[1:k]), b + k)
	
	# Generate value from full conditional of phi based on
	# current values of other parameters
	phi <- rgamma(1, c + sum(y[min((k+1),n):n]), d + n - k)

	# generate value of k
	# determine probability masses for full conditional of k
	# based on current parameters values
	pmf <- kprobloop(kposs, y, lambda, phi)
	k <- sample(x = kposs, size = 1, prob = pmf)
	
	out[i, ] <- c(lambda, phi, k)
    }
    out
}

# Given a vector of values x, the logsumexp function calculates
# log(sum(exp(x))), but in a "smart" way that helps avoid
# numerical underflow or overflow problems.

logsumexp <- function(x) {
    log(sum(exp(x - max(x)))) + max(x)
}

# Determine pmf for full conditional of k based on current values of other
# variables. Takes possible values of k, observed data y, current values of 
# lambda, and phi. It does this naively using a loop.

kprobloop <- function(kposs, y, lambda, phi) {	
    # create vector to store argument of exponential function of 
    # unnormalized pmf, then calculate them
    x <- numeric(length(kposs))
    for (i in kposs) {
	x[i] <- i*(phi - lambda) + sum(y[1:i]) * log(lambda/phi)
    }
    # return full conditional pmf of k
    return(exp(x - logsumexp(x)))
}
{% endhighlight %}

Looking through the first implemention, we realize that we can 
speed up our code by vectorizing when possible and not performing computations
more often than we have to.  Specifically, we are calculating `sum(y[1:i])` and `sum(y[min((k+1),n):n])` repeatedly in the loops.  We can dramatically improve the speed of our implementation by calculating the sum of `y` and the cumulative sum of `y` and then using the stored results appropriately.  We now reimplement the pmf and Gibbs sampler functions with this in
mind.  


{% highlight r %}
gibbsvec <- function(nsim, y, a, b, c, d, kposs, phi, k) {
    # matrix to store simulated values from each cycle
    out <- matrix(NA, nrow = nsim, ncol = 3)

    # determine number of observations
    n <- length(y)

    # determine sum of y and cumulative sum of y.  
    # Then cusum[k] == sum(y[1:k])
    # and sum(y[(k+1):n]) == sumy - cusum[k]
    sumy <- sum(y)
    cusum <- cumsum(y)

    for (i in 1:nsim) {
	# Generate value from full conditional of phi based on
	# current values of other parameters
	lambda <- rgamma(1, a + cusum[k], b + k)
		
	# Generate value from full conditional of phi based on
	# current values of other parameters
	phi <- rgamma(1, c + sumy - cusum[k], d + n - k)
		
	# generate value of k
	pmf <- kprobvec(kposs, cusum, lambda, phi)
	k <- sample(x = kposs, size = 1, prob = pmf)
		
	out[i, ] <- c(lambda, phi, k)
    }
    out
}

# Determine pmf for full conditional of k based on current values of other
# variables. Do this efficiently using vectors and stored information.
# cusum is the cumulative sum of y.

kprobvec <- function(kposs, cusum, lambda, phi) {
    # calculate exponential argument of numerator of unnormalized pmf
    x <- kposs * (phi - lambda) + cusum * log(lambda/phi)
    # return pmf of full conditional for k
    return(exp(x - logsumexp(x)))
}
{% endhighlight %}

We should be able to improve the speed of our sampler by implementing it in C++.  We can reimplement the vectorized version of the sampler fairly easily using Rcpp and RcppArmadillo.  Note that the parameterization of `rgamma` used by Rcpp is slightly different from base R.


{% highlight cpp %}
// [[Rcpp::depends(RcppArmadillo)]]
#include <RcppArmadilloExtensions/sample.h>
using namespace Rcpp;

// [[Rcpp::export]]
double logsumexp(NumericVector x) {
    return(log(sum(exp(x - max(x)))) + max(x));
}

// [[Rcpp::export]]
NumericVector kprobcpp(NumericVector kposs, NumericVector cusum, double lambda, double phi) {
    NumericVector x = kposs * (phi - lambda) + cusum * log(lambda/phi);
    return(exp(x - logsumexp(x)));
}

// [[Rcpp::export]]
NumericMatrix gibbscpp(int nsim, NumericVector y, double a, double b, double c,
                       double d, NumericVector kposs, NumericVector phi, NumericVector k) {

    NumericVector lambda;
    NumericVector pmf;
    NumericMatrix out(nsim, 3);

    // determine sum of y and cumulative sum of y.  
    double sumy = sum(y);
    NumericVector cusum = cumsum(y);

    for(int i = 0; i < nsim; i++) {
        lambda = rgamma(1, a + cusum[k[0] - 1], 1/(b + k[0]));
        phi = rgamma(1, c + sumy - cusum[k[0] - 1], 1/(d + 112 - k[0]));

        pmf = kprobcpp(kposs, cusum, lambda[0], phi[0]);
        k = RcppArmadillo::sample(kposs, 1, false, pmf) ;

        // store values of this cycle
        out(i, 0) = lambda[0];
        out(i, 1) = phi[0];
        out(i, 2) = k[0];
    }

    // return results
    return out;
}
{% endhighlight %}

We will now compare and apply the implementations to a coal mining data set.

Coal mining is a notoriously dangerous occupation.  Consider the tally of
coal mine disasters over a 112-year period (1851 to 1962) in the United
Kingdom.  The data have relatively high disaster counts in the early era, and
relatively low counts in the later era. When did technology improvements and
safety practices have an actual effect on the rate of serious coal mining
disasters in the United Kingdom?  

Let us first read the data into R:


{% highlight r %}
yr <- 1851:1962
counts <- c(4,5,4,1,0,4,3,4,0,6,3,3,4,0,2,6,3,3,5,4,5,3,1,4,4,1,5,5,3,4,2,
            5,2,2,3,4,2,1,3,2,2,1,1,1,1,3,0,0,1,0,1,1,0,0,3,1,0,3,2,2,0,1,
            1,1,0,1,0,1,0,0,0,2,1,0,0,0,1,1,0,2,3,3,1,1,2,1,1,1,1,2,4,2,0,
            0,0,1,4,0,0,0,1,0,0,0,0,0,1,0,0,1,0,1)
{% endhighlight %}

For simplicity, we'll refer to 1851 as year 1, 1852 as year 2, ..., 1962 as
year 112.  We will set the initial values for `phi` and `k` to be 1 and 40,
respectively.  We will set the hyperparameters `a`, `b`, `c`, and `d` to be 4, 1, 1, and 2, respectively.  

We will run the samplers for 10,000 cycles each.  We set the random number seed for each sampler to be 1 for
reproducible results.  We then check that the results for the three samplers are identical.


{% highlight r %}
nsim = 10000
set.seed(1)
chain1 <- gibbsloop(nsim = nsim, y = counts, a = 4, b = 1, c = 1, d = 2, kposs = 1:112, phi = 1, k = 40)
set.seed(1)
chain2 <- gibbsvec(nsim = nsim, y = counts, a = 4, b = 1, c = 1, d = 2, kposs = 1:112, phi = 1, k = 40)
set.seed(1)
chain3 <- gibbscpp(nsim = nsim, y = counts, a = 4, b = 1, c = 1, d = 2, kposs = 1:112, phi = 1, k = 40)
all.equal(chain1, chain2, chain3)
{% endhighlight %}



<pre class="output">
[1] TRUE
</pre>

We now compare compare the relative speed of the three implementations using the `benchmark` function.


{% highlight r %}
library(rbenchmark)
benchmark(gibbsloop(nsim=nsim,y=counts,a=4,b=1,c=1,d=2,kposs=1:112,phi=1,k=40),
          gibbsvec(nsim=nsim,y=counts,a=4,b=1,c=1,d=2,kposs=1:112,phi=1,k=40),
          gibbscpp(nsim=nsim,y=counts,a=4,b=1,c=1,d=2,kposs=1:112,phi=1,k=40),
	  order="relative",replications=10)[,1:4]
{% endhighlight %}



<pre class="output">
                                                                                            test
3  gibbscpp(nsim = nsim, y = counts, a = 4, b = 1, c = 1, d = 2, kposs = 1:112, phi = 1, k = 40)
2  gibbsvec(nsim = nsim, y = counts, a = 4, b = 1, c = 1, d = 2, kposs = 1:112, phi = 1, k = 40)
1 gibbsloop(nsim = nsim, y = counts, a = 4, b = 1, c = 1, d = 2, kposs = 1:112, phi = 1, k = 40)
  replications elapsed relative
3           10   2.207    1.000
2           10   5.611    2.542
1           10  68.118   30.865
</pre>

The C++ version of the Gibbs sampler is a vast improvement over the looped R
version and also quite a bit faster than the vectorized R version.   

To complete this application, we analyze the samples from the posterior
distribution of `k`. The median value of the posterior sample is 40, meaning
the estimated change point year is 1890.  To visualize the distribution of
the data before and after estimated change point, we can create a lineplot of the
time series with years 1851-1890 colored orange and the remaining years
colored blue.


{% highlight r %}
(changeyear = median(chain1[,3]))
{% endhighlight %}



<pre class="output">
[1] 40
</pre>



{% highlight r %}
(yr[changeyear])
{% endhighlight %}



<pre class="output">
[1] 1890
</pre>



{% highlight r %}
## plot not shown as Rcpp Gallery cannot display charts
#plot(yr, counts, xlab = "year", ylab = "number of mining accidents", type = "n")
#lines(yr[1:changeyear], counts[1:changeyear], col = "orange")
#lines(yr[-(1:(changeyear-1))], counts[-(1:(changeyear-1))], col = "blue")
{% endhighlight %}
